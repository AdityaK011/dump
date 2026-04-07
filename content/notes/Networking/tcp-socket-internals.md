---
title: "TCP Socket Internals: Listening vs Connected Sockets, Socket Buffers, and the Kernel Network Stack"
---

## Overview

A **socket** is the application's interface to the kernel network stack. It is a file descriptor -- an integer that the process uses to read and write bytes. Everything below that file descriptor -- TCP segmentation, sequence numbers, IP headers, route lookups, netfilter processing, NIC driver interaction -- is handled entirely by the kernel. The application never sees TCP segments or IP packets; it sees a byte stream.

This note covers: how the kernel manages listening vs connected sockets, how socket buffers implement flow control, the full path a packet takes from `write()` to the wire (and back), and how netfilter operates as a completely independent subsystem from sockets -- which is critical for understanding why kube-proxy's iptables DNAT is transparent to applications.

For the kernel networking primitives that underpin container networking (namespaces, veth pairs, bridges, iptables chains), see [[notes/Networking/container-networking-internals|Container Networking Internals]]. For TCP keepalive mechanics and middlebox state tables, see [[notes/Networking/TCP_keepalives|TCP Keepalives]]. For Unix domain sockets (which bypass the network stack entirely), see [[notes/Networking/Unix_socket|Unix Domain Sockets]].

---

## Sockets as Kernel Abstractions

A socket is created by the `socket()` syscall and returns a **file descriptor** (fd) -- an integer index into the process's file descriptor table. From the process's perspective, a socket fd behaves like a file: you `read()` from it and `write()` to it. But internally, the kernel associates this fd with a `struct socket` (and the underlying `struct sock`) that contains all TCP/IP state: connection state machine, sequence numbers, timers, buffer pointers, congestion window, and more.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Userspace Process                                                   │
│                                                                      │
│  fd = 3  ──────►  read(fd, buf, n)   // "give me bytes"            │
│                    write(fd, buf, n)  // "send these bytes"          │
│                    close(fd)          // "tear down"                  │
│                                                                      │
│  The process sees a byte stream.                                     │
│  It has NO knowledge of:                                             │
│    - TCP segments, sequence numbers, ACKs                            │
│    - IP headers, TTL, routing decisions                              │
│    - Netfilter/iptables rules rewriting packets                      │
│    - NIC ring buffers, DMA, interrupts                               │
├──────────────────────────────────────────────────────────────────────┤
│  Kernel                                                              │
│                                                                      │
│  fd 3 ──► struct file ──► struct socket ──► struct sock (TCP)        │
│                                                                      │
│  struct sock contains:                                               │
│    - TCP state (LISTEN, SYN_RECV, ESTABLISHED, FIN_WAIT, ...)        │
│    - Send buffer (sk_write_queue)                                    │
│    - Receive buffer (sk_receive_queue)                               │
│    - Sequence numbers (snd_nxt, rcv_nxt)                             │
│    - Congestion window (cwnd, ssthresh)                              │
│    - Retransmission timers                                           │
│    - Socket options (SO_REUSEADDR, TCP_NODELAY, ...)                 │
│    - Route cache entry                                               │
└──────────────────────────────────────────────────────────────────────┘
```

### The Syscall Boundary

Every socket operation crosses the **userspace-kernel boundary** via a syscall. This is a context switch: the CPU transitions from user mode (ring 3) to kernel mode (ring 0), the kernel validates arguments, performs the operation, and returns a result. Key syscalls:

| Syscall | Purpose |
|---|---|
| `socket()` | Create a new socket fd (specify domain, type, protocol) |
| `bind()` | Assign a local address (IP + port) to the socket |
| `listen()` | Convert socket to a passive/listening socket, create accept queue |
| `accept()` | Dequeue a completed connection, return a new connected socket fd |
| `connect()` | Initiate a 3-way handshake to a remote address |
| `read()` / `recv()` | Copy bytes from kernel receive buffer to userspace buffer |
| `write()` / `send()` | Copy bytes from userspace buffer to kernel send buffer |
| `close()` | Initiate connection teardown (FIN), release fd |
| `setsockopt()` | Configure socket options (buffer sizes, TCP_NODELAY, SO_REUSEADDR) |

---

## Listening Socket vs Connected Socket

This is the most important distinction in server-side socket programming. A **listening socket** and a **connected socket** are fundamentally different kernel objects, even though both are represented as file descriptors.

### The Listening Socket (LISTEN State)

When a server calls `listen(fd, backlog)`, the kernel transforms the socket into a passive listener. This socket:

- Is in the **LISTEN** TCP state
- Is bound to a local address (e.g., `0.0.0.0:8080`)
- **Never carries application data** -- it only handles incoming SYN packets
- Maintains two internal queues (the "backlog"):
  - **SYN queue** (half-open connections): connections where SYN has been received and SYN-ACK sent, but final ACK not yet received
  - **Accept queue** (completed connections): connections where the 3-way handshake is fully complete, waiting for `accept()`
- Stays alive for the entire lifetime of the server -- one listening socket can produce thousands of connected sockets

### The 3-Way Handshake: Entirely Kernel-Managed

The application does **not** participate in the TCP handshake. The kernel handles all of it:

```
 Client                          Kernel (on server)                  App (server)
   |                                   |                                  |
   |---- SYN seq=x ------------------>|                                  |
   |                                   |  (create entry in SYN queue)     |
   |                                   |                                  |
   |<--- SYN-ACK seq=y ack=x+1 ------|                                  |
   |                                   |                                  |
   |---- ACK seq=x+1 ack=y+1 ------->|                                  |
   |                                   |  (move from SYN queue            |
   |                                   |   to accept queue)               |
   |                                   |                                  |
   |                                   |  ............(waiting)...........|
   |                                   |                                  |
   |                                   |<--- accept() -------------------|
   |                                   |                                  |
   |                                   |---- return new fd=5 ----------->|
   |                                   |     (ESTABLISHED socket)         |
   |                                   |                                  |
   |<========= data exchange over fd=5 (connected socket) =============>|
