---
title: "Anatomy of a Userspace Hang: From a Freed Pointer to a Frozen Node"
---

## Why this note exists

A log agent running as a DaemonSet stopped shipping logs on 101 nodes out of 852. The pods were `1/1 Running`, `restartCount` was 0, the process was alive, and the HTTP health endpoint returned `200`. On 78 of those nodes the container sat at a flat one core of CPU forever. The oldest had been like that for three and a half months.

Working out *why* meant reading a register file out of a running process, walking a linked list one 8-byte word at a time, and understanding what `free()` does and does not do to the bytes it hands back.

None of that is Kubernetes knowledge. It is operating-system knowledge, and if you came to infrastructure through YAML rather than through C, it is the part that is missing. This note builds each concept from nothing and then shows the exact evidence from the investigation that the concept explains.

The arc, in one line: **a pointer to freed memory kept a circular list walkable, the freed chunk was later reused as a node in a different list, and a once-per-second bookkeeping walk started going round a ring that no longer contained its own starting point.**

Every section is the same shape. First the concept, from scratch. Then "how it showed up", with the real numbers.

---

## 1. Processes, threads, and what a container really is

### The concept

A **process** is an address space plus the resources attached to it: open file descriptors, credentials, a current directory, signal handlers. Think of it as a private universe of memory with a program running in it.

A **thread** is a flow of execution. It has its own registers and its own stack, and that is nearly all it owns privately. Everything else, above all the heap, is shared with the other threads in the same process.

On Linux there is no separate "thread" object in the kernel. There is one structure, the **task**, and both processes and threads are tasks. They are created by the same syscall, `clone()`, and the only difference is which resources you ask it to share:

```
clone() with no sharing flags        -> a new process (this is what fork() does)
clone(CLONE_VM | CLONE_FILES |
      CLONE_SIGHAND | CLONE_THREAD) -> a new thread in the same process
```

Because of that, the naming gets confusing, and the confusion matters when you read `/proc`:

```
 TGID  "thread group id"  = what userspace calls the PID
 TID   "thread id"        = what the kernel calls a PID
                            every thread has its own; the main thread's TID == TGID

 /proc/<TGID>/            -> the process
 /proc/<TGID>/task/<TID>/ -> one thread inside it
```

The scheduler schedules **threads**, not processes. So "the process is using 100% CPU" is never the useful statement. "One thread out of six is using 100% and the other five are asleep" is.

### Containers are process trees, not boxes

A container is not a virtual machine and there is no boundary in the CPU that enforces it. A container is an ordinary process tree with three things wrapped around it:

```
 namespaces  change what the process can SEE   (PIDs, mounts, network, users, ...)
 cgroups     change what the process can USE   (CPU, memory, io)
 the rootfs  changes what "/" means for it
```

The one that matters here is the **PID namespace**. When the runtime creates a new PID namespace, the first process placed in it is numbered 1 inside that namespace, and it cannot see any process outside it. The same process simultaneously has a different, larger PID on the host. Two views of one task:

```
   host view                          container view
   /proc/1        systemd             /proc/1        fluent-bit   <- same task
   /proc/2841019  fluent-bit          (nothing else visible)
```

Being PID 1 in a namespace carries real duties, because PID 1 is where orphaned children get reparented and it must reap them, and it gets different default signal handling. But for debugging, the useful consequence is simpler: **inside the container, the workload is always `/proc/1`.** You never have to hunt for the PID.

### Thread names

Each task has a short name in `/proc/<tid>/comm`. The kernel field is 16 bytes including the terminating NUL, so **names are truncated at 15 characters**. That truncation is a real source of confusion: you see a name that looks like a typo and it is just the tail cut off.

### How it showed up

Sampling `/proc/1/task/` inside the hung container listed six threads. Reading each one's `comm`:

```
 TID   comm               what it is
 ────  ─────────────────  ─────────────────────────────────────────────────────────
    1  fluent-bit         the process main thread; spawns the engine, then waits
   14  flb-pipeline       THE ENGINE. one event loop: inputs, filters, flush
                          scheduling, retries, and the 1-second metrics timer
   15  flb-out-stackdr    output worker (truncated from "flb-out-stackdriver")
   16  flb-out-stackdr    second output worker
   17  monkey             the embedded HTTP server that serves the metrics and
                          health endpoints on :2020  ("Monkey" is the library name)
   18  flb-logger         drains the internal log ring buffer and writes it out
```

Two things to take from that listing before we go any further.

The HTTP server is a **different thread** from the engine. That is the entire reason a liveness probe against `:2020` kept passing while the node shipped nothing for three months.

And `flb-out-stackdr` appearing twice is not a bug, it is `workers 2` on the output plugin. Knowing that made the next measurement interpretable, because when we found both of them idle we could rule out the whole family of "the output is stuck on the network" theories.

---

## 2. User mode, kernel mode, and reading CPU time in ticks

### The concept

The CPU runs your code in one of two privilege levels. On x86-64 they are called rings, and the two that get used are:

```
 ring 3   USER MODE     your program. cannot touch hardware, cannot read
                        another process's memory, cannot change page tables
 ring 0   KERNEL MODE   the kernel. can do everything
```

A program in user mode that needs anything from the outside world, a byte from a file, a packet from a socket, more memory, has to ask. The asking mechanism is the **system call**. On x86-64 it is literally one instruction, `syscall`, which flips the CPU to ring 0 and jumps to a fixed kernel entry point. The kernel does the work, then returns to ring 3.

That boundary crossing is not free, roughly hundreds of nanoseconds, but more importantly for us it is **countable and attributable**. The kernel keeps two separate CPU-time counters per task:

```
 utime   time spent executing in USER mode   (your code, library code)
 stime   time spent executing in KERNEL mode (inside syscalls, page faults, ...)
```

Both are reported in **clock ticks**, not seconds. The tick rate exposed to userspace is `USER_HZ`, and on Linux/x86-64 it is **100**, meaning one tick is 10 ms. You can confirm it with `getconf CLK_TCK`. Do not confuse it with the kernel's internal `CONFIG_HZ`, which is often 250 or 1000; the `/proc` interface is normalised to 100 for backwards compatibility.

They live in `/proc/<tid>/stat` as fields 14 and 15. Counters, never gauges, so you sample twice and subtract.

The arithmetic to remember:

```
 one core, fully busy, for T seconds  =  100 * T ticks

 cores_used = (ticks_after - ticks_before) / (100 * seconds_elapsed)
```

And the reason to care about the *split* rather than the total:

```
 high utime, ~zero stime   -> pure userspace computation. a loop in your own code.
                              it is not waiting for anything and not asking the
                              kernel for anything
 high stime                -> syscall-heavy. a retry loop hammering read(),
                              or genuine heavy I/O
 both near zero            -> the thread is not running at all
```

### How it showed up

The decisive measurement in the whole investigation took three seconds and needed no tools beyond `cat`.

Two samples of `/proc/1/task/14/stat`, three seconds apart:

```
 thread 14 (flb-pipeline)
   sample 1      utime = 89_401_209   stime = 3_118_442
   sample 2      utime = 89_401_522   stime = 3_118_443
   delta         utime = +313         stime = +1
```

Read it:

```
 elapsed        3 s   =  300 ticks of one core
 utime         +313 ticks  ->  313 / 300  =  1.04 cores of USERSPACE execution
 stime         +1  tick    ->  10 ms in 3 s  =  0.3% of the time in the kernel
```

`1.04` rather than a clean `1.00` is just tick granularity and the sampling window not being exactly 3.000 seconds. The signal is unmistakable: **this thread is running flat out and it is not making system calls.**

That one ratio eliminated most of the candidate explanations in a single step:

```
 hypothesis                              predicted stime      observed
 ─────────────────────────────────────    ─────────────────    ────────────────
 EAGAIN read() retry spin on a socket     large, 30-70%        1 tick.  ruled out
 tight epoll_wait() with timeout 0        large                1 tick.  ruled out
 network retries to a dead endpoint       moderate             1 tick.  ruled out
 blocked on a lock (deadlock)             ~0, and utime ~0     utime is 313
 infinite loop in the program's own code  ~0                   MATCHES
```

The published, well-known hang in this agent is the level-triggered TLS retry spin, in which `read()` returns `EAGAIN` roughly 8,500 times a second forever. That would have shown up as a mountain of `stime`. One tick in three seconds said this was something else entirely, and that was the moment the investigation changed direction.

The same two samples on the other five threads showed `+0` on both counters. Five idle threads and one thread burning a core.

---

## 3. What a thread is doing when it is not making progress

### The concept

Every task is in exactly one state, and `ps`, `top` and `/proc` all report it as a single letter:

```
 R   Running or runnable.  Either on a CPU right now, or on a run queue
                           waiting for one. "R" does not mean "healthy".
 S   Interruptible sleep.  Waiting for something, and a signal can wake it.
                           This is what normal idling looks like. Most threads
                           in a healthy system are S almost all the time.
 D   Uninterruptible sleep. Waiting inside the kernel in a way that cannot be
                           interrupted, usually storage or NFS. Cannot be
                           killed, not even with SIGKILL. If you see D, look
                           at the device, not the process.
 Z   Zombie.               Exited, not yet reaped by its parent.
 T   Stopped.              SIGSTOP, or stopped by a debugger.
```

When a task is in `S` or `D` it is parked inside some kernel function, and `/proc/<tid>/wchan` names that function. This is the single most informative one-word answer about a sleeping thread. The three you will meet constantly:

```
 do_epoll_wait      parked in epoll_wait(). An event loop with nothing to do.
                    This is a HEALTHY idle event-loop thread.
 hrtimer_nanosleep  parked in nanosleep()/clock_nanosleep(). A deliberate sleep.
                    Polling loops and "check every N ms" loops look like this.
 futex_wait         parked on a futex, which is what userspace mutexes and
                    condition variables are built on. The thread is waiting for
                    ANOTHER THREAD to release something. This is the state that
                    deadlocks live in.
```

The distinction that trips people up:

```
 IDLE-WAITING       S in do_epoll_wait / hrtimer_nanosleep.  Nothing to do.
                    Normal. Costs no CPU. Wakes instantly when work arrives.

 BLOCKED ON A LOCK  S in futex_wait.  Has work to do, cannot proceed until
                    someone else releases a lock. Costs no CPU.
                    Suspicious if it never changes.

 SPINNING           R, high utime.  Executing continuously and achieving
                    nothing. Costs a whole core.
```

### Three failure modes people use interchangeably, and should not

```
 DEADLOCK
   Two or more threads each hold a lock the other needs, so none can advance.
   Circular wait, and it is permanent by construction.
   Signature: every involved thread is S in futex_wait. Total CPU ~0.
   Nothing is executing. The process is frozen but perfectly quiet.

 INFINITE LOOP
   ONE thread is executing a loop whose exit condition can never become true.
   No other thread is required, no lock is involved, nothing is contended.
   Signature: one thread R, ~100% of one core in utime, near-zero stime,
   and the instruction pointer confined to a few bytes of code.

 LIVELOCK
   Threads are running and changing state, but the state changes undo each
   other so no net work completes. Think two threads politely backing off and
   retrying in lockstep forever.
   Signature: CPU is burning AND syscalls or state transitions are happening.
   Looks busy. This is the one that most resembles "working, just slowly",
   which is what makes it the nastiest of the three to name.
```

