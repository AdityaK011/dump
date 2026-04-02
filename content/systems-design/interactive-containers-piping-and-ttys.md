---
title: "Interactive Containers, Piping, and TTYs in Kubernetes"
date: 2026-04-02
tags:
  - kubernetes
  - linux
  - tty
  - containers
  - unix
  - networking
draft: false
---

What actually happens when you `kubectl exec`, `attach`, or try to pipe into a running container? Starting from "how do I make a CLI interactive in a cluster" and ending up at the Unix fundamentals that make it all possible.

## exec vs attach

Kubernetes gives you two ways to interact with a running container:

- **`kubectl exec`** spawns a *new* process inside the container. When you run `kubectl exec -it <pod> -- bash`, you get a fresh shell regardless of what the main process is doing. This is the equivalent of SSH-ing into a machine.

- **`kubectl attach`** connects your terminal to the *already running* main process (PID 1). What you see depends entirely on the entrypoint -- if it's an interactive CLI, you drop into it. If it's a web server, you just see logs scrolling by.

Think of `attach` as plugging a monitor and keyboard into a process that's already running. If that process doesn't expect interactive input, it's useless.

## What "Piping" Actually Means

Every Unix process starts with three file descriptors:

- `fd 0` → stdin (input)
- `fd 1` → stdout (output)
- `fd 2` → stderr (errors)

"Piping" means connecting one process's stdout to another's stdin. When you run `echo "hello" | grep "h"`, the shell wires echo's fd 1 to grep's fd 0. Data flows through like water in a pipe.

### Can You Pipe to a Running Process?

Here's the catch: **pipes are set up before a process starts.** Once a process is running, you can't magically connect to its stdin -- unless something is already listening on the other end.

This is why `kubectl attach` works: the container runtime held onto those file descriptor handles from the start. It kept the "pipe ends" open, waiting for someone to connect.

For local processes, your options include:

- **Named pipes (FIFOs):** `mkfifo /tmp/mypipe` creates a pipe as a file that two processes can use to communicate.
- **tmux/screen:** Creates a shared session that multiple terminals can attach to -- the local equivalent of `kubectl attach`.
- **reptyr:** Uses `ptrace` to literally steal a running process's file descriptors and reparent them to your terminal.

### Interactive CLIs and Piping

You *can* pipe to a new instance of most CLIs for one-shot queries (`echo "explain k8s" | gemini`). But piping to a *running* interactive CLI doesn't work well -- interactive tools detect when they're not on a TTY and often switch to non-interactive mode or break entirely.

## How kubectl attach Actually Works

The full chain looks like this:

```
your terminal → kubectl → API server → kubelet → containerd → process
```

Step by step:

1. You run `kubectl attach -it <pod>`.
2. kubectl opens a **WebSocket connection** to the Kubernetes API server.
3. The API server proxies that connection to the **kubelet** on the node where your pod runs.
4. The kubelet talks to the **container runtime** (containerd/CRI-O) and asks it to attach to the container's main process.
5. The container runtime connects to the **streams** -- the stdin/stdout/stderr file descriptors that were set up when the process first started.

kubectl itself is basically a dumb pipe -- it just shuttles bytes between your terminal and the API server. The real "session holding" is done by containerd on the node. It plays the same role tmux does locally: holding onto a pty so someone can connect later.

### kubectl vs tmux

Both solve the same problem (connect to a running process) but in different ways:

- **tmux** sits between the process and the terminal as a local middleman. It creates a pseudo-terminal, and the process talks to that. When you detach, tmux keeps the pty alive.
- **kubectl** doesn't hold anything itself. The container runtime (containerd) is the middleman that holds the pty. kubectl just relays bytes over the network through several hops.

## TTY: The Abstraction That Won't Die

TTY stands for **teletypewriter**. In the 1960s-70s, it was a physical machine -- a keyboard and printer connected to a computer via serial cable. Today, no physical teletypewriters exist, but the abstraction persists everywhere.

A TTY tells the system: "there's a human at the other end." When a process has a TTY attached, it gets special capabilities:

- Line editing (backspace, arrow keys)
- Signal handling (Ctrl+C → SIGINT, Ctrl+Z → SIGTSTP)
- Colors, cursor movement, screen clearing
- Interactive prompts

Without a TTY, the process assumes it's talking to another program and changes behavior accordingly -- skipping prompts, disabling colors, reading all input at once.

### Pseudo-Terminals (PTY)

The modern implementation is a **pseudo-terminal (pty)** -- a software-emulated TTY with two sides:

- **Master side:** held by the terminal application (or tmux, or containerd)
- **Slave side:** the process (bash, gemini, etc.) reads and writes here

The process has no idea whether it's talking to a real 1970s teletypewriter, a macOS Terminal window, or containerd proxied through three layers of Kubernetes networking. That's the beauty of the abstraction -- it's been working for 50+ years.

---

*Written from a conversation exploring how interactive processes work in Kubernetes -- starting from "how do I host an agent" and ending up at the Unix fundamentals that make it all possible.*
