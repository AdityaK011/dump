---
title: "Linux Fundamentals, Architecture & Debugging — a platform/security engineer's map"
date: 2026-06-29
tags:
  - linux
  - fundamentals
  - architecture
  - kernel
  - systems
  - debugging
  - observability
  - platform-engineering
  - security
draft: false
---

> [!summary] What this is
> A single map of how a Linux system is actually built — power-on to application — and the debugging route that lives at each layer. Written to be grown into, not read once. The throughline: **every bug and every attack lives at a specific layer, and the skill is locating the layer before touching anything.** Security and platform concerns are woven in rather than bolted on, because at the level of syscalls, namespaces, and privilege boundaries, "how it works" and "how it's attacked" are the same subject.

---

## 0. The one mental model

Linux is a stack of abstractions, each trusting the one below it:

```text
        application  (your service, a shell, a browser)
            │  library calls (libc)
   ─────────┼───────────────────────────────  user space (ring 3)
            │  system calls  ← the real API boundary
   ─────────┼───────────────────────────────  kernel space (ring 0)
          kernel  (scheduler, MM, VFS, net stack, drivers)
            │
        firmware  (UEFI, device firmware blobs)
            │
        hardware  (CPU, RAM, disks, NICs)
```

Two ideas do most of the work:

1. **The user/kernel boundary is the system call.** Userspace cannot touch hardware, memory it doesn't own, or other processes directly. It *asks* the kernel via syscalls (`open`, `read`, `mmap`, `clone`, `socket`…). This is the single most important boundary for both performance and security — everything privileged crosses it, so it's where you trace (`strace`) and where you sandbox (`seccomp`).
2. **"Everything is a file" (a file descriptor).** Files, sockets, pipes, devices, even kernel state (`/proc`, `/sys`) are reached through the same `read`/`write`/`ioctl` interface and referenced by integer file descriptors. Once you see that, half of `/proc` and most of debugging stops being magic.

> [!tip] The debugging corollary
> A failure cascades *upward*. Your app says "connection refused"; the daemon says "address unavailable"; the kernel says the device isn't up; the driver says firmware is dead. **The first error from the bottom is the bug; everything above it is a symptom.** Read bottom-up, fix the lowest thing.

---

## 1. Boot: from power to prompt

| Stage | What runs | Where it lives | Debug / inspect |
|---|---|---|---|
| Firmware | UEFI (or legacy BIOS) | flash on the board | firmware setup; `efibootmgr` |
| Boot manager | GRUB (usually) | EFI System Partition | edit entry at boot; `/etc/default/grub` |
| Kernel + initramfs | `vmlinuz` + early userspace | `/boot` | `cat /proc/cmdline` |
| Early userspace | `initramfs` (mounts real root, loads drivers) | `/boot/initrd.img` | `lsinitramfs`, rebuild with `update-initramfs -u` |
| PID 1 | `init` = **systemd** | `/sbin/init` → systemd | `systemd-analyze`, `journalctl -b` |
| Targets | systemd brings up units toward a target | `/etc/systemd/system`, `/lib/systemd/system` | `systemctl get-default`, `list-dependencies` |

Useful levers:
- **Kernel command line** (`/proc/cmdline`) is set per-boot in GRUB. Append `systemd.unit=rescue.target` or `single` for a minimal shell, `init=/bin/bash` to bypass init entirely, `nomodeset` to fight GPU issues. Persisted in `GRUB_CMDLINE_LINUX` then `update-grub`.
- **Boot performance:** `systemd-analyze blame` (slowest units) and `systemd-analyze critical-chain` (the dependency path that actually gated boot — different from "slowest").
- **Recovery targets:** `rescue.target` (single-user, root fs mounted) and `emergency.target` (almost nothing mounted) are your safety nets.

**Security view — the chain of trust.** Secure Boot verifies the bootloader and kernel against keys in firmware; **measured boot** additionally hashes each stage into the TPM's PCRs so you can later *attest* what booted. IMA/EVM can extend measurement to files at runtime. The point: integrity is established bottom-up, and a break anywhere below your code (firmware, bootloader, initramfs) invalidates everything above it.