A useful way to hold it: a deadlock is a *waiting* failure, an infinite loop is a *computing* failure, and a livelock is a *coordinating* failure. The CPU and syscall profile tells them apart before you look at a single line of code.

### How it showed up

State and `wchan` for all six threads of the hung pod:

```
 TID  comm              state  wchan               reading
 ───  ────────────────  ─────  ──────────────────  ──────────────────────────────
   1  fluent-bit        S      futex_wait          main thread parked waiting for
                                                   shutdown. normal, by design.
  14  flb-pipeline      R      (empty)             RUNNING. not sleeping anywhere.
  15  flb-out-stackdr   S      do_epoll_wait       idle event loop. no work queued.
  16  flb-out-stackdr   S      do_epoll_wait       idle event loop. no work queued.
  17  monkey            S      do_epoll_wait       idle HTTP server, waiting for
                                                   the next probe to answer.
  18  flb-logger        S      futex_wait          waiting on the log ring buffer,
                                                   which nobody is writing to.
```

`wchan` is empty for thread 14 because it is not sleeping in the kernel at all. That is consistent with the tick measurement and it is what you should expect for a spinning thread.

This table killed the deadlock hypothesis outright. A deadlock needs at least two threads blocked on each other. Here the two output workers were in `do_epoll_wait`, which is not "blocked on a lock", it is "idle, nothing has been handed to me". They were not waiting for the engine to release anything. They simply had no flushes to run, because the thread that schedules flushes had stopped scheduling.

So: **infinite loop in one thread**, with the rest of the process healthy and idle downstream of it. Not a deadlock, not a livelock, not backpressure.

### The variant we never examined

The fleet sweep found two distinct shapes among the 101 non-shipping nodes:

```
  78 nodes   container CPU flat at ~1000 m, zero variance   <- the spin, analysed here
  23 nodes   container CPU flat at EXACTLY 0 m              <- never investigated
  (13 more nodes reported zero output but were GPU nodes that were genuinely idle,
   which is a reminder that "no output" needs "and it should have had input")
```

The 23 zero-CPU pods were also `1/1 Ready` with no restarts, and they were also shipping nothing. Zero CPU with work outstanding is the deadlock signature from the table above, or a thread parked in `D`. It is a *different bug* and it is still open, and the honest thing to say is that we know the shape and not the cause.

The recipe to close it is the same three commands: sample `stat` twice and expect `+0` on both counters, then read every thread's state and `wchan`. If they are all `S` in `futex_wait`, it is a lock cycle and the next step is finding which two. If any thread is `D`, the problem is below the process, in the kernel or a device.

Worth noticing how much cheaper the flat-CPU heuristic was than it was accurate. Searching for pods pinned near one core found 54 of the 101. It missed all 23 zero-CPU pods and some of the spinning ones. Symptom-based detection found a bit over half the population; the output-rate detector found all of it. More on that in section 11.

---

## 4. /proc: the kernel's answer sheet, written when you read it

### The concept

`/proc` is not a directory of files. It is a **pseudo-filesystem**: the kernel registers a set of paths, and when you `read()` one, a kernel function runs and formats the answer *at that moment*. Nothing is stored. That is why:

```
 $ ls -l /proc/1/stat
 -r--r--r-- 1 root root 0 Sep  7 09:14 /proc/1/stat     <- size 0. it has no size.
```

The size is 0 because there are no bytes until you ask. Two consequences that matter in practice. Every read is a fresh, consistent-per-field snapshot, so `/proc` is the cheapest possible sampling interface, no agent required. And a `cat` of a `/proc` file is a syscall into the kernel, so you can sample a process from a container that contains nothing but `busybox`.

### The files worth knowing by heart

```
 /proc/<tid>/comm      the 15-character thread name. one line.

 /proc/<tid>/stat      the big one. ~52 space-separated fields.
                       field 1  = pid
                       field 2  = comm, WRAPPED IN PARENTHESES
                       field 3  = state letter
                       field 14 = utime  (ticks)
                       field 15 = stime  (ticks)
                       field 23 = vsize  (bytes)
                       field 24 = rss    (pages)
                       field 30 = kstkeip, the saved user instruction pointer

 /proc/<tid>/wchan     the kernel symbol the task is sleeping in. empty if running.

 /proc/<tid>/syscall   the syscall currently in progress:
                         "<nr> <arg0..arg5> <sp> <pc>"   if inside a syscall
                         "running"                        if in userspace
                         "-1 <sp> <pc>"                   if not in a syscall
                       needs PTRACE_MODE_ATTACH permission to read.

 /proc/<pid>/maps      every mapped memory region: address range, permissions,
                       file offset, and backing file. This is how you find where
                       the executable actually landed in memory.

 /proc/<pid>/task/     one subdirectory per thread. the whole per-thread story.

 /proc/<pid>/stack     the KERNEL stack. root only, and only useful for a task
                       that is inside the kernel. Useless for a userspace loop.
```

### The parsing trap in `stat`

Field 2 is the thread name in parentheses, and a thread name **may contain spaces and may contain parentheses**. So splitting the line on whitespace and indexing gives you the wrong fields for any process named `(sd-pam)` or `Chrome Helper`. This is a real bug in a large amount of monitoring code.

The fix is to cut at the **last** `)` in the line, then index from there:

```sh
# everything after the last ')' -- state becomes field 1, utime 12, stime 13
sed 's/.*) //' /proc/1/stat | awk '{print "state="$1, "utime="$12, "stime="$13}'
```

### The field that lies: kstkeip

Field 30 of `stat` is `kstkeip`, documented as the current instruction pointer. If it worked, finding a spinning loop would be a one-liner.

It does not work. On modern kernels it **reads 0 even for root, even with `CAP_SYS_PTRACE`**. Two reasons stack up. Since Linux 4.1 the field is gated behind `PTRACE_MODE_ATTACH_FSCREDS`, so an unprivileged reader always gets 0. And for a task that is *currently executing on another CPU* there is no reliable saved user-mode register frame to read, because the registers are live in the hardware, not spilled to the kernel stack. The kernel returns 0 rather than something misleading.

Which is exactly the case we care about. The thread we want to inspect is `R`, running, right now.

```
 want:  the instruction pointer of a running thread
 /proc: returns 0, because the thread is running
 => you must STOP the thread to read its registers, and the only interface
    that stops a thread and hands you its register file is ptrace()
```

That dead end is the reason section 9 exists.

### How it showed up

The first pass at the hung pod was a pure `/proc` sampler, from a `busybox` container, no compiler, no debugger, no `top`:

```sh
# run twice, 3 seconds apart, inside a container sharing the target's PID namespace
for t in /proc/1/task/*; do
  tid=$(basename "$t")
  comm=$(cat "$t/comm")
  vals=$(sed 's/.*) //' "$t/stat" | awk '{print $1, $12, $13}')
  wchan=$(cat "$t/wchan" 2>/dev/null)
  printf '%-5s %-16s state/utime/stime=%-24s wchan=%s\n' "$tid" "$comm" "$vals" "$wchan"
done
```

Sixteen lines of shell produced the thread inventory in section 1, the tick deltas in section 2, and the state/`wchan` table in section 3. That is three of the four load-bearing pieces of evidence, from `cat` and `awk`.

`/proc/1/task/14/syscall` returned `running`, which corroborated the near-zero `stime` independently. And `kstkeip` was `0`, as promised, which is what forced the escalation to `ptrace`.

---

## 5. Event loops, epoll, and coroutines

### The concept, starting from blocking I/O

The naive way to serve many connections is a thread per connection. Each thread calls `read()`, the kernel parks it until data arrives, and it wakes up and handles it. Simple, and it falls over at scale, because each thread costs a stack (typically 8 MB of virtual address space, and real memory for what it touches) and every switch between them is a trip through the kernel scheduler.

The alternative is **one thread watching many file descriptors**. Ask the kernel "tell me which of these thousand fds are ready", handle those, repeat. The modern Linux interface for that is `epoll`:

```c
 int ep = epoll_create1(0);                        /* make an epoll instance   */
 epoll_ctl(ep, EPOLL_CTL_ADD, fd, &event);         /* register interest in fd  */
 int n = epoll_wait(ep, events, maxevents, timeout);/* block until some are ready */
```

`epoll_wait` is the *only* place a healthy event-loop thread sleeps. Which is why `wchan = do_epoll_wait` reads as "idle and fine".

An **event loop** is then the whole program, structurally:

```c
 while (running) {
     n = epoll_wait(ep, events, MAX, timeout);     /* sleep here               */
     for (i = 0; i < n; i++) {
         handler_for(events[i])(events[i]);        /* run the callback          */
     }
 }
```

Two properties of that loop are the entire story of this note.

It is **cooperative**. Nothing preempts a callback. The kernel can move the thread off the CPU, but no other callback runs until the current one returns. If one callback never returns, the loop never comes back round, and every fd registered on it is never serviced again.

And **timers are just fds**. A `timerfd` becomes readable when it expires, so you register it in the same epoll set and a periodic task becomes an ordinary callback. Elegant, and it means periodic work shares the fate of every other callback on that loop. A one-second timer and a network socket are not isolated from each other in any way.

### Level-triggered vs edge-triggered

Worth a paragraph because it explains the *other* famous hang in this software.

```
 LEVEL-TRIGGERED (the default)
   epoll_wait reports an fd as ready as long as the CONDITION HOLDS.
   Did not drain the socket? It is reported ready again immediately.

 EDGE-TRIGGERED (EPOLLET)
   epoll_wait reports an fd only when the state CHANGES.
   Did not drain the socket? You may never hear about it again.
```

Level-triggered is safer and it has a sharp edge: if a callback is told "ready", tries to read, gets `EAGAIN`, and goes back to the loop without changing anything, the loop hands it the same fd again with no delay. That is a spin at full speed with no error recorded anywhere. It is the known TLS retry-spin hang, and it is *syscall-heavy*, which is precisely why the `+1 stime` tick ruled it out here.

### Coroutines

A **coroutine** is a function that can pause in the middle and be resumed later, from the same point, with its local variables intact. It needs its own stack, and switching between coroutines means saving the stack pointer and callee-saved registers and loading another set. That is maybe 20 instructions and it never enters the kernel, which makes it perhaps a hundred times cheaper than a thread switch.

The trade is stated in the name: **cooperative** scheduling. A coroutine runs until it *chooses* to yield.

```c
 /* inside a flush coroutine, conceptually */
 ret = net_write(conn, buf, len);
 if (ret == WOULD_BLOCK) {
     coro_yield();          /* <- the ONLY place control leaves this coroutine */
 }
```

