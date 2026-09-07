---
title: Linux
---

Notes on Linux internals, system administration, disk management, bootloaders, and debugging methodology.

## Fundamentals & Architecture
- [[notes/Linux/linux-fundamentals-architecture-and-debugging|Linux Fundamentals, Architecture & Debugging]] — Boot sequence, kernel subsystems, processes, memory, filesystems, networking, systemd, containers, and observability methodology

## Debugging & Postmortems
- [[notes/Linux/anatomy-of-a-userspace-hang|Anatomy of a Userspace Hang]] — Building up processes/threads, PID namespaces, user vs kernel mode, `/proc`, event loops and coroutines, the heap and what `free()` really does, intrusive circular linked lists, ptrace sampling and PIE symbolisation, then applying all of it to a real hang: a use-after-free on a metrics label node, reallocated during an error burst, leaving a once-per-second list walk going round a ring that no longer contained its own head

## System Administration
- [[notes/Linux/booting-linux-on-a-windows-pc|Booting Linux on a Windows PC]] — Cloning Ubuntu to a new disk layout, partition resizing, UEFI NVRAM boot entries, Secure Boot with shim/GRUB, and recovery via live USB
