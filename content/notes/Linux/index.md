---
title: Linux
---

Notes on Linux internals, system administration, disk management, bootloaders, and debugging methodology.

## Fundamentals & Architecture
- [[notes/Linux/linux-fundamentals-architecture-and-debugging|Linux Fundamentals, Architecture & Debugging]] — Boot sequence, kernel subsystems, processes, memory, filesystems, networking, systemd, containers, and observability methodology

## Debugging & Postmortems
- [[notes/Linux/anatomy-of-a-userspace-hang|Anatomy of a Userspace Hang]] — Building up processes/threads, PID namespaces, user vs kernel mode, `/proc`, event loops and coroutines, the heap and what `free()` really does, intrusive circular linked lists, ptrace sampling and PIE symbolisation, then applying all of it to a real hang: a use-after-free on a metrics label node, reallocated during an error burst, leaving a once-per-second list walk going round a ring that no longer contained its own head
- [[notes/Linux/blas-worker-thread-spin-wait|Why a BLAS Worker Thread Doubled Our Inference CPU]] — An embedding change pushed `gemv` over OpenBLAS's single-thread threshold, the worker pool then `sched_yield`-spun ~100 ms after every call and never slept at 33–40 ms request gaps: spin vs futex sleep, how `sched_yield` and cgroups account the time, measuring a per-pod CPU floor with `/proc` and `pidstat`, `OPENBLAS_NUM_THREADS` vs `OPENBLAS_THREAD_TIMEOUT` and the env var overwritten in code, and how an `AverageValue` HPA target and p98 rightsizing amplified it

## System Administration
- [[notes/Linux/booting-linux-on-a-windows-pc|Booting Linux on a Windows PC]] — Cloning Ubuntu to a new disk layout, partition resizing, UEFI NVRAM boot entries, Secure Boot with shim/GRUB, and recovery via live USB