```

The connection is **fully established** (3-way handshake complete, ESTABLISHED state) **before** the application even calls `accept()`. The kernel is doing all the work. If the application is slow to call `accept()`, completed connections pile up in the accept queue. If the accept queue overflows, the kernel drops incoming connections (the behavior depends on `tcp_abort_on_overflow` sysctl -- by default, it silently drops the final ACK, causing the client to retransmit).

### The Connected Socket (ESTABLISHED State)

When the application calls `accept()`, the kernel:

1. Pulls the next completed connection from the accept queue
2. Creates a **new** `struct sock` (with its own send and receive buffers)
3. Allocates a **new file descriptor** for it
4. Returns this new fd to the application

This connected socket:

- Is in the **ESTABLISHED** TCP state
- Is identified by its unique **4-tuple**: `(local IP, local port, remote IP, remote port)`
- Has its own send and receive buffers
- Is completely independent of the listening socket

### Why One Port Can Handle Thousands of Connections

A common misconception: "each connection needs its own port." This is wrong. The **listening socket** occupies one port (e.g., 8080). Every connected socket shares the **same local port** (8080) but has a different remote IP:port combination. The kernel demultiplexes incoming packets by the full 4-tuple, not just the destination port:

```
Listening socket:  *:8080  (state: LISTEN)
                     |
            ┌────────┼─────────┬──────────────┐
            |        |         |              |
Connected:  |        |         |              |
  (10.0.0.1:8080,   (10.0.0.1:8080,   (10.0.0.1:8080,   ...
   10.0.0.5:42301)   10.0.0.6:51992)   10.0.0.5:42302)

  fd=5               fd=6               fd=7
  ESTABLISHED        ESTABLISHED        ESTABLISHED

Each has its own:
  - Send buffer
  - Receive buffer
  - TCP sequence numbers
  - Congestion window
  - Retransmission timers
```

The theoretical maximum is the number of unique 4-tuples. With one server IP and one port, the limit is the number of unique (remote IP, remote port) pairs -- approximately 2^48 (~281 trillion). In practice, the limits are file descriptors (`ulimit -n`), memory (each socket consumes ~3-10 KB of kernel memory), and the conntrack table size.

### Go Example: Listening and Accepting

```go
package main

import (
    "fmt"
    "net"
    "os"
)

func main() {
    // net.Listen = socket() + bind() + listen()
    // Returns a net.Listener wrapping the listening socket fd.
    ln, err := net.Listen("tcp", ":8080")
    if err != nil {
        fmt.Fprintf(os.Stderr, "listen error: %v\n", err)
        os.Exit(1)
    }
    defer ln.Close()
    fmt.Println("Listening on :8080")

    for {
        // ln.Accept() = accept() syscall
        // Blocks until a completed connection is available in the accept queue.
        // Returns a net.Conn wrapping a NEW connected socket fd.
        conn, err := ln.Accept()
        if err != nil {
            fmt.Fprintf(os.Stderr, "accept error: %v\n", err)
            continue
        }

        // Each conn is a separate fd with its own buffers.
        // Handle in a goroutine so we can accept more connections.
        go handleConnection(conn)
    }
}

func handleConnection(conn net.Conn) {
    defer conn.Close() // close() syscall -- sends FIN, releases fd

    // conn.RemoteAddr() reveals the client's IP:port (the other half of the 4-tuple)
    fmt.Printf("New connection from %s\n", conn.RemoteAddr())

    buf := make([]byte, 4096)
    for {
        // conn.Read() = read()/recv() syscall
        // Copies bytes from the kernel receive buffer to userspace buf.
        // Blocks if receive buffer is empty (or returns io.EOF on FIN).
        n, err := conn.Read(buf)
        if err != nil {
            return // EOF or error -- connection closed
        }

        // conn.Write() = write()/send() syscall
        // Copies bytes from userspace buf to the kernel send buffer.
        // Blocks if send buffer is full (in blocking mode).
        _, err = conn.Write(buf[:n]) // echo back
        if err != nil {
            return
        }
    }
}
```

### Go Client Side

```go
package main

import (
    "fmt"
    "net"
    "os"
)