The convention is to yield where you would have blocked, so an output plugin's flush reads like straight-line blocking code while actually interleaving with dozens of other flushes at every I/O boundary.

The failure mode is the mirror of the benefit. A coroutine that does a long stretch of pure computation, or gets stuck in a loop with no I/O in it, **has no yield point**, so it is never descheduled and nothing else on that loop runs. Preemptive threads do not have this problem; cooperative schedulers trade that safety for speed.

### How it showed up: what one thread was carrying

The engine thread owned, on one event loop:

```
 flb-pipeline (one epoll set)
  ├── input collectors        tail read timer, directory scan timer,
  │                           rotation watch, inotify fds
  ├── ALL filters             metadata enrichment runs here, always, by design
  ├── flush scheduling        decides which buffered chunk goes to which output
  ├── the retry scheduler     the timers behind "retry in 7 seconds"
  └── the metrics timer       fires every 1 second, copies the whole metrics tree
```

When that loop stopped returning, here is the full cascade, and each line is a symptom someone actually reported:

```
 epoll_wait is never called again
   -> the tail read timer never fires        -> no log lines are read.
                                                Every pod on the node, at once.
   -> inotify events pile up unread          -> file rotations are not noticed
   -> flushes are never scheduled            -> the two output workers sit in
                                                do_epoll_wait with nothing to do.
                                                They LOOK healthy. They are idle.
   -> retry timers never fire                -> "retry in 7 seconds" never happens.
                                                The last log line the agent ever
                                                printed is a lie about the future.
   -> the metrics timer never fires again    -> every plugin counter FREEZES at
                                                its last value, forever
   -> no code path produces log lines        -> flb-logger has nothing to drain.
                                                The silence is not a broken logger.

 meanwhile, on a DIFFERENT thread:
   monkey keeps its own epoll loop           -> :2020 answers every request
   /api/v1/uptime is computed at request     -> uptime keeps CLIMBING
     time as (now - init_time)
   /api/v1/metrics returns the last          -> counters are FROZEN
     snapshot the engine pushed
```

That last pair is the cheapest remote fingerprint of this failure and it is *structural*, not coincidental. Uptime is computed by the thread answering the request. Counters are produced by a timer on the thread that died. **Uptime climbing while record counters sit still** cannot happen in any other way.

And it is why an HTTP liveness probe is worthless here. The probe asks the `monkey` thread whether the `monkey` thread is alive. It always is.

One more thing to notice, because it is the bridge to the rest of the note. The stuck callback was the **metrics timer**. Not a filter, not a parser, not the network path. A once-per-second bookkeeping walk over the agent's own counters is what took down logging for 146 application pods on that node.

---

## 6. Memory: stack, heap, and what free() actually does

### The concept

A process's virtual address space has a few distinct regions, and the two that hold your data behave completely differently:

```
   high addresses
   ┌───────────────────────────────┐
   │ stack (main thread)           │  grows DOWN
   │   ↓                           │
   ├───────────────────────────────┤
   │ ... unmapped ...              │
   ├───────────────────────────────┤
   │ mmap'd regions:               │
   │   shared libraries            │
   │   thread stacks               │
   │   large malloc allocations    │
   ├───────────────────────────────┤
   │ heap                          │  grows UP (brk)
   │   ↑                           │
   ├───────────────────────────────┤
   │ .bss    zero-initialised data │
   │ .data   initialised globals   │
   │ .text   the machine code      │  read + execute, not write
   └───────────────────────────────┘
   low addresses
```

**The stack** is automatic. Calling a function pushes a frame holding its local variables and the return address; returning pops it. Lifetime is exactly the function call, the compiler handles all of it, and each thread gets its own.

**The heap** is manual. You ask for memory, you get an address, and it is yours until you give it back. Lifetime is whatever you decide, and deciding wrongly is the subject of this section.

```c
 char *p = malloc(48);   /* "give me 48 bytes"; returns an ADDRESS, or NULL */
 strcpy(p, "403");       /*  use it                                        */
 free(p);                /* "I am done with these 48 bytes"                */
```

A **pointer** is nothing but a number that happens to be an address. `p` is a 64-bit integer. That is the whole of it, and every consequence below follows from it.

### What free() actually does

Here is the part that people who have not written C tend to get wrong, and it is the crux of this entire investigation.

`free(p)` does **not** erase the bytes. It does not tell the kernel anything. It does not unmap the page. All it does is record, in the allocator's own bookkeeping, that the chunk starting at `p` is available to hand out again.

glibc's allocator keeps freed chunks on per-size-class free lists. The fast ones are the **tcache** (a per-thread cache, 64 size classes, up to 7 chunks each by default) and the **fastbins**. Both are singly linked lists, and here is the trick: **the allocator stores the link pointers inside the freed chunk itself**, because the chunk is not in use, so its bytes are spare storage.

For tcache, it writes two 8-byte words at the start of the user area:

```
 offset  0  8   16                                              48
        ┌───┬───┬──────────────────────────────────────────────┐
 before │ '4' '0' '3' '\0'  ... whatever else you stored ...    │  in use
        └───┴───┴──────────────────────────────────────────────┘

        ┌───────┬───────┬──────────────────────────────────────┐
 after  │ next  │ key   │ ... UNTOUCHED. still your old bytes. │  freed
 free() └───────┴───────┴──────────────────────────────────────┘
          ^^^^^^^^^^^^^^
          only these 16 bytes are overwritten
```

So a freed 48-byte chunk still contains, verbatim, whatever was in bytes 16 through 47 when you freed it. Nobody wipes it. Wiping would cost cycles for no benefit to a correct program.

Reuse is **LIFO** on those lists, which is a performance decision (the most recently freed chunk is the most likely to still be in cache) with a debugging consequence: the *next* allocation of that size class very often gets back the chunk you just freed. Freed memory is not reused eventually and at random. It is frequently reused almost immediately.

### The three bugs that follow

```
 DANGLING POINTER
   A pointer variable still holds an address whose object's lifetime has ended.
   The pointer itself is not wrong in any detectable way. It is a number, and
   it is the same number it always was.

 USE-AFTER-FREE
   Dereferencing a dangling pointer: reading or writing through it.
   This is UNDEFINED BEHAVIOUR, which in practice means one of:
     - it silently works, returning the stale bytes           (most of the time)
     - it returns bytes belonging to a DIFFERENT object       (after reuse)
     - it faults, if the allocator returned the page to the kernel

 DOUBLE FREE
   free()ing the same chunk twice. Now it appears twice on the free list, so
   two future allocations get the SAME address, and the two owners silently
   overwrite each other. glibc's tcache "key" field exists to catch the
   simplest form of this, which is why the second word gets clobbered.
```

### Why freed memory keeps working, and why that is the real danger

This deserves stating plainly because it is the counterintuitive part.

```
 t0   node = malloc(24);  node->next = X;      /* valid, obviously */
 t1   free(node);                              /* chunk goes on the tcache list.
                                                  next 16 bytes overwritten.
                                                  bytes 16..23 preserved. */
 t2   read node->next  ->  still returns X     /* NO SYMPTOM. page still mapped,
                                                  bytes still there, and the read
                                                  is 8 bytes at a valid address. */
 t3   other = malloc(24);                      /* allocator hands back the SAME
                                                  chunk. other != node as far as
                                                  the program is concerned, but
                                                  they are the same 24 bytes. */
 t4   other->next = Y;                         /* legitimately writes offset 8 */
 t5   read node->next  ->  now returns Y       /* SYMPTOM, at last. And it looks
                                                  nothing like a memory bug. */
```

The gap between `t1` and `t3` can be milliseconds or three months. Nothing detects the error at `t1`, and nothing detects it at `t2`. The bug becomes visible only when an unrelated allocation of the same size happens, which is why use-after-free bugs are famously non-reproducible and famously load-dependent.

### "Memory corruption" does not have to mean a scribble

The mental picture most people carry for corruption is a wild write: a buffer overflow spraying bytes over neighbouring structures, leaving obvious garbage behind.

That is one kind. It is not the interesting kind, and it is not what happened here.

The other kind is a **violated invariant reached through a stale reference**. Every byte is well-formed. Every pointer points at a real, correctly typed, allocated object. Nothing looks damaged in a hex dump. What is wrong is the *graph*: two owners believe they own the same node, so a structure that must satisfy a property, say "this list is a ring that contains its own head", no longer does.

```
 SCRIBBLE                              STALE REFERENCE
 ────────────────────────────────      ──────────────────────────────────────
 bytes are garbage                     every byte is a valid value
 pointers point at nonsense            every pointer points at a real object
 obvious in a dump                     invisible in a dump
 usually crashes fast                  can hang, or work for months
 caught by ASan/valgrind writes        caught only by lifetime tracking
```

Hold on to that distinction. Section 8 corrects an earlier version of this investigation that assumed a scribble, precisely because a hex dump *looked* like one.

---

## 7. Intrusive circular doubly linked lists

### The concept

A linked list you might write in Python or Go is **non-intrusive**: the list owns node objects, and each node holds a pointer to your payload. Every insertion allocates a node.

C systems code overwhelmingly uses the other arrangement, **intrusive** lists, where the link fields are embedded *inside* the payload struct:

```c
/* the link. two pointers, nothing else. no payload, no type, no length. */
struct cfl_list {
    struct cfl_list *prev;
    struct cfl_list *next;
};
```

This is the exact shape of the Linux kernel's `list_head`, and libraries copy it deliberately. `cfl_list` and `mk_list` in this codebase are the same idea under different names.

Embedding it looks like:

```c
struct thing {
    char             *value;
    struct cfl_list   _head;   /* the link lives INSIDE the thing */
};
```

The advantages are real: zero allocations for list membership, no extra indirection when walking, and one struct can be on several lists at once by embedding several links.

Getting from a link back to its owner is pointer arithmetic. Given a `struct cfl_list *p` you know is the `_head` of a `struct thing`, subtract the field's offset:

```c
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))
```

That is what `cfl_list_entry` and the kernel's `list_entry` expand to. It also means **the addresses stored in the list are not the addresses of your objects**, they are the addresses of a field inside them. Remember that when you walk one by hand.

### Circular, with the head as a sentinel

Now the part that produces the hang.

These lists are **circular** and the head is a **sentinel node embedded in the owner**, not a pointer to the first element. An empty list points at itself:

```
 empty list:                       three elements:

   ┌─────────────┐                   ┌──────────────────────────────────┐
   │             │                   │                                  │
   ▼             │                   ▼                                  │
 ┌──────┐        │                 ┌──────┐   ┌───┐   ┌───┐   ┌───┐     │
 │ head │────────┘                 │ head │──▶│ A │──▶│ B │──▶│ C │─────┘
 └──────┘                          └──────┘   └───┘   └───┘   └───┘
 head.next == head.prev == &head    and every prev pointer runs the other way
```

Why circular and sentinel-based? Because it removes every special case. Insertion at the head, insertion at the tail, and deletion all become the same four pointer assignments with no `if (list is empty)` branch and no NULL checks. `list_del` is famously branch-free for this reason. It is a genuinely excellent design and it has one sharp condition attached.