---

## 2. The kernel: what it actually does

The kernel is one privileged program providing five core services:

- **Process scheduling** — deciding which thread runs on which CPU, when.
- **Memory management** — virtual address spaces, paging, the page cache.
- **Virtual File System (VFS)** — one interface (`open/read/write`) over many filesystems and pseudo-filesystems.
- **Networking stack** — sockets down to packets.
- **Device drivers** — the only code that talks to hardware.

…all exposed through the **system call interface**, plus a logging facility.

### Kernel modules
Drivers and features load on demand as modules (`.ko`) from `/lib/modules/$(uname -r)/`:

```bash
lsmod                       # loaded modules
modinfo <mod>               # author, params, firmware deps, signature
sudo modprobe <mod>         # load (resolves deps)
sudo modprobe -r <mod>      # unload
cat /sys/module/<mod>/parameters/*   # live parameter values
```
Options go in `/etc/modprobe.d/*.conf` as `options <mod> key=val`; rebuild initramfs if the module loads early.

### Kernel logging & crashes
- The **ring buffer** (`dmesg`) is the kernel's own log — fixed size, wraps under load, wiped on reboot. `printk` levels run `emerg`→`debug`.
- An **oops** is a recoverable kernel fault (a thread dies, system limps on); a **panic** is unrecoverable (halt). Both dump a register/stack trace — read the top line (the failing function) first.
- **Magic SysRq** (`Alt+SysRq+<key>`, gated by `kernel.sysrq`) is the in-kernel emergency console: `s` sync, `u` remount-ro, `b` reboot — the safe way to recover a wedged box without a hard reset.

### Tracing the kernel (the modern toolbox)
| Tool | Use |
|---|---|
| `dmesg` / `journalctl -k` | kernel messages (latter persists across reboots) |
| `ftrace` (`/sys/kernel/tracing`) | built-in function & event tracer |
| `perf` | sampling profiler + tracepoints; CPU flame graphs |
| `kprobes` / `uprobes` | dynamic probes on (almost) any kernel/user function |
| **eBPF** (`bpftrace`, `bcc`) | safe, programmable in-kernel tracing — the unifying modern tool |