func main() {
    // net.Dial = socket() + connect()
    // connect() triggers the 3-way handshake.
    // Dial blocks until the handshake completes (or times out).
    // Returns a net.Conn wrapping the connected socket fd.
    conn, err := net.Dial("tcp", "10.0.0.1:8080")
    if err != nil {
        fmt.Fprintf(os.Stderr, "dial error: %v\n", err)
        os.Exit(1)
    }
    defer conn.Close()

    // The kernel assigned an ephemeral port (e.g., 42301)
    fmt.Printf("Local address:  %s\n", conn.LocalAddr())  // e.g., 10.0.0.5:42301
    fmt.Printf("Remote address: %s\n", conn.RemoteAddr()) // 10.0.0.1:8080

    conn.Write([]byte("hello"))

    buf := make([]byte, 4096)
    n, _ := conn.Read(buf)
    fmt.Printf("Received: %s\n", string(buf[:n]))
}
```

---

## Socket Buffers (Send and Receive)

Every connected TCP socket has two kernel-space buffers. These buffers decouple the application's read/write pace from the network's transmission pace. They are central to TCP's flow control and congestion control mechanisms.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                            Connected Socket (fd=5)                            │
│                                                                              │
│  ┌─ Send Buffer (sk_write_queue) ──────────────────────────────────────┐     │
│  │                                                                      │     │
│  │   App writes here ──►  [bytes waiting to be sent as TCP segments]    │     │
│  │                                                                      │     │
│  │   Kernel drains this buffer:                                         │     │
│  │     - Segments data according to MSS                                 │     │
│  │     - Applies Nagle's algorithm (unless TCP_NODELAY)                 │     │
│  │     - Sends when congestion window allows                            │     │
│  │     - Keeps copy until ACKed (for retransmission)                    │     │
│  │                                                                      │     │
│  │   If FULL:                                                           │     │
│  │     - Blocking mode: write() blocks until space available            │     │
│  │     - Non-blocking mode: write() returns EAGAIN/EWOULDBLOCK          │     │
│  │     - epoll: fd becomes NOT writable (no EPOLLOUT event)             │     │
│  └──────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  ┌─ Receive Buffer (sk_receive_queue) ─────────────────────────────────┐     │
│  │                                                                      │     │
│  │   Kernel fills this ──►  [bytes received from network, in-order]     │     │
│  │                                                                      │     │
│  │   App reads from here via read()/recv()                              │     │
│  │                                                                      │     │
│  │   If FULL:                                                           │     │
│  │     - Kernel advertises TCP window = 0 to sender                     │     │
│  │     - Sender STOPS sending (TCP flow control)                        │     │
│  │     - Sender periodically probes with "window probe" segments        │     │
│  │     - When app reads and frees space, kernel advertises new window   │     │
│  │                                                                      │     │
│  │   If EMPTY:                                                          │     │
│  │     - Blocking mode: read() blocks until data arrives                │     │
│  │     - Non-blocking mode: read() returns EAGAIN/EWOULDBLOCK           │     │
│  │     - epoll: fd is NOT readable (no EPOLLIN event)                   │     │
│  └──────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Buffer Sizes and Tuning

Default buffer sizes on Linux are controlled by sysctl:

```bash
# Receive buffer: min, default, max (in bytes)
cat /proc/sys/net/ipv4/tcp_rmem
#  4096   131072   6291456
#  (4KB)  (128KB)  (6MB)

# Send buffer: min, default, max (in bytes)
cat /proc/sys/net/ipv4/tcp_wmem
#  4096   16384   4194304
#  (4KB)  (16KB)  (4MB)