### The invariant, and the only terminator there is

Walking such a list is always this:

```c
for (it = head->next; it != head; it = it->next) { ... }
```

Read that termination condition carefully. The loop stops when `it` **is identically the head address**. There is no other exit:

```
 no NULL terminator      -- a well-formed ring contains no NULLs anywhere
 no length field         -- the list does not know how long it is
 no bounds check         -- there is nothing to bound against
 no iteration cap        -- adding one would be considered a bug, not a safeguard
```

So the *entire* correctness of every list walk rests on one invariant: **following `next` from the head must eventually arrive back at the head.**

Break that invariant and you get exactly two outcomes, decided by what the broken pointer happens to point at:

```
 stale next -> unmapped or non-canonical address
    the dereference faults. the MMU has no valid mapping.
    -> SIGSEGV. loud, immediate, restarts the container.

 stale next -> a VALID ring that does not contain the head
    every dereference succeeds. every pointer is real.
    the walk goes round and round.
    -> INFINITE LOOP. silent, permanent, one core burnt forever.
```

**Same defect. Two symptoms. The difference is luck about heap layout.** That sentence is the thesis of this note, and it explains why the fleet showed thousands of segfaults per day *and* a hundred permanently frozen pods: those were not two bugs, they were one bug rolling dice.

### The code, and the seven instructions it becomes

The function the hung thread was inside is a list length count. It is about as simple as C gets:

```c
static inline int cfl_list_size(struct cfl_list *head)
{
    int ret = 0;
    struct cfl_list *it;

    for (it = head->next; it != head; it = it->next) {
        ret++;
    }

    return ret;
}
```

Compiled for x86-64, the loop body is a handful of instructions with no calls and no syscalls in it. Shape, with the two offsets the sampler actually landed on marked (exact offsets depend on the compiler and build, the shape does not):

```asm
<cfl_list_size>:
  +0x00    endbr64
  +0x04    xor    %eax,%eax             ; ret = 0
  +0x06    mov    0x8(%rdi),%rdx        ; it = head->next      (next is at +8)
  +0x0a    cmp    %rdi,%rdx             ; it == head ?
  +0x0d    je     <+0x40>               ; empty list -> return 0

           ; ---- the loop. seven instructions. never leaves the CPU. ----
  +0x25    add    $0x1,%eax             ; ret++                <-- 50 of 60 samples
  +0x28    mov    %rdx,%rcx             ;
  +0x2b    test   %rcx,%rcx             ;
  +0x2e    je     <+0x40>               ;
  +0x31    nop                          ;
  +0x35    mov    0x8(%rdx),%rdx        ; it = it->next         <-- 10 of 60 samples
  +0x39    cmp    %rdi,%rdx             ; it == head ?
  +0x3c    jne    <+0x25>               ; no -> go round again
           ; --------------------------------------------------------------
  +0x40    ret
```

Three things about that block are worth internalising.

There is **no syscall**, no `call`, no memory allocation. Nothing in it can be observed by `strace`. This is why `strace -c` on the hung process printed an empty table, and why anyone using `strace` as their first tool concluded the process was doing nothing at all.

The whole loop is a couple of dozen bytes, so it fits in L1 instruction cache and the two data loads hit the same few cache lines. It runs as fast as the machine can go, which is what turns "a bug" into "exactly 1.00 cores".

And `ret` is an `int` that keeps incrementing. After about 2.1 billion iterations it overflows to negative, which is undefined behaviour and, on this path, entirely irrelevant. The loop was never going to end either way.

### How it showed up

Reading the actual list out of the hung process, one word at a time, gave this:

```
 the stuck metric's list head lives at   0x...247358
 head->next                              ->  A
 A->next                                 ->  B
 B->next                                 ->  C
 C->next                                 ->  A        <-- back to A. never to head.

   ┌──────────┐
   │  head    │ (embedded in the stuck metric)
   │ 0x247358 │
   └────┬─────┘
        │ next
        ▼
      ┌───┐      ┌───┐      ┌───┐
      │ A │─────▶│ B │─────▶│ C │
      └───┘      └───┘      └───┘
        ▲                     │
        └─────────────────────┘
                next

   walk order: A, B, C, A, B, C, A, B, C, ...
   `it != head` is true every single time, because head is not in the ring.
```

A three-node ring, and the head dangling off the outside of it pointing in. Every pointer valid, every dereference successful, no fault, no error, no log line. Just seven instructions, forever.

Now the question that took another day to answer: *why* is the head pointing into a ring it is not part of?

---

## 8. The metrics model, and the corrected mechanism

### The concept: what a labelled counter is made of

A **counter** in this model is one number that only goes up. Not a series, not a histogram, one `double`.

Labels are what make it useful. `requests_total{status="200"}` and `requests_total{status="403"}` are two separate counters that share a name and a set of label *keys*, and differ in their label *values*. The library keeps that as a small tree:

```
 cmt_context                     the whole registry for this process
   └── cmt_counter               one metric family: name, help, label KEYS
         └── cmt_map             the set of instances
               ├── label_keys    list of key names:      ["status", "name"]
               └── metrics       hash-keyed by the label values, each entry:
                     cmt_metric
                       ├── value        the number
                       └── labels       list of label VALUES: ["403", "out.0"]
```

And a label value node is exactly the intrusive struct from section 7:

```c
struct cmt_map_label {
    cfl_sds_t         name;    /* the string: "403", "out.0"        */
    struct cfl_list   _head;   /* the embedded link, at offset +8   */
};
```

So the label values of one metric instance are a small circular doubly linked list, typically two or three nodes plus the head, and the head is embedded in the `cmt_metric` struct itself.

The crucial mechanic: **a label value combination that has never been seen before allocates a brand new `cmt_metric` and a brand new set of `cmt_map_label` nodes.** Counters appear at runtime, driven by data.

### The once-a-second full walk

The exporter cannot hand out live pointers into structures another thread mutates, so once a second the engine takes a **copy** of the whole registry, `cmt_cat` (concatenate contexts) being the entry point:

```
 flb_engine_start                 the engine event loop
   flb_me_fd_event                the 1-second metrics timer fires
     collect_metrics
       flb_me_get_cmetrics
         cmt_cat                  copy the whole registry into a snapshot
           append_context         for each context
             copy_counter         for each metric family
               copy_map           for each map
                 copy_label_values    for each metric instance
                   cfl_list_size(&metric->labels)   <-- walk the label list
```

That last line is the hot spot. To copy a label list, it first counts it. **Every metric instance's label list is walked, in full, every single second.** So this path is a periodic integrity check over the entire metrics tree that nobody designed as one. It is the first code to trip over any damage, which is why it appears at the top of both the crash stacks and the hang.

> A periodic full-structure walk is a corruption **detector**. The function at the top of the stack is whoever tripped over the damage, not whoever caused it.

### The legitimate "403"

One fact removes a red herring before we start. The output plugin in this version has:

```c
    /* metric declared with two label keys */
    requests_total{status, name}

    /* and status is filled in from the HTTP response, literally: */
    snprintf(status_str, sizeof(status_str), "%d", http_status);
```

So `"403"` is a **completely normal label value**. The first time the endpoint returned 403, the library allocated a new `cmt_metric` for `requests_total{status="403", name="out.0"}` along with its two label nodes, one holding the string `"403"` and one holding `"out.0"`. Nothing about a `403` appearing in the heap is suspicious. It is the software working as designed.

### The corrected mechanism

Here is what actually happened, in order:

**Step 1. A label node was freed while still linked into its list.**

Some metric M's `cmt_map_label` node was passed to `free()` without first being unlinked from M's label list. M's head still pointed at it, and its `next` still pointed onwards into M's list. The free site is **not pinned down**; the two live candidates are the dangling-pointer bugs in the OAuth and metadata-token parsing path (a string buffer reallocated with the old pointer retained), and the map and concatenate lifecycle in the version of the metrics library in use, which has had a long run of `cat:` and `map:` fixes upstream since. That gap is stated rather than papered over.

**Step 2. Nothing happened for a long time.**

`free()` overwrote only the first 16 bytes of the chunk. The `_head` link sits at offset +8, and the `next` pointer within it at +8 from there, so at least part of the link survived and, critically, the *ring topology still worked*. Walks over M's label list completed normally. Zero symptoms. This is the `t1` to `t3` gap from section 6, and on the pods we examined it lasted weeks.

**Step 3. The first 403 arrived and the chunk was reallocated.**

The error burst created `requests_total{status="403", ...}`, which needed new `cmt_map_label` nodes. Same size class, LIFO free list, so the allocator handed back precisely that freed chunk. Now the chunk is a legitimate, correctly initialised label node of the 403 metric, linked into the 403 metric's own healthy list.

**Step 4. Two owners, one node.**

```
 The 403 metric's label list is PERFECTLY HEALTHY:

     B (head, embedded in the 403 cmt_metric)
     │
     ├──▶ C   name = "403"
     │
     └──▶ A   name = "out.0"        (the reused chunk)
              └──▶ back to B

     ring:  B -> C -> A -> B         contains its head. walks terminate. fine.

 The STUCK metric M's head was never updated:

     M.head (0x...247358)  ──next──▶  A

     walk from M.head:  A -> B -> C -> A -> B -> C -> ...
     head is B's ring, not M's. M.head is outside the ring it points into.
     `it != M.head` is true forever.
```

Both structures are internally consistent. Every byte is a valid value. Every pointer targets a real allocated object. There is nothing to see in a hex dump. And one of the two walks never ends.

**Step 5. One second later, the metrics timer fires.** `copy_label_values` reaches M, calls `cfl_list_size(&M->labels)`, and enters the seven-instruction loop. The engine thread never returns to `epoll_wait`. The node stops shipping logs. `restartCount` stays 0.

### Retraction: the "403 was written over the list" story was wrong

An earlier write-up of this investigation, including the first version of the companion note, said the error-handling path had **scribbled the string `403` over the list nodes**, corrupting them. That is retracted. It is the wrong mechanism.

Why it was believable. Dumping memory around the ring showed the string `403` and the string `logging.googleapis.com` sitting right there among the nodes. Both are strings the error path produces. It looks exactly like an error handler spraying response data over adjacent structures.

Why it is wrong, on two counts:

```
 1. The "403" is not debris. It is the VALUE of a real, expected label on a real
    metric: requests_total{status="403"}. The label key exists in the source.
    We mistook a feature for wreckage.

 2. The URL string nearby is HEAP NEIGHBOUR NOISE. The error path allocated
    several strings at around the same time, so they landed in nearby chunks in
    the same arena. Adjacency in a dump is not causation. It never is.
```

And the evidence that settled it was there all along, in what the dump did **not** show. A scribble leaves garbage: misaligned values, non-canonical addresses, half-overwritten pointers. Every pointer in that ring was well-formed, correctly aligned, and mutually consistent, forming a valid closed ring with valid `prev` links. **Well-formed structures that encode a wrong graph is the signature of reuse, not of overwriting.**

