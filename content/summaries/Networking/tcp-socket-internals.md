---
title: "Summary: TCP Socket Internals"
---

> **Full notes:** [[notes/Networking/tcp-socket-internals|TCP Socket Internals -->]]

## Key Concepts

### Sockets as Kernel Abstractions
- Socket = file descriptor (integer) wrapping kernel `struct sock`
- App sees byte stream; kernel handles TCP segments, IP headers, routing, netfilter
- Every socket op crosses userspace-kernel boundary via syscall (context switch ring 3 -> ring 0)
- Key syscalls: `socket()`, `bind()`, `listen()`, `accept()`, `connect()`, `read()`, `write()`, `close()`

### Listening Socket vs Connected Socket
- **Listening socket**: LISTEN state, bound to `0.0.0.0:port`, never carries data, manages two queues
- **Connected socket**: ESTABLISHED state, unique 4-tuple `(local IP, local port, remote IP, remote port)`
- `accept()` returns a NEW fd with its own buffers, independent of listening socket
- One port handles thousands of connections -- kernel demuxes by full 4-tuple (O(1) hash lookup)
- Theoretical max: ~2^48 unique 4-tuples; practical limits: `ulimit -n`, memory (~3-10 KB/socket)

### 3-Way Handshake (Entirely Kernel-Managed)
- App does NOT participate -- kernel handles SYN, SYN-ACK, ACK
- Connection is ESTABLISHED before `accept()` is called
- **SYN queue** (half-open): SYN received, SYN-ACK sent, awaiting final ACK
- **Accept queue** (completed): handshake done, waiting for `accept()`
- Accept queue overflow: kernel drops final ACK silently (`tcp_abort_on_overflow=0`)
- SYN cookies (`tcp_syncookies=1`): encode params in seq num, bypass SYN queue during floods

### Socket Buffers & Flow Control
- Each connected socket has kernel **send buffer** (`sk_write_queue`) and **receive buffer** (`sk_receive_queue`)
- `write()` copies to send buffer and returns -- does NOT mean data is on the wire
- **Receive buffer free space = TCP receive window** (advertised in every ACK)
- Buffer full -> window=0 -> sender STOPS (flow control, RFC 9293)
- Flow control (protects receiver) != congestion control (protects network)
- Effective send window = `min(rwnd, cwnd)`
- BDP formula: `buffer >= bandwidth * RTT` (128KB buffer + 100ms RTT = max 10 Mbps)

### Netfilter vs Sockets: Independent Subsystems
- Socket layer and netfilter are completely unaware of each other
- kube-proxy DNAT is transparent: app connects to ClusterIP, netfilter rewrites to Pod IP
- Conntrack records mapping; return packets are reverse-NATted automatically
- App never sees the real Pod IP -- `getpeername()` always returns ClusterIP

## Quick Reference

```
Packet Flow: write() -> Socket Layer -> TCP (segment, seq nums, checksum)
  -> IP (header, routing) -> Netfilter (OUTPUT, POSTROUTING) -> NIC -> Wire

Receive: Wire -> NIC -> Netfilter (PREROUTING, INPUT) -> TCP (demux by 4-tuple)
  -> Socket receive buffer -> read()

Buffer Tuning:
  /proc/sys/net/ipv4/tcp_rmem   4096  131072  6291456  (min/default/max)
  /proc/sys/net/ipv4/tcp_wmem   4096  16384   4194304

Queue Monitoring:
  ss -ltn                       # Recv-Q=current depth, Send-Q=max
  ss -tn state syn-recv | wc -l # SYN queue depth
  nstat -az TcpExtListenOverflows  # accept queue drops

Key Sysctls:
  net.core.somaxconn             # max accept queue size (default 4096)
  net.ipv4.tcp_max_syn_backlog   # max SYN queue size
  net.ipv4.tcp_syncookies        # SYN flood defense (default 1)
  net.ipv4.tcp_abort_on_overflow # RST on full accept queue (default 0)
```
