---
title: "Summary: HTTP, gRPC & Streaming Protocols"
---

> **Full notes:** [[notes/Networking/http-grpc-and-streaming|HTTP, gRPC & Streaming -->]]

## Key Concepts

### HTTP/1.1
- Persistent connections (keep-alive) default -- reuse TCP conn across requests
- **HOL blocking**: sequential on single conn; browsers open up to 6 parallel TCP conns per origin
- Chunked transfer encoding: stream responses without knowing total size upfront
- Pipelining: abandoned (FIFO ordering, buggy proxies, non-idempotent methods unsafe)

### HTTP/2 Multiplexing
- Binary framing layer: 9-byte frame header + payload, each frame tagged with stream ID
- **Stream**: bidirectional flow of frames (one per request/response pair)
- Frames from different streams interleaved on wire -- eliminates app-layer HOL blocking
- **TCP-level HOL blocking still exists** -- lost packet blocks all streams (motivates HTTP/3/QUIC)
- HPACK header compression: static table (61 entries) + dynamic table + Huffman; 85-90% reduction
- Flow control: per-connection (65KB default) + per-stream (65KB default) via WINDOW_UPDATE
- Server push removed from Chrome 2022; replaced by HTTP 103 Early Hints

### Go HTTP Client Pitfalls
- **`MaxIdleConnsPerHost: 2`** (default) is almost always too low for service-to-service
- Production: set `MaxIdleConnsPerHost: 100`, `MaxConnsPerHost: 100`
- **Must drain response body** (`io.Copy(io.Discard, resp.Body)`) or connection leaks -> `too many open files`
- Connection not returned to pool if body not fully read

### gRPC over HTTP/2
- Each RPC = one HTTP/2 stream; path = `/package.Service/Method`
- Protobuf framing: 1-byte compression flag + 4-byte length (big-endian) + payload
- **gRPC status in trailers**, not HTTP status -- HTTP 200 can mean gRPC UNAVAILABLE (14)
- L7 load balancers required -- L4 sends all RPCs to same backend on one connection
- Keepalive: HTTP/2 PING frames, not TCP keepalive; `ENHANCE_YOUR_CALM` if client pings too aggressively

### gRPC Streaming Patterns
- **Unary** (1:1), **Server streaming** (1:N), **Client streaming** (N:1), **Bidirectional** (N:N)
- Flow control: slow consumer fills stream window -> backpressure; can block connection-level window
- Tune: `InitialWindowSize` and `InitialConnWindowSize` (default 64KB too small; use 1MB+)

### SSE (Server-Sent Events)
- `Content-Type: text/event-stream`, fields: `data`, `event`, `id`, `retry`
- Built-in auto-reconnection via `EventSource` API; `Last-Event-ID` header on reconnect
- **Why LLMs use SSE**: unidirectional (1 prompt, N tokens), HTTP-native, proxy-friendly, simple
- Limitations: unidirectional only, text-only, 6-connection limit in HTTP/1.1

### WebSockets
- HTTP/1.1 upgrade handshake (101 Switching Protocols), then raw TCP
- Full-duplex, native binary support (opcode 0x2), Ping/Pong keepalive
- Client frames MUST be masked; server frames MUST NOT
- No auto-reconnection, no multiplexing, proxy traversal can be problematic

### Comparison Decision Matrix

| Use Case | Best Choice |
|---|---|
| Internal service-to-service | gRPC (protobuf, multiplexing, deadlines) |
| Public APIs, browser clients | REST/JSON |
| LLM token streaming | SSE |
| Full-duplex (chat, games) | WebSocket |
| Behind restrictive proxies | Long polling (fallback) |

## Quick Reference

```
gRPC vs REST decision:
  gRPC: internal svc-to-svc, streaming, performance, typed contracts, deadline propagation
  REST: public APIs, browser/mobile, human-readable, existing HTTP/1.1 infra

L4 vs L7 LB for gRPC:
  L4: 1 TCP conn -> all RPCs to 1 backend (broken)
  L7: routes individual streams to different backends (correct)

gRPC Status Codes:
  0=OK  1=CANCELLED  3=INVALID_ARGUMENT  4=DEADLINE_EXCEEDED
  5=NOT_FOUND  7=PERMISSION_DENIED  8=RESOURCE_EXHAUSTED  14=UNAVAILABLE

Go HTTP Transport (production):
  MaxIdleConnsPerHost: 100
  MaxConnsPerHost: 100
  IdleConnTimeout: 30s
  Always: defer resp.Body.Close() + io.Copy(io.Discard, resp.Body)

HTTP/2 Flow Control:
  TCP flow control (kernel, byte-stream) vs HTTP/2 flow control (app-level, per-stream)
  Independent and layered -- both can independently limit sending
```