The generalisable lesson, which is worth more than the specific bug:

> When a hex dump around a corrupted structure contains strings from an error path, the boring explanation is that the error path allocated memory at the same time. Check whether the values are *legitimate content of a different object* before concluding they were written over yours. Read the source for the label keys before theorising about the bytes.

The correction also changes the fix category. If the mechanism were an error-path buffer overrun, the fix would be bounds checking in the error path. Because the mechanism is a use-after-free on a metrics structure, the fix is **lifetime ownership**: unlink before free, or do not free while linked, plus a version upgrade to pick up the upstream lifecycle fixes. Those are different patches in different files owned by different projects.

---

## 9. Debugging a container with no shell

### The concept

Modern container images are deliberately minimal. A **distroless** image contains the application binary, its shared libraries, CA certificates, and nothing else. No shell, no `ps`, no `top`, no `cat`. It is a genuinely good security posture and it means the first thing you try does not work:

```
 $ kubectl exec -it pod/log-agent-abcde -- sh
 error: Internal error occurred: exec: "sh": executable file not found in $PATH
```

The answer is **ephemeral containers**, GA since Kubernetes 1.25. You add a temporary container to a running pod, with an image *you* choose, and the pod is not restarted:

```
 kubectl debug -it pod/log-agent-abcde \
   --image=busybox \
   --target=log-agent
```

The `--target` flag is the load-bearing part and it is easy to miss:

```
 WITHOUT --target
   the debug container joins the pod's network and volume namespaces, but gets
   its OWN PID namespace. /proc/1 is YOUR sleep command. You cannot see the
   workload at all. This silently produces useless output.

 WITH --target=<container>
   the debug container SHARES the target container's PID namespace.
   /proc/1 is the workload. /proc/1/task/ is its threads.
   (Requires the runtime to support process namespace targeting; containerd and
    CRI-O do.)
```

That alone gets you everything in sections 1 through 4, because it is all just reading files.

### Why ptrace needed extra privileges

Reading registers is different from reading `/proc`. It needs `ptrace()`, and two independent mechanisms will refuse you.

**Capabilities.** Attaching to a process that is not your descendant requires `CAP_SYS_PTRACE`. A default container drops it. Separately, the default `RuntimeDefault` seccomp profile **blocks the `ptrace` syscall outright**, so even holding the capability you get `EPERM` from the filter rather than from the permission check.

`kubectl debug --profile=sysadmin` fixes both, by running the debug container privileged, which grants the full capability set and lifts seccomp confinement:

```
 kubectl debug -it pod/log-agent-abcde \
   --image=python:3.12-slim \
   --target=log-agent \
   --profile=sysadmin
```

**Yama.** Even with the capability, the kernel has a second gate, the Yama LSM:

```
 /proc/sys/kernel/yama/ptrace_scope
   0   classic: any process may ptrace another with the same uid
   1   restricted: only a DIRECT ANCESTOR may ptrace, unless CAP_SYS_PTRACE
       (this is the default on Ubuntu, Debian, and most container hosts)
   2   admin-only: CAP_SYS_PTRACE required, no exceptions
   3   no ptrace at all. AND IT CANNOT BE LOWERED without a reboot.
```

Levels 0 through 2 are all satisfied by `CAP_SYS_PTRACE`, which is why `--profile=sysadmin` is sufficient in practice. Level 3 is a hard stop, so if you meet it, plan on a core dump instead.

### What ptrace actually does

`ptrace()` is one syscall with a request code, and it is the foundation under `gdb`, `strace`, and every sampling profiler:

```
 PTRACE_ATTACH    stop the target thread and become its tracer.
                  ASYNCHRONOUS: you must waitpid() to know it has stopped.
 PTRACE_GETREGS   copy the target's register file into your buffer.
                  (or PTRACE_GETREGSET with NT_PRSTATUS, the portable form)
 PTRACE_PEEKDATA  read ONE WORD from the target's address space.
                  The word is the RETURN VALUE, so -1 is ambiguous with an
                  error and you must clear and check errno.
 PTRACE_DETACH    resume the target and stop tracing.
```

Three properties to know before you use it:

It is **per-thread**. You attach to a TID, not to the process. Attaching to the PID gets you only the main thread, which in our case was the idle one. Read `/proc/1/task/` first and pick your target.

It **stops the thread** while you read. That is the whole point, since section 4 showed a running thread has no readable instruction pointer, but it also means you are perturbing what you measure. Keep the stop short.

It is **exclusive**. One tracer per thread. If something else is attached, you get `EPERM`.

### The sampler

With `ATTACH`, `GETREGS`, `DETACH` in a loop you have a **sampling profiler**: repeatedly stop the thread, note where the instruction pointer is, let it go. If a thread spends all its time in one loop, every sample lands in that loop. No symbols and no debug info required to collect.

```python
import ctypes, os, sys, time

libc = ctypes.CDLL("libc.so.6", use_errno=True)
libc.ptrace.restype  = ctypes.c_long
libc.ptrace.argtypes = [ctypes.c_long, ctypes.c_long, ctypes.c_void_p, ctypes.c_void_p]

PTRACE_PEEKDATA, PTRACE_GETREGS = 2, 12
PTRACE_ATTACH,   PTRACE_DETACH  = 16, 17
__WALL = 0x40000000                      # needed to wait on a non-child thread

class Regs(ctypes.Structure):            # struct user_regs_struct, x86-64 order
    _fields_ = [(n, ctypes.c_ulonglong) for n in (
        "r15","r14","r13","r12","rbp","rbx","r11","r10","r9","r8","rax","rcx",
        "rdx","rsi","rdi","orig_rax","rip","cs","eflags","rsp","ss",
        "fs_base","gs_base","ds","es","fs","gs")]

def ptrace(req, tid, addr=0, data=None):
    ctypes.set_errno(0)
    r = libc.ptrace(req, tid, addr, data)
    err = ctypes.get_errno()
    if r == -1 and err != 0:
        raise OSError(err, os.strerror(err))
    return r

tid = int(sys.argv[1])                   # a TID from /proc/1/task, not the PID

for _ in range(60):
    ptrace(PTRACE_ATTACH, tid)
    os.waitpid(tid, __WALL)              # block until it is actually stopped
    regs = Regs()
    ptrace(PTRACE_GETREGS, tid, 0, ctypes.byref(regs))
    print(hex(regs.rip), hex(regs.rsp))
    ptrace(PTRACE_DETACH, tid)
    time.sleep(0.05)
```

### How it showed up: the 60-sample histogram

Sixty samples of the engine thread's `RIP`, at 50 ms intervals, after symbolisation (section 10):

```
 samples  location
 ───────  ─────────────────────────────────────────────
      50  cfl_list_size + 0x25        ; add $0x1,%eax     -> ret++
      10  cfl_list_size + 0x35        ; mov 0x8(%rdx),%rdx -> it = it->next
       0  anywhere else
 ───────
      60  total
```

Sixty out of sixty in one function, at two addresses 16 bytes apart. That is not a profile, it is a proof. A thread doing real work spreads across dozens of functions; a thread that never leaves an eight-instruction window is in a loop with no exit.

The 50/10 split is just where the CPU happens to be when interrupted, and it is consistent with the loop shape: the increment is a fast ALU op that retires immediately while the pointer chase is a dependent load, so the CPU spends more cycles stalled around the load. Do not over-read the ratio. The zero in the third row is the finding.

### Getting the call chain without a real unwinder

`RIP` says *where* the loop is. It does not say *who called it*, and `cfl_list_size` is called from many places. Without frame-pointer or DWARF unwinding, the cheap trick is a **stack scan**:

```
 read RSP from GETREGS
 walk upward from RSP one 8-byte word at a time, via PTRACE_PEEKDATA
 for each word, ask: does this value fall inside the executable's text mapping?
     (get the range from /proc/1/maps)
 if yes, it is PROBABLY a saved return address -> symbolise it
```

It is heuristic. It produces false positives, since any integer that happens to look like a code address is picked up, and it can miss frames when the compiler has omitted the frame pointer. But return addresses appear in the right order, so reading the plausible hits top to bottom gives you the call chain.

The chain it produced, and it is worth comparing against section 8:

```
 cfl_list_size            <- RIP is here, spinning
 copy_label_values
 copy_map
 copy_counter
 append_context
 cmt_cat
 flb_me_get_cmetrics
 collect_metrics
 flb_me_fd_event
 flb_engine_start         <- the engine event loop. confirms WHICH thread and
                             WHICH callback never returned.
```

That is the once-per-second metrics copy, stuck in the label list walk, on the engine thread. Everything in sections 5 through 8 follows from those ten lines.

---

## 10. Binaries: ELF, PIE, and turning an address into a symbol

### The concept

The sampler gives you numbers like `0x5cd56a8c1b25`. Turning that into `cfl_list_size + 0x25` requires knowing two things: what is in the binary, and where the binary got loaded.

**ELF** (Executable and Linkable Format) is the container format for Linux executables and shared libraries. Three parts matter:

```
 ELF header        magic, architecture, entry point, and where the tables are
 program headers   what the KERNEL LOADER reads. "map file offset X, length L,
                   at virtual address V, with permissions rwx". This is the
                   list that /proc/<pid>/maps ends up reflecting.
 section headers   what LINKERS AND TOOLS read. .text, .data, .symtab, .debug_*
                   Not needed at runtime, and often stripped.
 symbol table      name -> address+size, for every function and object
```

**PIE** (Position Independent Executable) is the part that trips people up. Historically an executable declared a fixed load address and always landed there. Modern toolchains build PIE by default so that **ASLR** can randomise the load address on every execution, which is a real exploit-mitigation win. The cost is that the addresses in the file are no longer runtime addresses:

```
 non-PIE   file address == runtime address.               symbolisation is trivial.
 PIE       runtime address = file address + LOAD BIAS.    you must find the bias.
```

### Finding the load bias

The bias is not printed anywhere. You compute it from `/proc/<pid>/maps`, which gives you, for each mapping, the runtime start address and the **file offset** it came from:

```
 $ grep ' r-xp ' /proc/1/maps
 5cd56a697000-5cd56ac41000 r-xp 000bf000 00:6d 1234  /fluent-bit/bin/fluent-bit
 └── runtime start ──┘      │    └ file offset
                            └ r-xp = readable + executable = this is .text
```

```
 load bias = runtime start of the text mapping  -  its file offset

           = 0x5cd56a697000 - 0xbf000
           = 0x5cd56a5d8000
```

Then, for any sampled address:

```
 file address = runtime RIP - load bias
```

Two mistakes to avoid. Use the `r-xp` mapping, since a binary has several mappings (`r--p` for read-only data, `rw-p` for writable) with different file offsets and only the executable one is relevant for instruction pointers. And do not assume the file offset is 0, because it usually is not for the text segment.

### From file address to symbol

`nm -n` prints the symbol table sorted by address. The symbol you want is the **greatest symbol whose address is less than or equal to** your file address, and the difference is the offset within it:

```
 $ nm -n /tmp/fluent-bit | grep -i ' t '
 ...
 00000000002e9b00 t cfl_list_size
 00000000002e9b60 t copy_label_values
 ...

 file address 0x2e9b25
   greatest symbol <= 0x2e9b25   ->   0x2e9b00  cfl_list_size
   offset                        ->   0x25
   result                        ->   cfl_list_size + 0x25
```

Then `objdump` shows you the instructions, so you can see what the loop actually is:

```
 objdump -d --start-address=0x2e9b00 --stop-address=0x2e9b50 /tmp/fluent-bit
```

If the binary is stripped, `nm` returns nothing and you need the matching debug symbols or an unstripped build of the same version. Which is a reason to keep an unstripped copy of anything you run in production.

### The cross-architecture wrinkle

The investigation was done from an arm64 laptop against a linux/amd64 container image. Docker will happily give you the wrong architecture's image by default, and then every address you compute is meaningless.

```bash
# pull the SAME architecture the cluster runs, not the laptop's
docker pull --platform linux/amd64 fluent/fluent-bit:2.1.10

# copy the binary out without running anything
cid=$(docker create --platform linux/amd64 fluent/fluent-bit:2.1.10)
docker cp "$cid:/fluent-bit/bin/fluent-bit" /tmp/fluent-bit
docker rm "$cid"

file /tmp/fluent-bit        # confirm: ELF 64-bit LSB pie executable, x86-64
```

`nm` and `objdump` from GNU binutils read foreign-architecture ELF files without complaint, because they parse the file, they do not execute it. You do not need an x86 machine to symbolise x86 addresses. You do need the exact same image digest, since a different build has different offsets.

---

## 11. What Kubernetes could and could not see

### The concept

Kubernetes restarts a container when the process **exits**, or when a **liveness probe fails**. That is the complete list. The kubelet's knowledge of a healthy container is "the PID I started is still present", which is true and useless for a hung process.

Exit status arithmetic, because it is the fastest classifier of a dead container:

```
 the shell convention: a process killed by signal N reports 128 + N

 exit 137 = 128 + 9  = SIGKILL   OOM kill, liveness kill, or an overrun SIGTERM
 exit 139 = 128 + 11 = SIGSEGV   invalid memory access. a memory-safety bug.
 exit 143 = 128 + 15 = SIGTERM   normal shutdown that missed the grace period
 exit 1/2                        the program chose to exit. config, credentials
```

Ownership follows the code. A 137 is usually yours: limits, buffer sizing. A 139 is usually upstream's, and your levers are a version pin or not running the faulting component.

### The asymmetry that made this incident possible

```
                        CRASH (139)                HANG
 ────────────────────   ────────────────────────   ──────────────────────────────
 process state          exits                     alive, running
 kubelet action         restarts the container     nothing
 restartCount           increments                 stays 0
 visibility             loud, counted, graphed     invisible
 self-healing           yes, seconds               never
 data loss              seconds of buffer          hours to MONTHS
```

Read that table and the perverse conclusion is unavoidable: **the crash is the good outcome.** The fleet took roughly 2,300 segfault restarts a day and each one cost seconds. The hangs cost up to three and a half months each.

Worse, the crashes were an **anchoring trap**. A restart count in the thousands looks like the problem has been found, so attention went to the loud, self-healing failure sitting next to the silent, total one. Both were the same defect from section 7, and the visible symptom was the harmless one.

### Liveness is not progress

```
 LIVENESS   "is the process there, and does it answer?"
            An HTTP probe on :2020 answers YES throughout the hang, because the
            HTTP server is a DIFFERENT THREAD with its own event loop (section 5).
            A TCP probe is worse: the listening socket is in the kernel and stays
            accept-able even if no userspace thread ever calls accept().

 PROGRESS   "has a monotonic counter moved in the last N minutes?"
            This is the only question that distinguishes working from wedged.
```

The general form, and it is worth memorising because it applies to every queue consumer, every controller, and every ingestion pipeline you will ever operate:

> A probe served by a different thread than the work cannot observe the work stopping. Liveness for a pipeline component must assert that a monotonic counter has advanced inside a bounded window.

### The detectors that worked, and the one that did not

```
 X  restartCount > 0                         0 on every hung pod. blind.
 X  HTTP liveness on the metrics port        200 for three months. blind.
 X  error-rate health endpoint               zero errors, because nothing ran
                                             and nothing failed. blind.
 X  memory limit / OOM kill                  hung pods sat at 10 to 200 MiB
                                             against a 512 MiB limit. never fired.
 ~  CPU flat at ~1 core, low variance        found 54 of 101. cheap, and it
                                             structurally misses the 0-CPU variant.
 OK per-node output rate == 0                found all 101. the right question.
```

The winning query is a per-node minimum, not a fleet sum. One node out of 852 going to zero moves the cluster total by 0.1%, which is invisible:

```promql
# any node whose agent has shipped nothing for 10 minutes
min by (node) (rate(agent_output_records_total[10m])) == 0
```

Two refinements the sweep forced:

Pair it with an **absence** rule. `== 0` does not fire on a missing series, so a dead exporter looks like silence rather than zero. `absent_over_time(...)` at the same severity covers that.

And qualify it with "and it should have had input". Thirteen GPU nodes reported zero output and were simply idle, correctly. Zero output is only a fault when there is something to ship, so join against pods scheduled on the node or compare input rate to output rate.

### The one piece of good news: durable position

The agent stores its per-file read offsets in a small database on a `hostPath` volume, outside the container's filesystem. So deleting the hung pod is a genuinely clean recovery:

```
 pod deleted  ->  DaemonSet recreates it  ->  new process reads the offset db
              ->  resumes each file from the byte where it stopped
```

The important caveat, and it is what makes detection latency the number that matters: it resumes from the offset, but the **files** may be gone. Node-local log history is bounded by the rotation policy, typically five files of 10 MiB per container, and a chatty container can churn all five in seconds. A pod hung for three months resumes into inodes that were unlinked in May. The offsets survive; the data does not.

---

## 12. Signals, crash handlers, and why the crash stack matches the hang stack

### The concept

A **signal** is the kernel's way of interrupting a process with a short, numbered message. Some are sent by users or other processes (`SIGTERM`, `SIGKILL`, `SIGHUP`), and some are raised by the CPU itself and translated by the kernel:

```
 SIGSEGV  11   invalid memory access. the MMU raised a page fault, the kernel
               looked up the address, found no valid mapping (or the wrong
               permissions), and there is nothing sensible to do but signal.
 SIGBUS    7   valid mapping, unusable access: misalignment, or reading past
               the end of a truncated mmap'd file.
 SIGFPE    8   integer divide by zero and friends.
 SIGILL    4   invalid instruction.
```

For each signal a process can install a **handler**. When the signal is delivered, the kernel pushes a frame onto the *faulting thread's* stack and jumps to the handler, so the handler runs with the faulting thread's stack still underneath it, which is exactly what makes a crash backtrace possible.

Default action for `SIGSEGV` is terminate plus core dump. `SIGKILL` and `SIGSTOP` cannot be caught, which is why `kill -9` always works and why a `D`-state task is unkillable (it is not that the signal is blocked, it is that the task never reaches a point where it can act on one).

### How a symbolized crash stack gets produced

The agent installs a `SIGSEGV` handler that prints a backtrace before dying. Mechanically:

```
 fault at the dereference
   -> kernel delivers SIGSEGV to the faulting thread
     -> handler runs on that thread's stack
       -> libbacktrace walks the stack using DWARF unwind tables from the ELF
          file (.eh_frame / .debug_frame), which is a REAL unwind, not the
          heuristic scan of section 9
         -> resolves each return address to file:line via debug info
           -> prints, then re-raises so the process dies with exit 139
```

Worth knowing the sharp edge: signal handlers must be **async-signal-safe**, and `malloc`, `printf` and most backtrace libraries are not. If the crash happened *inside* the allocator while holding its lock, the handler's own allocation deadlocks and the process hangs in the handler instead of dying. Which is one more way a crash can present as a hang, and a reason not to trust a crash handler as your only evidence.

### How it showed up: one defect, two stacks

The daily crash backtrace, thousands of times a day across the fleet:

```
 cfl_list_size            cfl_list.h        <- SIGSEGV here
   copy_label_values      cmt_cat.c
     copy_map
       copy_counter
         append_context
           cmt_cat
             flb_me_get_cmetrics
               collect_metrics
                 flb_me_fd_event
                   flb_engine_start
```

And the hang's call chain, from the ptrace stack scan in section 9:

```
 cfl_list_size            <- spinning here
   copy_label_values
     copy_map
       copy_counter
         append_context
           cmt_cat
             flb_me_get_cmetrics
               collect_metrics
                 flb_me_fd_event
                   flb_engine_start
```

**Identical.** Same function, same call chain, same thread, same one-second timer. Not two bugs that happen to be nearby. One bug, and the branch is decided entirely by what the freed chunk was reused for:

```
              a label node is freed while still linked
                            │
              the chunk is later reallocated as ...
                            │
        ┌───────────────────┴───────────────────┐
        ▼                                       ▼
  something that leaves a               a valid list node in a
  garbage value at offset +8            DIFFERENT healthy ring
        │                                       │
        ▼                                       ▼
  mov 0x8(%rax),%rax loads garbage        every load succeeds
  next dereference faults                 the walk never meets the head
        │                                       │
        ▼                                       ▼
     SIGSEGV                              INFINITE LOOP
  exit 139, restart, ~5 s of loss    3 months of total log loss on that node
  LOUD, COUNTED, SELF-HEALING        SILENT, UNCOUNTED, PERMANENT
```

The practical value of noticing this is that it collapses two separate investigations into one, and it tells you the crash rate is a **proxy for the hang risk**. A fleet doing thousands of segfaults a day in a list walk is a fleet that will accumulate hung pods, at some rate set by heap layout luck. The crashes were not a separate cosmetic annoyance to be triaged later. They were the same bug announcing itself, and the announcement was ignored because it was self-healing.

---

## 13. Trace it yourself

Everything below is generic. Substitute your own pod, container and binary. The order matters: each step either narrows the class or tells you the next tool.

### Step 0: get a shell into the PID namespace

```bash
POD=<pod-name>
NS=<namespace>
TARGET=<container-name>

# read-only investigation. busybox is enough for steps 1 to 3.
kubectl debug -it -n "$NS" "pod/$POD" --image=busybox --target="$TARGET" -- sh

# if you will need ptrace (step 4), start here instead
kubectl debug -it -n "$NS" "pod/$POD" \
  --image=python:3.12-slim --target="$TARGET" --profile=sysadmin -- bash
```

Sanity check before you trust anything: `cat /proc/1/comm` must print the workload, not `sleep` or `bash`. If it does not, `--target` did not take effect.

### Step 1: thread inventory, CPU split, and states

