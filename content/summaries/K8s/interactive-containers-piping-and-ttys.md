---
title: "Summary: Interactive Containers, Piping, and TTYs in Kubernetes"
---

> **Full notes:** [[notes/K8s/interactive-containers-piping-and-ttys|Interactive Containers, Piping, and TTYs in Kubernetes -->]]

## Key Concepts

### exec vs attach
- **`kubectl exec`**: spawns a **new** process inside the container (like SSH)
- **`kubectl attach`**: connects to the **already running** main process (PID 1)
- attach is useless if the main process does not expect interactive input

### Unix File Descriptors and Piping
- Every process starts with: fd 0 (stdin), fd 1 (stdout), fd 2 (stderr)
- Piping = connecting one process's stdout to another's stdin
- **Pipes are set up before a process starts** -- cannot retroactively connect to a running process's stdin
- `kubectl attach` works because the container runtime held the fd handles open from the start

### Connecting to Running Processes (Workarounds)
- **Named pipes (FIFOs)**: `mkfifo /tmp/mypipe` -- file-based IPC
- **tmux/screen**: shared session, local equivalent of kubectl attach
- **reptyr**: uses `ptrace` to steal a running process's fds and reparent to your terminal

### kubectl attach Chain
- `terminal -> kubectl -> API server (WebSocket) -> kubelet -> containerd -> process`
- kubectl is a dumb byte pipe; containerd does the real session holding (like tmux)

### TTY (Teletypewriter)
- Tells the system "there's a human at the other end"
- With TTY: line editing, signal handling (Ctrl+C = SIGINT, Ctrl+Z = SIGTSTP), colors, prompts
- Without TTY: process assumes it talks to another program, disables interactive features

### Pseudo-Terminals (PTY)
- Software-emulated TTY with two sides:
  - **Master**: held by terminal app (or tmux, or containerd)
  - **Slave**: the process reads/writes here
- Process cannot tell if it is talking to a 1970s teletypewriter or containerd through 3 layers of K8s networking

## Quick Reference

```
kubectl exec vs attach:
  exec   -> NEW process in container   (like SSH)
  attach -> EXISTING PID 1 process     (like plugging in a monitor)

The attach chain:
  terminal -> kubectl -> API server -> kubelet -> containerd -> process
             (bytes)    (WebSocket)   (CRI)      (holds pty)

Key signals via TTY:
  Ctrl+C -> SIGINT   (interrupt)
  Ctrl+Z -> SIGTSTP  (suspend)
  Ctrl+D -> EOF      (end of input)

PTY architecture:
  Master side: terminal emulator / tmux / containerd
  Slave side:  bash / application process
  (Process sees no difference between real TTY and PTY)

Piping to running processes:
  Regular pipe  -> impossible (set up at process start)
  Named pipe    -> mkfifo, file-based, two processes agree on path
  tmux/screen   -> shared session, attach/detach
  reptyr        -> ptrace-based fd theft (hack)
```