# Per-socket override (in application code):
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &size, sizeof(size));
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &size, sizeof(size));
```

With **tcp_moderate_rcvbuf** enabled (default), the kernel auto-tunes the receive buffer between the min and max values based on available memory and traffic patterns. The application can override this with `SO_RCVBUF`, but the kernel caps it at 2x the requested value (unless `SO_RCVBUFFORCE` with `CAP_NET_ADMIN`).

### Receive Buffer = TCP Receive Window

This is a critical relationship. The TCP **receive window** (advertised in every ACK segment's `Window` field) is derived directly from the free space in the receive buffer:

```
┌──────────────────────────────────────────────────────────────────┐
│  Receive Buffer (131072 bytes = 128 KB)                          │
│                                                                  │
│  ┌─────────────────────┬─────────────────────────────────────┐  │
│  │   Data waiting to    │         Free space                  │  │
│  │   be read by app     │      (= TCP receive window)         │  │
│  │   (50 KB consumed)   │         (78 KB free)                │  │
│  └─────────────────────┴─────────────────────────────────────┘  │
│                                                                  │
│  ACK sent to peer:  Window = 78 KB                               │
│  (actually uses Window Scale, so: Window = 78KB >> wscale)       │
│                                                                  │
│  If app stops reading and buffer fills to 128 KB:                │
│    ACK sent to peer:  Window = 0                                 │
│    Peer MUST stop sending (TCP flow control)                     │
│    Peer sends periodic "window probes" (~every 30-60s)           │
│    When app reads and frees space → new ACK with Window > 0      │
└──────────────────────────────────────────────────────────────────┘
```

This is the **flow control** mechanism of TCP (RFC 9293, Section 3.8.6). The receive buffer size directly controls how much unread data the sender is allowed to push. A slow application that doesn't read fast enough will eventually cause the sender to pause. This is by design -- it prevents the sender from overwhelming the receiver.

> **Note:** Flow control (receive window) is different from **congestion control** (cwnd). Flow control protects the receiver. Congestion control protects the network. Both can independently limit the sending rate. The effective send window is `min(rwnd, cwnd)`.

### Practical Implications

**High-latency, high-bandwidth links (long fat networks):** The default 128 KB receive buffer limits throughput. With a 100ms RTT, the maximum throughput is `buffer_size / RTT = 128KB / 0.1s = 1.28 MB/s` (~10 Mbps). For a 10 Gbps link, you need receive buffers of at least `10 Gbps * 0.1s = 125 MB`. This is the **bandwidth-delay product** (BDP).

```bash
# Increase max buffer for high-BDP links:
sysctl -w net.ipv4.tcp_rmem="4096 131072 134217728"   # max 128 MB
sysctl -w net.ipv4.tcp_wmem="4096 16384 134217728"    # max 128 MB
```

**Slow consumers:** If an application reads slowly from a socket, the receive buffer fills, the window shrinks to zero, and the sender pauses. This is often observed in scenarios where an HTTP server streams data to a client with a slow downstream link -- the server's `write()` eventually blocks when the send buffer fills because the remote receive window has shrunk.

---

## Kernel Network Stack: Packet Flow

### Sending: `write()` to Wire

When an application calls `write(fd, data, len)`, here is the full path through the kernel:

```
┌──────────────────────────────────────────────────────────────────────────┐
│  APPLICATION (userspace)                                                  │
│    write(fd, "Hello", 5)                                                 │
│         │                                                                │
│         │  syscall (context switch: user → kernel mode)                  │
│         ▼                                                                │
├──────────────────────────────────────────────────────────────────────────┤
│  SOCKET LAYER                                                            │
│    - Validate fd, check socket state (must be ESTABLISHED)               │
│    - Copy data from userspace buffer to kernel send buffer               │
│    - write() returns to userspace (data is now kernel's responsibility)  │
│         │                                                                │
│         ▼                                                                │
├──────────────────────────────────────────────────────────────────────────┤
│  TCP LAYER (L4)                                                          │
│    - Segment data according to MSS (typically 1460 bytes for Ethernet)   │
│    - Apply Nagle's algorithm (coalesce small writes, unless TCP_NODELAY) │
│    - Assign sequence numbers                                             │
│    - Calculate checksum (over pseudo-header + TCP header + payload)       │
│    - Set flags (PSH, ACK, etc.)                                          │
│    - Place copy in retransmit queue (until ACKed)                        │
│    - Respect congestion window (cwnd) and receive window (rwnd)          │
│         │                                                                │
│         │  TCP Header (20 bytes minimum):                                │
│         │  ┌────────────┬────────────┬────────────┬──────────────┐      │
│         │  │ Src Port   │ Dst Port   │  Seq Num   │  Ack Num     │      │
│         │  │  (16 bit)  │  (16 bit)  │  (32 bit)  │  (32 bit)    │      │
│         │  ├────────────┴────────────┴────────────┴──────────────┤      │
│         │  │ Flags │ Window │ Checksum │ Urgent Ptr │ Options    │      │
│         │  └─────────────────────────────────────────────────────┘      │
│         ▼                                                                │
├──────────────────────────────────────────────────────────────────────────┤
│  IP LAYER (L3)                                                           │
│    - Add IP header (20 bytes): src IP, dst IP, TTL, protocol=6 (TCP)     │
│    - Route lookup (FIB table): determine output interface and next hop    │
│    - Set TTL (default 64 on Linux)                                       │
│    - Fragment if needed (rare with Path MTU Discovery)                   │
│         │                                                                │
│         ▼                                                                │
├──────────────────────────────────────────────────────────────────────────┤
│  NETFILTER (iptables/nftables)                                           │
│    - OUTPUT chain (locally generated packets)                            │
│    - POSTROUTING chain (after routing decision)                          │
│    - DNAT/SNAT rules applied here (e.g., kube-proxy ClusterIP rewrite)  │
│    - Conntrack creates/updates connection tracking entry                  │
│    - Packet may be modified (dst IP rewritten), dropped, or marked       │
│         │                                                                │
│         ▼                                                                │
├──────────────────────────────────────────────────────────────────────────┤
│  NETWORK DEVICE / NIC DRIVER (L2)                                        │
│    - ARP lookup for next-hop MAC address (or use cached entry)           │
│    - Add Ethernet frame header (14 bytes): src MAC, dst MAC, EtherType   │
│    - Enqueue frame in NIC's transmit ring buffer (TX ring)               │
│    - NIC DMA's frame from ring buffer to the wire                        │
│    - Hardware interrupt when transmission complete                        │
│         │                                                                │
│         ▼                                                                │
├──────────────────────────────────────────────────────────────────────────┤
│  WIRE                                                                    │
│    Physical/electrical signal on the medium                              │
└──────────────────────────────────────────────────────────────────────────┘
```

### Receiving: Wire to `read()`

The reverse path for an incoming packet:

```
  WIRE
    │
    ▼
  NIC receives frame → DMA to kernel memory → hardware interrupt → softirq
    │
    ▼
  NETWORK DEVICE LAYER (L2)
    - Validate Ethernet frame (FCS check)
    - Strip Ethernet header
    - Identify protocol via EtherType (0x0800 = IPv4)
    │
    ▼
  NETFILTER: PREROUTING chain
    - DNAT rules applied (e.g., kube-proxy rewrites dst IP)
    - Conntrack matches existing connection entry
    │
    ▼
  IP LAYER (L3)
    - Validate IP header (checksum, TTL)
    - Routing decision: is this packet for us (INPUT) or forwarded (FORWARD)?
    - For local delivery → proceed to INPUT chain
    │
    ▼
  NETFILTER: INPUT chain
    - Firewall rules (accept/drop)
    │
    ▼
  TCP LAYER (L4)
    - Demultiplex by 4-tuple → find the matching struct sock
    - Validate TCP checksum
    - Process TCP state machine (handle SYN, ACK, FIN, RST, etc.)
    - For data segments: place payload in socket receive buffer (in sequence order)
    - Send ACK back to sender
    - If receive buffer full → advertise window = 0
    │
    ▼
  SOCKET LAYER
    - Wake up any process blocked in read()/recv()/epoll_wait()
    │
    ▼
  APPLICATION
    - read(fd, buf, n) copies data from kernel receive buffer to userspace
```

### The Complete Round-Trip Picture

```
┌─────────┐     write()      ┌──────────┐     TCP/IP      ┌─────────┐
│  App A  │ ──────────────►  │ Send Buf │ ──────────────►  │         │
│ (client)│                  │ (kernel)  │    segments      │  NIC    │──► Wire
│         │     read()       │          │                  │         │
│         │ ◄────────────── │ Recv Buf │ ◄──────────────  │         │◄── Wire
└─────────┘                  └──────────┘                  └─────────┘
                                  ▲
                                  │
                             Netfilter hooks
                             (OUTPUT, POSTROUTING,
                              PREROUTING, INPUT)
                             operate on packets
                             in transit -- invisible
                             to both the socket
                             layer and the application
```

---

## Netfilter vs Sockets: Independent Kernel Subsystems

Sockets and netfilter are **completely independent** kernel subsystems that operate at different layers of the kernel's network processing pipeline. Understanding this separation is essential for reasoning about Kubernetes networking, especially kube-proxy's iptables-mode Service implementation.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Kernel Network Stack                                                    │
│                                                                          │
│  ┌─────────────────────┐       ┌──────────────────────────────────┐     │
│  │   SOCKET SUBSYSTEM   │       │     NETFILTER SUBSYSTEM          │     │
│  │                      │       │                                  │     │
│  │  - App ↔ Kernel      │       │  - Packet processing pipeline    │     │
│  │    interface          │       │  - Hook points in the stack:     │     │
│  │  - fd, read, write   │       │    PREROUTING, INPUT, FORWARD,   │     │
│  │  - Send/recv buffers │       │    OUTPUT, POSTROUTING            │     │
│  │  - TCP state machine │       │  - NAT (DNAT, SNAT, MASQUERADE)  │     │
│  │                      │       │  - Conntrack (connection tracking)│     │
│  │  Knows: "I'm talking │       │  - Firewall (accept/drop/reject) │     │
│  │   to 10.96.0.1:80"   │       │                                  │     │
│  │                      │       │  Can silently rewrite 10.96.0.1  │     │
│  │  Does NOT know:      │       │  → 10.48.2.7 in the packet       │     │
│  │  netfilter exists    │       │  headers                          │     │
│  │                      │       │                                  │     │
│  │                      │       │  Does NOT know: which process     │     │
│  │                      │       │  or socket owns this packet       │     │
│  └─────────────────────┘       └──────────────────────────────────┘     │
│                                                                          │
│  They share the same packets flowing through the stack, but neither      │
│  subsystem is aware of the other's existence.                            │
└─────────────────────────────────────────────────────────────────────────┘
```

### Why This Matters: kube-proxy DNAT Transparency

When a Pod connects to a Kubernetes Service ClusterIP (e.g., `10.96.0.1:80`), the application's socket thinks it is connected to `10.96.0.1:80`. But kube-proxy has installed iptables DNAT rules in the OUTPUT and PREROUTING chains. As the packet leaves the socket layer and passes through netfilter, the destination address is rewritten to a real Pod IP (e.g., `10.48.2.7:8080`). The conntrack entry records this mapping so that return packets are reverse-NATted before reaching the socket.

```
  App: conn = dial("10.96.0.1:80")    ← App thinks it's talking to ClusterIP
         │
         ▼
  Socket layer: dst = 10.96.0.1:80    ← Socket sees ClusterIP
         │
         ▼
  Netfilter OUTPUT chain:
    DNAT rule: 10.96.0.1:80 → 10.48.2.7:8080   ← kube-proxy iptables rule
    conntrack entry: {10.0.0.5:42301 → 10.96.0.1:80} ↔ {10.0.0.5:42301 → 10.48.2.7:8080}
         │
         ▼
  IP layer: dst = 10.48.2.7:8080     ← Actual packet on the wire
         │
         ▼
  [packet reaches Pod at 10.48.2.7]

  Return path:
  Pod replies: src = 10.48.2.7:8080
         │
         ▼
  Netfilter PREROUTING: conntrack reverse-NAT
    src rewritten: 10.48.2.7:8080 → 10.96.0.1:80
         │
         ▼
  Socket layer: sees reply from 10.96.0.1:80    ← Socket never saw the real Pod IP
         │
         ▼
  App: reads from conn (thinks it's from 10.96.0.1:80)
```

The application is **completely unaware** that DNAT happened. It connected to `10.96.0.1:80`, it reads and writes to `10.96.0.1:80`, and it closes its connection to `10.96.0.1:80`. The netfilter subsystem transparently rewrote every packet, and conntrack ensured return packets were properly un-rewritten. This is why kube-proxy's iptables mode "just works" without any application code changes.

For the full details on how kube-proxy writes these iptables chains, conntrack mechanics, and race conditions with conntrack table overflow, see [[notes/Networking/container-networking-internals|Container Networking Internals]].

---

## SYN Queue and Accept Queue Internals

The two-queue model deserves a closer look because it is the source of many production issues (SYN floods, connection timeouts, slow `accept()` loops).

```
                     Incoming SYN
                         │
                         ▼
              ┌─────────────────────┐
              │     SYN Queue        │   (aka "half-open" queue)
              │  (incomplete conns)  │
              │                     │
              │  State: SYN_RECV     │   Kernel sent SYN-ACK, waiting for ACK
              │  Size: net.ipv4.     │
              │   tcp_max_syn_backlog│   (default: 128-1024 depending on distro)
              │                     │
              │  If full:           │
              │   - SYN cookies     │   (if net.ipv4.tcp_syncookies=1)
              │   - Or drop SYN     │
              └────────┬────────────┘
                       │  (3rd ACK arrives → handshake complete)
                       ▼
              ┌─────────────────────┐
              │    Accept Queue      │   (aka "completed" queue)
              │  (established conns) │
              │                     │
              │  State: ESTABLISHED  │   Fully connected, waiting for accept()
              │  Size: min(backlog   │
              │    arg, somaxconn)   │   backlog = listen(fd, backlog)
              │                     │   somaxconn = net.core.somaxconn (default 4096)
              │                     │
              │  If full:           │
              │   - Drop final ACK  │   (tcp_abort_on_overflow=0, default)
              │   - Or send RST     │   (tcp_abort_on_overflow=1)
              └────────┬────────────┘
                       │  accept() called by application
                       ▼
              ┌─────────────────────┐
              │  New Connected Socket│
              │  fd returned to app  │
              └─────────────────────┘
```

### SYN Cookies

When the SYN queue is full (SYN flood attack or legitimate burst), and `tcp_syncookies=1` (default on most distros), the kernel does **not** allocate a SYN queue entry. Instead, it encodes the connection parameters (MSS, timestamp, etc.) into the **initial sequence number** of the SYN-ACK. When the client's final ACK arrives, the kernel validates the cookie and creates the connection directly in the accept queue, bypassing the SYN queue entirely. This is described in RFC 4987 and implemented per D.J. Bernstein's original SYN cookies proposal.

Trade-off: SYN cookies disable TCP options negotiated in the SYN (like window scaling beyond a small set of values and SACK). This is acceptable for defense against SYN floods but slightly reduces TCP performance for legitimate connections.

### Monitoring Queue Depths

```bash
# View SYN queue (SYN_RECV state connections):
ss -tn state syn-recv | wc -l

# View accept queue overflow (connections dropped due to full accept queue):
netstat -s | grep "SYNs to LISTEN"
#  or
nstat -az TcpExtListenOverflows
nstat -az TcpExtListenDrops

# View current accept queue depth per listening socket:
ss -ltn
#  Recv-Q = current accept queue depth
#  Send-Q = accept queue max size (backlog)
```

---

## Interview Prep

### Q: What happens in the kernel between `listen()` and `accept()`? Walk through the full flow when a client connects.

**A:**

```
  Server app calls listen(fd, 128)
    → Kernel converts socket to LISTEN state
    → Creates SYN queue (max: tcp_max_syn_backlog)
    → Creates accept queue (max: min(128, somaxconn))

  Client sends SYN
    → Kernel (NOT the app) receives the SYN
    → Creates SYN queue entry (state: SYN_RECV)
    → Sends SYN-ACK back to client
    → Starts retransmit timer for SYN-ACK

  Client sends final ACK
    → Kernel matches to SYN queue entry
    → Moves connection to accept queue (state: ESTABLISHED)
    → The connection is FULLY ESTABLISHED at this point
    → Application has NOT been involved yet

  Server app calls accept()
    → Kernel pops the head of the accept queue
    → Creates new struct sock + file descriptor
    → Returns new fd to application
    → This new fd is the connected socket (ESTABLISHED)
    → Listening socket remains unchanged, still accepting new SYNs
```

The critical insight: the 3-way handshake completes **entirely in the kernel**. The application only sees the result when it calls `accept()`. If `accept()` is slow, completed connections queue up. If the accept queue overflows, the kernel silently drops incoming connections (the client's final ACK is dropped, causing the client to retransmit the ACK, eventually timing out).

---

### Q: Can two connections share the same local port? How does the kernel tell them apart?

**A:** Yes. Every connected socket on a server shares the same local port (the one passed to `bind()`). The kernel identifies each connection by its **4-tuple**:

```
(local IP, local port, remote IP, remote port)
```

Example: a server listening on port 8080 with three clients:

```
┌──────────────────────────────────────────────────────────────┐
│  Listening:  0.0.0.0:8080  (LISTEN)                          │
│                                                              │
│  Connected sockets (all share local port 8080):              │
│                                                              │
│  fd=5:  (10.0.0.1:8080, 10.0.0.5:42301)  ESTABLISHED       │
│  fd=6:  (10.0.0.1:8080, 10.0.0.5:42302)  ESTABLISHED       │
│  fd=7:  (10.0.0.1:8080, 10.0.0.6:51990)  ESTABLISHED       │
│                                                              │
│  Even fd=5 and fd=6 share both local IP AND local port.      │
│  They differ only in remote port (42301 vs 42302).           │
│  The 4-tuple is unique → kernel can demultiplex.             │
└──────────────────────────────────────────────────────────────┘
```

When a packet arrives, the kernel looks up the 4-tuple in a hash table to find the matching `struct sock`. This lookup is O(1) amortized.

---

### Q: What is the relationship between the TCP receive buffer and the TCP receive window? How does flow control work?

**A:**

The TCP receive window (advertised in the `Window` field of every ACK) is directly derived from the **free space in the receive buffer**:

```
  Receive buffer size: 128 KB
  Data in buffer (unread by app): 100 KB
  Free space: 28 KB

  → TCP advertises Window = 28 KB in next ACK
  → Sender can send at most 28 KB more before waiting

  If app doesn't read, buffer fills:
  Free space → 0 KB → Window = 0 → Sender STOPS

  Sender enters "persist" mode:
    → Sends "window probe" (1-byte segment) every ~30-60s
    → Waits for a non-zero window advertisement
    → When app reads data → free space opens → Window > 0 → Sender resumes
```

This is TCP **flow control** (RFC 9293, Section 3.8.6). It protects the receiver from being overwhelmed by a fast sender. It is entirely separate from **congestion control** (which protects the network). The actual sending rate is limited by `min(receive window, congestion window)`.

---

### Q: An application calls `write(fd, data, 1MB)`. Does the data immediately go on the wire?

**A:** No. The `write()` syscall copies data from userspace into the kernel **send buffer**. What happens next depends on multiple factors:

```
  App: write(fd, data, 1MB)
    │
    ▼
  Kernel: copy 1MB into send buffer
    │
    │  write() RETURNS here (data accepted by kernel)
    │  The app can continue doing other work.
    │  Kernel now "owns" the data.
    │
    ▼
  TCP decides WHEN and HOW MUCH to send:
    │
    ├── Nagle's algorithm: may coalesce small writes
    │     (disabled with TCP_NODELAY for latency-sensitive apps)
    │
    ├── Congestion window (cwnd): limits bytes in flight
    │     (slow start / congestion avoidance)
    │
    ├── Receive window (rwnd): receiver's advertised capacity
    │
    ├── MSS: each segment carries at most ~1460 bytes payload
    │     1MB → at least 700 TCP segments
    │
    └── Retransmission: kernel keeps data in send buffer until
        ACKed. Lost segments are retransmitted from this copy.

  Key: write() returning does NOT mean data is on the wire.
       It means the kernel accepted the data into the send buffer.
       The kernel sends it at its own pace.
```

If the send buffer is full when `write()` is called, a blocking socket will cause `write()` to block until space is available. A non-blocking socket returns `EAGAIN`.

---

### Q: How does kube-proxy's iptables DNAT work without the application knowing? Walk through the packet path.

**A:**

```
  Pod A wants to reach Service "my-svc" at ClusterIP 10.96.0.1:80
  kube-proxy has installed DNAT rule: 10.96.0.1:80 → 10.48.2.7:8080

  ┌─ Pod A (10.0.0.5) ────────────────────────────────────────────────┐
  │                                                                    │
  │  App: conn, _ := net.Dial("tcp", "10.96.0.1:80")                  │
  │       Socket layer: dst = 10.96.0.1:80                            │
  │       getpeername() → 10.96.0.1:80  (this NEVER changes)          │
  │                                                                    │
  │  Packet leaves socket layer:                                       │
  │    IP dst: 10.96.0.1    TCP dst port: 80                           │
  │         │                                                          │
  │         ▼                                                          │
  │  Netfilter OUTPUT chain:                                           │
  │    Rule: -d 10.96.0.1/32 -p tcp --dport 80 -j DNAT               │
  │           --to-destination 10.48.2.7:8080                          │
  │    Conntrack records: orig={10.0.0.5:42301→10.96.0.1:80}          │
  │                       reply={10.48.2.7:8080→10.0.0.5:42301}       │
  │         │                                                          │
  │         ▼                                                          │
  │  Packet on wire:                                                   │
  │    IP dst: 10.48.2.7    TCP dst port: 8080    ← REWRITTEN         │
  └────────────────────────────────────────────────────────────────────┘

  Reply from Pod B (10.48.2.7:8080):
    → Arrives at Pod A's node
    → Netfilter PREROUTING: conntrack reverse-NAT
    → src rewritten: 10.48.2.7:8080 → 10.96.0.1:80
    → Socket layer sees reply from 10.96.0.1:80 ← matches socket's 4-tuple
    → App reads data, completely unaware of 10.48.2.7
```

The key architectural insight: **sockets and netfilter are independent subsystems**. The socket layer records `dst = 10.96.0.1:80` at `connect()` time. Netfilter rewrites packets after they leave the socket layer (on send) and before they reach the socket layer (on receive). The socket never sees the rewritten addresses. This is why DNAT is transparent -- no application code changes needed.

---

### Q: What happens when the accept queue is full? How do you diagnose it?

**A:**

When the accept queue is full and a new 3-way handshake completes (client sends final ACK):

| `tcp_abort_on_overflow` | Behavior |
|---|---|
| 0 (default) | Kernel silently **drops** the client's final ACK. The server's SYN-ACK retransmit timer fires, sending another SYN-ACK. Client retransmits ACK. This creates a retry loop. From the client's perspective, the connection "hangs" after SYN-ACK. |
| 1 | Kernel sends **RST** to client. Client sees "Connection reset by peer" immediately. Faster failure, but more aggressive. |

Diagnosis:

```bash
# Check for accept queue overflows:
nstat -az TcpExtListenOverflows
# Non-zero = connections were dropped because accept queue was full

# Check current queue depth per listening socket:
ss -ltn
# State    Recv-Q   Send-Q   Local Address:Port
# LISTEN   15       4096     0.0.0.0:8080
#          ^^^      ^^^^
#          current  max (backlog)
#          queue    size
#          depth

# If Recv-Q is consistently near Send-Q → app is too slow calling accept()
# Fix: increase backlog, speed up accept() loop, or use SO_REUSEPORT
```

---

### Q: What is `SO_REUSEADDR` and why does almost every server set it?

**A:** When a TCP connection closes, it enters the `TIME_WAIT` state for 2*MSL (Maximum Segment Lifetime, typically 60 seconds on Linux). During `TIME_WAIT`, the 4-tuple is reserved in the kernel to handle stale duplicate packets from the old connection.

Without `SO_REUSEADDR`, if you restart a server, `bind()` fails with `EADDRINUSE` because the old listening socket's port is still held by `TIME_WAIT` connections. `SO_REUSEADDR` tells the kernel: "let me bind to this port even if there are `TIME_WAIT` connections on it."

```go
// In Go, the net package sets SO_REUSEADDR automatically.
// You get this for free with net.Listen().
// In C, you must set it explicitly:
//   int opt = 1;
//   setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

> **Note:** `SO_REUSEADDR` does **not** allow two active listeners on the same port. It only allows binding when existing connections are in `TIME_WAIT`. For true multi-listener on the same port (load balancing across threads), use `SO_REUSEPORT`.

---

### Q: What is the difference between `SO_REUSEPORT` and `SO_REUSEADDR`?

**A:**

| | `SO_REUSEADDR` | `SO_REUSEPORT` |
|---|---|---|
| **Purpose** | Bypass `TIME_WAIT` bind conflicts | Allow multiple listeners on same port |
| **Multiple LISTEN sockets on same port?** | No | Yes |
| **Load balancing** | N/A | Kernel distributes incoming connections across all listening sockets (consistent hashing by 4-tuple) |
| **Use case** | Server restart without waiting for `TIME_WAIT` to expire | Multi-threaded/multi-process accept (e.g., NGINX, envoy) to avoid accept queue contention |
| **Security** | Any process can bind after `TIME_WAIT` | Only processes with same effective UID can share |

---

### Q: An engineer says "each TCP connection needs its own port." Is this correct?

**A:** No. This is one of the most common networking misconceptions. The **server** uses one port (the listening port) for all connections. Each connection is uniquely identified by the **4-tuple** `(src IP, src port, dst IP, dst port)`, not just the port.

The **client** does use a unique ephemeral port per connection (assigned by the kernel from the range in `/proc/sys/net/ipv4/ip_local_port_range`, typically 32768-60999). So the client-side limit per destination IP is ~28,000 simultaneous connections. This can be a bottleneck for load generators or reverse proxies making many outbound connections to the same backend. Solutions: multiple source IPs (`ip_local_port_range` expansion is limited), or connection pooling/multiplexing (HTTP/2, gRPC).

The server side has no such port limit -- it can handle millions of connections on a single port, limited only by memory and file descriptors.

---

## Related Notes

- [[notes/Networking/container-networking-internals|Container Networking Internals]]
- [[notes/Networking/tls-1.3-handshake|TLS 1.3 Handshake]]
- [[notes/Networking/http-grpc-and-streaming|HTTP, gRPC, and Streaming]]