```sh
# run this twice, ~3 seconds apart, and diff the numbers
for t in /proc/1/task/*; do
  tid=${t##*/}
  comm=$(cat "$t/comm")
  vals=$(sed 's/.*) //' "$t/stat" | awk '{print $1, $12, $13}')   # state utime stime
  wchan=$(cat "$t/wchan" 2>/dev/null)
  printf '%-6s %-16s %-26s %s\n' "$tid" "$comm" "$vals" "$wchan"
done
```

Interpretation table:

```
 utime rising ~100 ticks/s, stime ~0     -> USERSPACE INFINITE LOOP. go to step 4.
 stime rising fast, many syscalls        -> syscall spin. go to step 3, use strace.
 both flat at 0, states all S/futex_wait -> DEADLOCK. find the lock cycle.
 both flat at 0, any state D             -> stuck in the kernel. look at the device.
 utime rising modestly, records moving   -> it is just working. you are done.
```

### Step 2: confirm it is not doing anything useful

```sh
cat /proc/1/task/<tid>/syscall     # "running" = in userspace, not in a syscall
cat /proc/1/task/<tid>/wchan       # empty = not sleeping in the kernel
cat /proc/1/task/<tid>/stat | sed 's/.*) //' | awk '{print $30}'   # kstkeip: expect 0
```

### Step 3: rule out the syscall-spin family

```bash
timeout 5 strace -f -p 1 -c        # syscall summary. an EMPTY table means
                                   # zero syscalls, which means userspace loop.
ss -tinp 2>/dev/null | head -30    # sockets: look for ESTABLISHED with an empty
                                   # rx queue and lastrcv measured in days
```

### Step 4: sample the instruction pointer

Use the Python sampler from section 9 against the busy TID. Sixty samples at 50 ms is plenty:

```bash
python3 /tmp/sample.py <tid> | sort | uniq -c | sort -rn
```

If every sample is inside one function, you have found the loop. If they spread across many, it is not a tight loop and you should widen the sampling window.

### Step 5: symbolise

```bash
# on your workstation, matching the cluster's architecture and image digest
docker pull --platform linux/amd64 <image>:<tag>
cid=$(docker create --platform linux/amd64 <image>:<tag>)
docker cp "$cid:/path/to/binary" /tmp/bin && docker rm "$cid"

# in the debug container: find the load bias
grep ' r-xp ' /proc/1/maps | head -1
#   bias = <runtime start> - <file offset>
#   file_addr = RIP - bias

# on your workstation: greatest symbol <= file_addr
nm -n /tmp/bin > /tmp/syms
awk -v a=$((0x2e9b25)) '{ v=strtonum("0x"$1); if (v<=a) { s=$3; sv=v } } \
                        END { printf "%s + 0x%x\n", s, a-sv }' /tmp/syms

# and read the code
objdump -d --start-address=0x2e9b00 --stop-address=0x2e9b60 /tmp/bin
```

### Step 6: walk the structure by hand

Once you know which structure the loop is walking, read it out with `PTRACE_PEEKDATA`. Offsets are version-specific, so confirm them against the headers for your version:

```python
def peek(tid, addr):
    return ptrace(PTRACE_PEEKDATA, tid, addr) & 0xFFFFFFFFFFFFFFFF

# struct cfl_list { prev; next; }   ->  next is at +8
# struct cmt_map_label { name; _head; }  ->  _head at +8, so from a link
#   pointer p, the owning label's name pointer is at p-8
HEAD = 0x...                       # from a register or a nearby pointer
it, seen = peek(tid, HEAD + 8), []
for _ in range(32):
    if it == HEAD:
        print("terminated normally, length", len(seen)); break
    if it in seen:
        print("RING closes at", hex(it), "WITHOUT the head. this is the hang."); break
    seen.append(it)
    print(hex(it), "name_ptr=", hex(peek(tid, it - 8)))
    it = peek(tid, it + 8)
else:
    print("no terminator in 32 hops")
```

Read the strings the name pointers target before theorising. If they are legitimate values of a *different* object, you are looking at reuse, not a scribble.

### Step 7: capture, then recover

```bash
# per-container restart counts and last termination reason
kubectl get pod -n "$NS" "$POD" -o jsonpath=\
'{range .status.containerStatuses[*]}{.name}{" restarts="}{.restartCount}\
{" reason="}{.lastState.terminated.reason}{" exit="}{.lastState.terminated.exitCode}{"\n"}{end}'

# the fault address, from the node
kubectl debug node/<node> -it --image=ubuntu --profile=sysadmin -- \
  bash -c 'dmesg -T | grep -Ei "segfault|general protection" | tail -20'

# the log lines BEFORE the crash, which is where the cause usually is
kubectl logs -n "$NS" "$POD" -c "$TARGET" --previous | tail -50

# only now: recover
kubectl delete pod -n "$NS" "$POD"
```

Do not delete first. A live hang is the only chance to learn the mechanism, and the restart destroys it. Leave one hung pod alone if you have several.

### The signatures, condensed

**Log signature of this specific failure**, reading the last lines before silence:

```
 [error] [output:...] unable to parse token body
 [error] [output:...] cannot retrieve oauth2 token
 { "error": { "code": 403, "status": "PERMISSION_DENIED", ... } }
 <!DOCTYPE html> ... 400 Bad Request ...          (a malformed request line)
 <!DOCTYPE html> ... 404 Not Found ...            (a corrupted request path)
 [ warn] [engine] failed to flush chunk ... retry in 7 seconds
 (nothing. ever again. and the retry never happens.)
```

The last line is the tell. A promise to retry in 7 seconds, followed by permanent silence, means the thread that owns the retry timer stopped.

**Metrics signature:**

```
 output records/s            0, flat, from one exact minute onward
 input records/s             0, flat, same minute
 CPU                         flat ~1000 m, near-zero variance
                             OR exactly 0 m (the second, unexamined variant)
 memory RSS                  flat, unchanging, far below the limit
 restartCount                flat at 0
 uptime metric               climbing normally
 error counters              0, and staying 0
 HTTP health endpoint        200
```

Two flat lines starting in the same minute, one at zero throughput and one at one core, with no restarts and no errors. Nothing else in this class of system produces that shape.

---

## Interview Prep

### Q: Explain how a stale pointer to freed memory causes a process to hang rather than crash.

**A:** It needs three ingredients: an intrusive circular linked list, a use-after-free, and lucky heap reuse.

Systems C uses circular doubly linked lists where the head is a sentinel embedded in the owning struct, the same shape as the kernel's `list_head`. Walking one is always `for (it = head->next; it != head; it = it->next)`. The only termination condition is arriving back at the head address. There is no NULL terminator, no length field, and no iteration cap, because a well-formed ring does not need any of those and adding them would be considered a bug.

Now free a node while it is still linked into the list. `free()` does not erase the chunk, it only marks it available and overwrites the first sixteen bytes with allocator bookkeeping. So the ring still works and there is no symptom at all, possibly for months.

Then something allocates the same size class. glibc's tcache and fastbins are LIFO, so the recently freed chunk gets handed back almost immediately. It becomes a legitimate node in a *different* list, correctly initialised, correctly linked into that list's ring. The original owner's head still points at it.

At that point the first list's head is pointing into a ring that does not contain it. Following `next` from the head goes round the second list's ring forever. Every dereference is a valid read of a valid, allocated object, so nothing faults, no error is recorded, and no log line is emitted. That is the hang.

The crash is the same defect with a different reuse outcome. If the chunk gets reused for something that leaves a garbage value where `next` used to be, the next dereference hits an unmapped address and you get SIGSEGV. Same function at the top of the stack, same call chain, different symptom. Which is why, in the real incident, the fleet was producing thousands of segfaults a day in exactly the function where the hung pods were spinning: one bug, and heap layout deciding whether each instance became a five-second restart or three months of silent total log loss.

### Q: Why does freed memory keep working, and why is that worse than it failing immediately?

**A:** Because a pointer is just an integer, and nothing invalidates it.

`free()` returns a chunk to the allocator's free list. It does not tell the kernel anything, does not unmap the page, and does not zero the bytes. glibc's tcache writes two words at the start of the chunk, a `next` link and a double-free `key`, so exactly sixteen bytes get clobbered and everything past offset sixteen is preserved verbatim. Wiping would cost cycles that a correct program does not need spent.

So reading through a dangling pointer succeeds. The page is still mapped, the address is still valid, the bytes are still there. You get the old value and no indication anything is wrong.

That is worse than an immediate failure for three reasons. The bug is silent at the moment the mistake is made, so the free and the symptom are separated by an unbounded amount of time, in our case weeks. The symptom appears only when an *unrelated* allocation of the same size class happens, which makes it load-dependent and non-reproducible, and it points the investigation at innocent code. And when the symptom finally arrives, the structure looks perfectly well-formed, so a hex dump shows nothing suspicious.

The last point is the one I would emphasise. "Memory corruption" makes people picture a buffer overflow spraying garbage. The more dangerous form is a violated invariant reached through a stale reference, where every byte is a valid value and every pointer targets a real object, and what is broken is the *graph*, not the bytes. Tools that look for bad writes will not find it. Only lifetime tracking will, which is what ASan's quarantine and MSan-style poisoning are for, and it is why the durable fix is ownership discipline (unlink before free) rather than validation.

### Q: You have a process pinned at 100% of one core. How do you tell an infinite loop from a deadlock from a livelock, and what evidence do you need?

**A:** The CPU and syscall profile separates all three before I read any code, and it is all available from `/proc`.

A deadlock is a *waiting* failure. Two or more threads each hold something the other needs, so nothing executes. Total CPU is approximately zero, and every involved thread is in state `S` with `wchan` reading `futex_wait`, since that is what userspace mutexes are built on. The process is frozen and completely quiet.

An infinite loop is a *computing* failure. One thread is executing a loop whose exit condition can never become true. State `R`, one full core of `utime`, near-zero `stime` because it is not asking the kernel for anything, and the instruction pointer confined to a few bytes of code.

A livelock is a *coordinating* failure. Threads are running and changing state, but the changes undo each other. It burns CPU *and* generates syscalls or visible state transitions, so it looks busy, which makes it the hardest of the three to name.

Concretely, the sequence I run: sample `/proc/<pid>/task/*/stat` twice a few seconds apart and diff fields 14 and 15, which are `utime` and `stime` in 100 Hz ticks. In our case one thread showed `+313 utime` and `+1 stime` over three seconds. That is 1.04 cores of pure userspace execution with 10 ms of kernel time, and it eliminated every syscall-spin hypothesis in one measurement, including the well-known `EAGAIN` retry spin in that software, which would have shown thousands of `read()` calls a second and a large `stime`.

Then `wchan` on every thread. The two output workers were `S` in `do_epoll_wait`, which is idle-waiting, not blocked-on-a-lock. That is what ruled out a deadlock: nobody was waiting for the spinning thread to release anything, they simply had no work because the thread that schedules work had stopped.

