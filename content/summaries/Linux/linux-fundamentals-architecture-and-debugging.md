---
title: "Summary: Linux Fundamentals, Architecture & Debugging"
---

> **Full notes:** [[notes/Linux/linux-fundamentals-architecture-and-debugging|Linux Fundamentals, Architecture & Debugging -->]]

## Key Concepts

### The Mental Model
- Linux is a stack: app → libc → **syscall boundary** → kernel → firmware → hardware
- The syscall is the only sanctioned crossing of user/kernel space — trace with `strace`, sandbox with `seccomp`
- "Everything is a file descriptor" — files, sockets, pipes, `/proc`, `/sys` all share the same `read`/`write` interface
- **Debug bottom-up**: first error from the bottom is the bug; everything above is a symptom

### Boot Sequence
- UEFI → GRUB (ESP) → kernel + initramfs → systemd (PID 1) → targets
- Kernel command line set in GRUB: `systemd.unit=rescue.target`, `nomodeset`, `init=/bin/bash` for recovery
- `systemd-analyze blame` (slowest units), `systemd-analyze critical-chain` (actual boot gate)
- **Secure Boot chain of trust:** firmware → shim (MS-signed) → GRUB (Canonical) → kernel; measured boot hashes stages into TPM PCRs

### Processes & Threads
- `fork()` (copy-on-write clone) + `exec()` (replace image) + `wait()` (reap child)
- Unkilled zombies = parent isn't reaping; container PID 1 must reap or zombies accumulate
- `D` state (uninterruptible sleep) = stuck in kernel I/O — cannot be `kill -9`'d; check `/proc/<pid>/stack` + `wchan`
- Scheduler: CFS → EEVDF (kernel 6.6+); cgroups cap CPU per group (basis of container limits)

### Memory
- Virtual address space per process; MMU translates via page tables; TLB caches translations
- Page cache = reclaimable buffer using "free" RAM — `buff/cache` in `free -h` is not lost memory
- OOM killer: triggered when physical RAM exhausted; victim chosen by `oom_score`; shows in `dmesg`
- ASLR (`kernel.randomize_va_space=2`) + W^X/NX are baseline exploit mitigations

### Filesystems & Storage
- VFS: inode (metadata) + dentry (name→inode cache) + file descriptor (process handle)
- "Disk full but `du` disagrees" = process holds open fd to a deleted file; `lsof | grep deleted`
- Mount options for hardening: `noexec`, `nosuid`, `nodev` (apply to `/tmp`, uploads, removable media)
- `df -i` for inode exhaustion (disk "full" with free space)

### Networking Layers
- `ip link` → `ip addr` → `ip route` → DNS (`dig`) → firewall (`nft`) → `tcpdump`
- `ss -tulpn` replaces `netstat`; shows listening sockets + owning process
- Control channel is **netlink** (`RTNETLINK`) — errors like "RTNETLINK answers:" are kernel replies
- nftables + conntrack = stateful firewall; `conntrack -L` shows live NAT table

### systemd
- Units (services/sockets/timers/mounts/targets) with dependencies, supervised inside cgroups
- `systemctl --failed` for triage; `journalctl -k -b -1` for kernel log from the *previous* boot (post-crash)
- Make journal persistent: create `/var/log/journal`, set `Storage=persistent` in `journald.conf`
- `systemd-analyze security <unit>` scores hardening exposure; key directives: `NoNewPrivileges`, `ProtectSystem=strict`, `SystemCallFilter=`

### Containers & Isolation
- Container = process wrapped in **namespaces** (what it sees) + **cgroups v2** (what it can use) + overlayfs + seccomp/caps
- 8 namespaces: `mnt`, `pid`, `net`, `uts`, `ipc`, `user`, `cgroup`, `time`
- Debug from **host**: `nsenter -t <pid> -n -m` brings host tools into container's namespace view
- Escape paths: `--privileged`, `CAP_SYS_ADMIN`, `--pid=host`/`--net=host`, host socket mounts

### Security Cross-Cutting
- Capabilities decompose root: grant only needed caps (`capsh --print` to inspect)
- Kernel lockdown mode: blocks `/dev/mem`, raw device pokes, unsigned modules
- `auditd` for security events (in addition to journald); ship logs off-box
- Key sysctls: `kernel.kptr_restrict=1`, `kernel.dmesg_restrict=1`, `kernel.yama.ptrace_scope=1`
- IOMMU (`intel_iommu=on`) constrains DMA from peripheral firmware

## Quick Reference

```
First-look sweep:
  uptime; systemctl --failed
  journalctl -p err -b --no-pager | tail
  journalctl -k -b -1 | tail           # crash → reboot evidence
  dmesg -T | grep -iE 'error|fail|oom'
  free -h; df -h; df -i
  ss -tulpn
  top -H

USE method (per resource): Utilization → Saturation → Errors

Process debug:
  strace -f -p <pid>      # every syscall
  lsof -p <pid>           # open fds/sockets
  cat /proc/<pid>/stack   # where kernel is blocked (D-state)
  cat /proc/<pid>/wchan

Memory debug:
  vmstat 1                # si/so = swap activity
  pmap -x <pid>

Disk debug:
  iostat -xz 1            # per-device util + latency
  lsof | grep deleted     # space held by open-but-deleted files

Network debug:
  ip a; ip r
  ss -tulpn
  sudo tcpdump -ni any port 443
  conntrack -L

Kernel modules:
  lsmod / modinfo <mod> / modprobe <mod>

Boot recovery:
  GRUB edit: append systemd.unit=rescue.target or init=/bin/bash
  journalctl -k -b -1     # previous boot kernel log
```