**Security view — shrinking the kernel attack surface.** The kernel is the most valuable target (compromise = total control), and its attack surface is *syscalls + drivers + modules*. Reduce it: sign modules and enforce via Secure Boot; enable **kernel lockdown** (`/sys/kernel/security/lockdown`) to block `/dev/mem`, raw `/sys` device pokes, and unsigned module loads; rely on **KASLR** to randomize the kernel base; and use **seccomp** to cut the syscalls a process may make. Mandatory access control (**SELinux**/**AppArmor**, both LSMs) confines what even root-equivalent processes can do.

---

## 3. Processes & threads

A process is an address space + one or more threads + a bundle of resources (open fds, credentials, namespaces). Creation is the classic dance:

- **`fork()`** clones the calling process (copy-on-write — pages are shared until written, so fork is cheap).
- **`exec()`** replaces the program image in place.
- **`wait()`** reaps a finished child; if no one reaps it, it's a **zombie** (`Z`). If a parent dies first, children are re-parented to PID 1, which must reap them — *this is why a container's PID 1 needs to reap, or you leak zombies.*
- Linux threads are processes sharing an address space (`clone()` with shared flags); the scheduler schedules threads.

**Process states** (the `S` column in `ps`):

| State | Meaning | Debugging note |
|---|---|---|
| `R` | running / runnable | on or waiting for CPU |
| `S` | interruptible sleep | normal idle-waiting |
| `D` | **uninterruptible sleep** | stuck in kernel (usually I/O); **can't even be `kill -9`'d** — look here for "frozen, won't die" |
| `Z` | zombie | finished, not reaped — a parent bug |
| `T`/`t` | stopped / trace-stopped | `SIGSTOP` or under a debugger |

**Scheduling:** historically **CFS** (Completely Fair Scheduler); recent kernels (6.6+) default to **EEVDF**. `nice`/`renice` set politeness; `chrt` sets real-time policies; **cgroups** cap and account CPU per group (the basis of container limits).

### Debugging routes for processes
```bash
ps aux / ps -ef            # snapshot; htop for live
top -H                     # per-thread
strace -f -p <pid>         # every syscall a process makes — the workhorse
ltrace -p <pid>            # library calls instead
lsof -p <pid>              # open files/sockets
gdb -p <pid>               # attach a debugger; coredumpctl for post-mortem cores
```
The `/proc/<pid>/` directory is a process X-ray:
```text
status   creds, state, memory summary       maps     memory regions (libs, stack, heap)
fd/      every open file descriptor          smaps    per-region memory detail
cmdline  exact argv                           stack    current kernel stack (root)
environ  environment variables                wchan    what kernel function it's sleeping in
limits   rlimits in effect                    cgroup   which cgroups it's in
```
**The `D`-state hang route:** `cat /proc/<pid>/stack` and `/proc/<pid>/wchan` to see *where in the kernel* it's blocked; correlate with `dmesg` for the failing device/fs. You don't kill a `D`-state process — you fix what it's waiting on.

**Security view.** Credentials (uid/gid/**capabilities**) live on the process. `/proc/<pid>/maps` and `environ` are recon targets (secrets in env vars leak here). `kernel.yama.ptrace_scope` restricts which processes may `ptrace` others — it's *why* `strace`/`gdb` on another user's process may be denied on a hardened host.

---

## 4. Memory

Every process sees a private, contiguous **virtual address space**; the **MMU** translates virtual→physical pages via page tables, cached by the **TLB**. Physical RAM is shared and overcommitted behind the scenes.

Key concepts:
- **Paging & page faults** — pages are mapped lazily; a fault pulls them in (or kills you if invalid).
- **Page cache** — free RAM caches file data; this is why "used memory" looks high and `free` shows a big `buff/cache`. It's reclaimable, not lost.
- **Anonymous vs file-backed** — heap/stack (anon) vs `mmap`'d files. **Swap** spills anon pages to disk under pressure.
- **Overcommit & the OOM killer** — Linux hands out more virtual memory than exists (`vm.overcommit_memory`); when it actually runs out, the **OOM killer** picks a victim by `oom_score`. OOM kills show up in `dmesg`/`journalctl -k`.

### Debugging routes for memory
```bash
free -h                       # totals; mind buff/cache vs available
cat /proc/meminfo             # the source of truth
vmstat 1                      # paging/swap activity over time (si/so = swapping!)
pmap -x <pid>                 # per-process memory map
cat /proc/<pid>/smaps_rollup  # PSS/RSS rollup for a process
# leaks: valgrind --tool=massif, or heaptrack
```
Triage: high RSS + swapping (`si/so` in vmstat) → memory pressure; sudden process death + `dmesg` OOM line → overcommit caught up; growing RSS over time → leak.

**Security view.** **ASLR** (`kernel.randomize_va_space=2`) randomizes layout to thwart exploits; **W^X** / NX means no page is both writable and executable. `/proc/<pid>/maps` reveals the layout (recon), which is why hardened kernels also restrict it.

---

## 5. Filesystems & storage

The **VFS** gives one interface over everything. Core objects: **inode** (the file's metadata + block pointers; the *name* is separate), **dentry** (directory entry caching name→inode), **file descriptor** (a process's handle into the open-file table).

Layers below VFS: filesystem driver (ext4/xfs/btrfs) → **block layer** (I/O scheduler, queueing) → device. Above and beside: **page cache** (writeback of dirty pages), **pseudo-filesystems** (`proc`, `sys`, `tmpfs`, `cgroup2`) that are kernel state dressed as files, and **overlayfs**/**bind mounts** (the basis of container images and `mnt` namespaces).

### Debugging routes for storage/IO
```bash
df -h                 # space by mount
df -i                 # INODES by mount — "disk full" with free space = inode exhaustion
du -sh *              # what's using space (but see the gotcha below)
lsof | grep deleted   # deleted-but-still-open files holding space (df full, du clean!)
iostat -xz 1          # per-device utilization & latency (sysstat)
iotop                 # per-process I/O
cat /proc/mounts      # what's actually mounted, with options
sudo smartctl -a /dev/sda   # disk health (SMART)
dmesg | grep -i 'I/O error' # hardware/fs errors
```

> [!tip] The classic "disk full but `du` disagrees"
> A process holding an open fd to a deleted file keeps its blocks allocated until the fd closes. `du` (walks names) won't see it; `df` (counts blocks) will. `lsof | grep deleted` finds the culprit; restart the holder or truncate via its `/proc/<pid>/fd/<n>`.

**Security view.** Mount options are cheap, strong controls: `noexec` (no running binaries from here), `nosuid` (ignore setuid bits), `nodev` (no device nodes) — apply to `/tmp`, uploads, removable media. **File capabilities** (`getcap`/`setcap`) grant specific powers to a binary without full setuid-root. The **immutable bit** (`chattr +i`) blocks modification even by root.

---

## 6. Networking

The path of a packet — and the debugging tool at each rung:

| Layer | Component | Inspect with |
|---|---|---|
| L1/L2 | NIC + driver | `ethtool`, `dmesg` |
| Link/interface | netdev (`eth0`/`wlp*`) | `ip link` |
| Addressing/routing | IP, routes, neighbors | `ip addr`, `ip route`, `ip neigh` |
| Filtering/NAT | **nftables** (legacy iptables), conntrack | `nft list ruleset`, `conntrack -L` |
| Transport | TCP/UDP sockets, ports | `ss -tulpn` |
| Resolution | DNS | `dig`, `resolvectl status` |
| App | your client | `curl -v`, `tcpdump` |

The kernel↔userspace control channel for all of this is **netlink** (the `RTNETLINK` family handles links/routes). So an error like `RTNETLINK answers: ...` is literally the kernel replying to an `ip` request — precise, just often several layers above the root cause.

### Debugging routes for networking
```bash
ip a; ip r                 # interfaces, addresses, routes
ss -tulpn                  # listening/established sockets + owning process (replaces netstat)
ping / mtr <host>          # reachability + per-hop loss/latency
dig <name> +short          # DNS without the noise
sudo tcpdump -ni any port 443   # see the actual packets — ground truth
curl -v https://host       # full request/response incl. TLS handshake
conntrack -L               # live NAT/connection-tracking table
```
Method: work the layers — link up? (`ip link`) → address? (`ip addr`) → route to dest? (`ip route get <ip>`) → DNS resolves? (`dig`) → port open / firewall? (`ss`, `nft`) → does the packet leave/return? (`tcpdump`).

**Security view.** `nftables` + `conntrack` are your stateful firewall. **Packet capture** is core incident-response evidence. **Network namespaces** isolate stacks per container/tenant. **eBPF/XDP** enables high-performance filtering and observability right at the driver. `ss` and `/proc/net/*` reveal who's listening — first thing to check on a suspected-compromised host.

---

## 7. Init & service management (systemd)

systemd is PID 1 and the service manager. Mental model: **units** (services, sockets, timers, mounts, targets) with **dependencies**, supervised inside **cgroups** (so it always knows every process a service spawned), logging to **journald**.

```bash
systemctl status <unit>           # state + recent logs in one view
systemctl cat <unit>              # the actual unit file (stop guessing paths)
systemctl edit <unit>             # drop-in override; daemon-reload after manual edits
systemctl --failed                # everything broken right now — triage start
systemctl list-dependencies <unit>
journalctl -u <unit> -b           # this unit, this boot
journalctl -k -b -1               # kernel log from the PREVIOUS boot (post-crash gold)
journalctl -p err --since "1h ago"
```

> [!tip] Make the journal persistent
> Default is often volatile (`/run/log/journal`, lost on reboot). For any box you'd investigate: create `/var/log/journal` and set `Storage=persistent` in `journald.conf`, then restart `systemd-journald`. Without this, a reboot erases the evidence.

**Security view — systemd as a sandbox.** Modern unit files are a hardening surface. Key directives: `NoNewPrivileges=yes`, `ProtectSystem=strict` + `ProtectHome=yes`, `PrivateTmp=yes`, `ReadOnlyPaths=`/`InaccessiblePaths=`, `CapabilityBoundingSet=` (drop all but what's needed), `SystemCallFilter=` (seccomp), `RestrictAddressFamilies=`. Audit any unit's exposure with **`systemd-analyze security <unit>`** — it scores and explains each gap. journald vs **auditd**: the former is general logging, the latter is the kernel audit subsystem for security events (syscalls, file/credential access) — run both.

---

## 8. Containers & isolation (the platform core)

**A container is not a kernel object.** It's a normal process the host can see in `ps`, wrapped in:

- **Namespaces** (what it *sees*) — 8 of them: `mnt`, `pid`, `net`, `uts`, `ipc`, `user`, `cgroup`, `time`. Each virtualizes one global resource.
- **cgroups v2** (what it *can use*) — CPU, memory, I/O, pids limits + accounting, one unified hierarchy.
- **Filesystem** — usually **overlayfs** layering image + writable top.
- **Confinement** — **seccomp** profile (syscall allowlist), **capabilities** dropped, often a **user namespace** so container-root maps to an unprivileged host uid (rootless).

```bash
lsns                       # list namespaces and their member processes
nsenter -t <pid> -n -m     # enter a container's net+mnt namespace from the host
unshare --net --pid --fork bash   # hand-roll a namespace to learn how it works
cat /sys/fs/cgroup/<...>/memory.max   # a container's actual memory cap
```
Debugging containers is mostly **debugging from the host**: the processes, fds, cgroup files, and namespaces are all visible there — `nsenter` lets you bring host tools *into* the container's view when the container lacks them.

**Security view.** Container isolation is exactly the sum of the pieces above; weaken one and you weaken the boundary. The dangerous knobs: `--privileged`, adding `CAP_SYS_ADMIN` ("the new root"), sharing host namespaces (`--pid=host`, `--net=host`), or mounting the host filesystem / Docker socket — each is a documented escape path. Defense: least privilege (drop caps, tight seccomp), user namespaces, read-only rootfs, and no host-socket mounts.

---

## 9. Observability & debugging — the meta-method

Tools are easy; **method** is the skill.

**The layered triage (the through-line of this whole note):**
1. **Identify the layer.** Hardware? kernel/driver? a service? the app? the network?
2. **Find the *first* error, not the loudest.** Read bottom-up: `dmesg`/`journalctl -k` (kernel) → `journalctl -u` (service) → app logs.
3. **Reproduce → isolate → bisect.** Make it happen on demand, shrink the variables, then binary-search the cause (`git bisect` for code regressions; boot an older kernel to bisect kernel/firmware regressions).
4. **Reset at the right layer.** And remember **state persists** — a warm reboot doesn't power-cycle peripherals or clear firmware/BMC/TPM state. When a "clean restart" doesn't clear a fault, ask *what didn't actually lose power.*

**The USE method for any resource** (Brendan Gregg): check **U**tilization, **S**aturation, **E**rrors. Apply it to CPU, memory, disk, and network in turn and you rarely miss a bottleneck.

**The tool ladder** — start broad, descend to tracing only as needed:

| Resource | Top-level | Per-subsystem | Tracing |
|---|---|---|---|
| CPU | `top`, `uptime` (load) | `mpstat`, `pidstat` | `perf top`, flame graphs |
| Memory | `free`, `vmstat` | `pmap`, `smaps` | `bpftrace`, `valgrind` |
| Disk/IO | `iostat` | `iotop`, `/proc/mounts` | `biolatency` (bcc) |
| Network | `ss`, `ip -s link` | `nstat`, `/proc/net` | `tcpdump`, `tcplife` (bcc) |
| Syscalls | — | `strace`, `ltrace` | `bpftrace`, `perf trace` |

**eBPF** (`bpftrace`, `bcc`) is the modern unifier: safe, programmable, in-kernel observability that ties all of the above together with minimal overhead. Worth investing in.

---

## 10. The security engineer's cross-cutting layer

Pulling the security threads into one place — the recurring theme is **privilege boundaries and defense in depth**.

- **Privilege model.** Ring 0 vs 3; the syscall is the only sanctioned crossing. Root is being decomposed into **capabilities** (`CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, …) — grant the few needed, drop the rest (`capsh --print` to inspect).
- **Reduce kernel attack surface.** `seccomp` (syscall allowlists), module signing + **lockdown**, minimal drivers.
- **Mandatory access control.** SELinux / AppArmor (LSMs) confine processes beyond Unix permissions; LSMs can stack.
- **Audit & evidence.** `auditd` for security events; persistent **journald** (+ Forward Secure Sealing: `journalctl --setup-keys` / `--verify`) so a reboot or tamper doesn't erase the trail; ship logs off-box.
- **Hardening sysctls worth knowing:**
  ```text
  kernel.kptr_restrict=1        # hide kernel pointers (anti-infoleak)
  kernel.dmesg_restrict=1       # restrict dmesg to privileged users
  kernel.yama.ptrace_scope=1    # limit who can ptrace what
  kernel.randomize_va_space=2   # full ASLR
  kernel.kexec_load_disabled=1  # block loading a new kernel at runtime
  ```
  (These are *why* your own unprivileged `dmesg`/`strace` may come back empty on a hardened box — the system working as intended.)
- **Hardware/firmware trust.** Peripherals run opaque firmware with **DMA**; constrain them with the **IOMMU** (`intel_iommu=on`/`amd_iommu=on`). Firmware ships via packages → it's part of your **supply chain**.
- **Boot integrity.** Secure Boot + measured boot + TPM attestation establish trust bottom-up; IMA/EVM extend it to runtime files.

---

## 11. Reusable debugging playbook

> [!example] The questions, in order
> 1. **What changed?** (deploy, upgrade, config, kernel, time/cert expiry, traffic) — most incidents have a trigger.
> 2. **Which layer?** hardware → kernel/driver → service → app → network.
> 3. **What's the *first* error?** Read logs bottom-up; ignore downstream noise.
> 4. **Is it resource-bound?** Run USE on CPU/mem/disk/net.
> 5. **Can I reproduce and isolate it?** Then bisect.
> 6. **What state persists across a restart?** (firmware, BMC, caches, conntrack, TPM) — a reboot may not clear it.
> 7. **Write it down.** (You're reading the output of step 7.)

**First-look command sweep:**
```bash
uptime; systemctl --failed                 # load + broken services
journalctl -p err -b --no-pager | tail      # recent errors, this boot
journalctl -k -b -1 | tail                  # kernel log from a crash-then-reboot
dmesg -T | grep -iE 'error|fail|oom'        # hardware/kernel trouble
free -h; df -h; df -i                       # memory, disk space, inodes
ss -tulpn                                   # who's listening
top -H                                      # live CPU/mem by thread
```

---

## Related notes & further reading

- [[notes/Networking/container-networking-internals|Container Networking Internals]] — Linux network namespaces, bridges, iptables, and veth pairs
- [[notes/Linux/booting-linux-on-a-windows-pc|Booting Linux on a Windows PC]] — UEFI NVRAM, Secure Boot shim chain, and GRUB recovery chroot
- [[notes/K8s/node-log-pipeline-silent-failures|Alive But Not Shipping]] — these tools applied to a real hang: using `top -H`, `strace -c`, `perf top` and `eu-stack` to prove a process is spinning in userspace rather than blocked in a syscall

External anchors worth bookmarking: `man7.org` (the canonical man pages), LWN.net (kernel development), and Brendan Gregg's site (performance & tracing methodology).