Then, since `/proc/<pid>/stat` field 30 (`kstkeip`) reads 0 for a running thread even as root, I used ptrace to sample the instruction pointer: `PTRACE_ATTACH`, `waitpid`, `PTRACE_GETREGS`, `PTRACE_DETACH` in a loop, sixty times at fifty milliseconds. All sixty samples landed in one function at two addresses sixteen bytes apart. Sixty out of sixty inside an eight-instruction window is not a profile, it is a proof.

The one caveat I would state unprompted: a hang can also present at *zero* CPU, and in the same fleet 23 of the 101 broken nodes did exactly that. That is a different bug with the deadlock signature, and I would not assume the mechanism I proved for the spinning ones applies to it.

### Q: The pod was Running with zero restarts and its HTTP health check returned 200 for three months. Why did every Kubernetes signal miss it?

**A:** Because Kubernetes only knows two things about a container: whether the process exited, and whether a liveness probe failed. A hung-but-alive process satisfies neither, and the kubelet's entire view is "the PID I started is still present", which is true and useless.

Then each individual signal failed for its own reason, and they are worth separating because they have different fixes.

The HTTP probe was answered by a *different thread*. The agent runs the engine on one thread and an embedded HTTP server on another, each with its own event loop. The probe asks the HTTP thread whether the HTTP thread is alive, and it always is. A TCP probe would be worse, because the listening socket lives in the kernel and stays accept-able even if no userspace thread ever calls `accept()`.

The purpose-built health endpoint measures the wrong predicate. It is defined as an error-rate check over a rolling window, and a hang produces zero errors because nothing runs and therefore nothing fails. It also fails *open* structurally, because the window is populated by a timer on the thread that stalled, so the newest sample freezes and the delta becomes zero forever.

The metrics were frozen rather than absent, which matters. Uptime kept climbing because it is computed at request time by the HTTP thread as `now - init_time`. The plugin counters froze because they are pushed by a one-second timer on the engine loop. So "uptime rising while record counters sit still" is not a coincidence, it is structural, and it is the cheapest remote fingerprint of this failure.

And resource guardrails never engaged. The hung pods sat between 10 and 200 MiB against a 512 MiB limit, so the OOM killer, which is the one mechanism that would have recovered them for free, was never close to firing. A hang is not a resource problem.

The fix is to stop asking about liveness and ask about progress: has a monotonic counter advanced inside a bounded window. In-container that is an exec probe comparing the current output-records total against the previous reading, with `failureThreshold` wide enough to cover the longest legitimate quiet period, and it is only safe if buffering is on disk so the restart does not discard in-flight data. Outside the container, and this is the control I would build first, a per-node minimum on shipped records: `min by (node) (rate(output_records_total[10m])) == 0`, paired with an `absent_over_time()` rule at the same severity, because `== 0` never fires on a missing series. In the real sweep the CPU-shape heuristic found 54 of 101 broken nodes and the output-rate query found all 101.

### Q: A crash backtrace ends in the metrics-collection code. Is the metrics code the bug?

**A:** Almost certainly not, and the reason is a general principle worth stating first: a periodic full-structure walk is a corruption *detector*. The function at the top of the stack is whoever tripped over the damage, not whoever caused it.

In this case the metrics exporter copies the entire metrics registry once a second, and copying a metric's label list starts by counting it, which walks the list. So it touches every metric structure in the process every second. It is therefore the first code to dereference any damage anywhere in that tree, no matter which component created it. It is the canary, not the gas leak.

The practical habit that follows is to read the log lines *before* the backtrace rather than the frames in it. Here those lines were token-parse and OAuth-refresh failures followed by HTTP error bodies, which pointed at the credential path, and separately there are known dangling-pointer bugs in the metadata token fetch where a string buffer is reallocated and the old pointer retained.

I would be careful about how strongly I state that, though, and this is where I would show the correction. An earlier version of my own analysis claimed the error-handling path had written response data over the list nodes, because a memory dump around the corrupted ring contained the string `403` and a Google API hostname, which looks exactly like an error handler spraying bytes over neighbouring structures.

That was wrong on two counts. The `403` is a legitimate label value: the plugin declares `requests_total{status, name}` and fills `status` from `snprintf` of the HTTP status code, so the first 403 response creates a real new metric with a real label node holding the string `403`. I had mistaken a feature for wreckage. And the hostname nearby was heap-neighbour noise, because the error path allocated several strings at about the same time and they landed in adjacent chunks. Adjacency in a dump is not causation.

What settled it was what the dump did *not* show. A scribble leaves misaligned values and non-canonical addresses. Every pointer in that ring was well-formed, correctly aligned, and mutually consistent, forming a valid closed ring with valid `prev` links. Well-formed structures encoding a wrong graph is the signature of *reuse*, not of overwriting. So the real mechanism is a use-after-free: a label node freed while still linked, then reallocated as a label node of the newly created 403 metric.

That distinction changes the fix category, which is why it was worth chasing. An error-path overrun would be fixed with bounds checking in the error path. A use-after-free on a metrics structure is fixed with lifetime ownership, unlink before free, plus picking up the upstream lifecycle fixes. Different patches, different files, different projects.

One structural recommendation I would add: if a process's job is shipping logs, do not also make it scrape its own metrics tree on a timer inside the same event loop. The crash reports in that codebase cluster in the metrics and events paths, none of which are on the log path, and running telemetry in a separate process means a bug there cannot take logging with it.

### Q: How do you debug a distroless container with no shell, and what permissions does that need?

**A:** Ephemeral containers, GA since Kubernetes 1.25. `kubectl debug -it pod/X --image=busybox --target=<container>` adds a temporary container to the running pod without restarting it, and I bring my own tools.

The flag that people miss is `--target`. Without it the debug container gets its own PID namespace and `/proc/1` is your own `sleep`, so you see nothing and the output looks plausible. With it you share the target's PID namespace, and since the workload is PID 1 in its own namespace, `/proc/1/task/` is exactly the thread list you want. I sanity-check with `cat /proc/1/comm` before trusting any measurement.

For everything in `/proc` that is enough, and it is a lot: thread inventory from `task/`, the CPU split from `stat` fields 14 and 15, states and `wchan`, memory mappings from `maps`. All of it is `cat` and `awk`, which busybox has. Worth knowing that `/proc` files are generated by kernel functions at read time, which is why they report size 0 and why sampling them is essentially free.

For registers it is different. `/proc/<pid>/stat` field 30 is documented as the instruction pointer and reads 0 for a running thread even with privileges, partly because it is gated behind `PTRACE_MODE_ATTACH_FSCREDS` and partly because a thread executing on another CPU has no reliable saved user register frame. So you have to stop the thread, and the only interface that stops a thread and hands you its registers is `ptrace`.

That needs `--profile=sysadmin`, and it is worth knowing it is fixing two independent blocks. `ptrace` on a process that is not your descendant requires `CAP_SYS_PTRACE`, which a default container drops, and the default `RuntimeDefault` seccomp profile blocks the `ptrace` syscall outright, so you would get `EPERM` from the filter even holding the capability. Running privileged lifts both. There is a third gate on top, the Yama LSM at `/proc/sys/kernel/yama/ptrace_scope`, where the common default of 1 restricts tracing to direct ancestors; levels 0 through 2 are all bypassed by `CAP_SYS_PTRACE`, but level 3 disables `ptrace` entirely and cannot be lowered without a reboot, so on such a host I would plan for a core dump instead.

Two operational notes from doing it for real. RBAC for ephemeral containers is a separate permission (`pods/ephemeralcontainers`) from `pods/exec`, and in the incident the production namespace granted neither, which is why the mechanism was only ever proved in a development cluster on a pod with the identical symptom. And symbolisation happens offline: the binary is PIE, so I read the load bias from the `r-xp` mapping in `/proc/1/maps` as `runtime start minus file offset`, subtract it from each sampled address, and look the result up with `nm -n` against the exact same image digest, pulled with `--platform linux/amd64` because my laptop is arm64. `nm` and `objdump` parse foreign-architecture ELF happily since they never execute it.

### Q: You have found the mechanism. What do you actually ship?

**A:** Four things, in priority order, and I would separate the ones that fix *this* bug from the ones that fix the *class*.

Immediate recovery is deleting the affected pods, which is clean here because the agent keeps its per-file read offsets in a database on a `hostPath` volume, so a new pod resumes from the byte where the old one stopped. The caveat that makes detection latency the number that matters is that the offsets survive but the files may not: node-local log history is bounded by rotation, typically five files of 10 MiB per container, and a pod hung for three months resumes into inodes unlinked in May. Nothing backfills that.

The real fix is a version upgrade, because the defect is upstream and the metrics library has had a long run of lifecycle fixes since the pinned version. I would not attempt to patch it locally: the free site is not pinned down, and shipping a fork of a log agent across a thousand nodes to work around a bug that upstream has already touched repeatedly is the wrong trade.

Detection is the part I would insist on regardless of the upgrade, because it converts every future instance of this class into a page instead of an archaeology project. A per-node output-rate-zero alert, joined against "this node has application pods scheduled", plus an absence rule at equal severity. That single alert would have caught the hangs, the zero-CPU variant, crash loops, config-parse failures and expired credentials without knowing which one happened.

Then the structural changes so the next failure is loud instead of silent. A progress-based liveness probe rather than an HTTP one, which is only safe with on-disk buffering. Telemetry collection out of the process whose job is shipping logs. And a monitor on the segfault rate, treated as a leading indicator rather than as cosmetic noise, since the crashes and the hangs are the same defect and the crash rate is a proxy for how fast hung pods will accumulate.

The meta-lesson I would offer is about which signal we trusted. The loud symptom, thousands of restarts a day, was the harmless one and it absorbed the attention. The silent symptom, zero restarts and a 200 from the health endpoint, was the total outage. Any time a self-healing failure sits next to a non-self-healing one, expect the investigation to anchor on the wrong one.

---

## Related Notes

- [[notes/K8s/node-log-pipeline-silent-failures|Alive But Not Shipping]] — the operational half of this story: the full log pipeline, why the blast radius was node-shaped, the probes and alerts that missed it, and the two unrelated failure modes it was tangled up with
- [[notes/Linux/linux-fundamentals-architecture-and-debugging|Linux Fundamentals, Architecture & Debugging]] — the wider map: boot, kernel subsystems, cgroups, namespaces, and where each debugging tool attaches
- [[notes/Networking/tcp-socket-internals|TCP Socket Internals]] — sockets, `epoll`, non-blocking I/O and `EAGAIN`, which is the syscall-spin sibling of this hang
- [[notes/K8s/interactive-containers-piping-and-ttys|Interactive Containers, Piping & TTYs]] — what `kubectl exec` and `kubectl debug` are doing underneath, and how namespaces are joined
- [[notes/K8s/when-gauge-sums-lie|When Gauge Sums Lie]] — the other way instrumentation misleads you, at query time rather than in the process
- [[notes/AuthNZ/oauth-oidc-and-workload-identity|OAuth, OIDC & Workload Identity Federation]] — the token-refresh path whose failures preceded the hang, and one of the two candidate free sites
