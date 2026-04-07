---
title: "HTTP, gRPC & Streaming Protocols"
---

This note is a low-level reference on how HTTP connections work, how gRPC builds on HTTP/2, and how streaming protocols (SSE, WebSockets, long polling) compare in practice. Written for platform engineers who need to debug real connection issues and answer deep interview questions.

## HTTP/1.1 Connection Model

### Persistent Connections (Keep-Alive)

Before HTTP/1.1, every request opened a new TCP connection. HTTP/1.1 made persistent connections the **default** -- the connection stays open after a response so it can be reused for subsequent requests.

```
Client                          Server
  |                               |
  |------- TCP SYN -------------->|
  |<------ TCP SYN-ACK ----------|
  |------- TCP ACK -------------->|   (3-way handshake, once)
  |                               |
  |------- GET /page HTTP/1.1 -->|
  |<------ 200 OK + body --------|
  |                               |   (connection stays open)
  |------- GET /style.css ------>|
  |<------ 200 OK + body --------|
  |                               |   (reused, no new handshake)
  |------- Connection: close --->|
  |<------ FIN ------------------|
```

The `Connection: keep-alive` header is implicit in HTTP/1.1. The server or client can send `Connection: close` to signal it wants to tear down after the current response.

### Head-of-Line (HOL) Blocking

HTTP/1.1 is strictly sequential on a single connection: request 1 must complete before request 2 can begin. If the server is slow generating the response to request 1, every subsequent request on that connection is blocked behind it.

```
Connection 1 (sequential):
  |-- GET /slow (500ms) --|-- GET /fast (10ms) --|-- GET /fast (10ms) --|
  Total: 520ms

What browsers actually do (6 parallel connections):
  Conn 1: |-- GET /slow (500ms) --|
  Conn 2: |-- GET /fast (10ms) --|
  Conn 3: |-- GET /fast (10ms) --|
  Conn 4: |-- GET /fast (10ms) --|
  Conn 5: (idle)
  Conn 6: (idle)
  Total: 500ms (but uses 4 TCP connections + 4 TLS handshakes)
```

Browsers work around HOL blocking by opening **up to 6 parallel TCP connections per origin**. This is expensive -- each connection requires a TCP handshake (1 RTT) and a TLS handshake (1-2 more RTTs).

### Content-Length vs Chunked Transfer Encoding

HTTP/1.1 has two ways to frame a response body:

**Content-Length** -- the server knows the body size upfront:
```
HTTP/1.1 200 OK
Content-Length: 1234
Content-Type: application/json

{"data": "...exactly 1234 bytes..."}
```

**Chunked Transfer Encoding** -- the server sends the body in chunks, each prefixed with its hex-encoded size. Terminated by a zero-length chunk:
```
HTTP/1.1 200 OK
Transfer-Encoding: chunked

1a\r\n                          <-- 26 bytes in hex
This is the first chunk.\r\n
1c\r\n                          <-- 28 bytes
And this is the second one.\r\n
0\r\n                           <-- zero-length chunk = end
\r\n
```

Chunked encoding is critical for streaming responses where the server does not know the total size ahead of time (log tailing, SSE, LLM token streaming). Without it, the server would have to buffer the entire response in memory to compute Content-Length.

### HTTP Pipelining (and Why It Failed)

HTTP/1.1 pipelining allows a client to send multiple requests without waiting for responses, but responses **must arrive in the same order as requests** (FIFO). This still suffers from HOL blocking at the response level, plus:

- Intermediary proxies often do not support it.
- A single slow response stalls everything behind it.
- Non-idempotent methods (POST) cannot be pipelined safely.
- Buggy server implementations would send responses out of order.

All major browsers **disabled pipelining by default**. It was abandoned in favor of HTTP/2 multiplexing.

## HTTP/2 Multiplexing

HTTP/2 solves HOL blocking at the application layer by introducing a **binary framing layer** that multiplexes multiple logical streams over a single TCP connection.

### Binary Framing Layer

HTTP/2 replaces the text-based HTTP/1.1 wire format with binary frames:

```
HTTP/2 Frame Structure (9-byte header + payload):

+-----------------------------------------------+
|                Length (24 bits)                |
+-------+---------------------------------------+
| Type  |       Flags (8 bits)                  |
| (8b)  +---+-----------------------------------+
+-------+-+-+-----------------------------------+
|R|         Stream Identifier (31 bits)         |
+-+---------------------------------------------+
|              Frame Payload (0...)              |
+-----------------------------------------------+

Frame Types:
  0x0 = DATA          (request/response body)
  0x1 = HEADERS       (HTTP headers, compressed)
  0x2 = PRIORITY      (stream priority, deprecated in RFC 9113)
  0x3 = RST_STREAM    (cancel a stream)
  0x4 = SETTINGS      (connection-level config)
  0x5 = PUSH_PROMISE  (server push)
  0x6 = PING          (keepalive / RTT measurement)
  0x7 = GOAWAY        (graceful shutdown)
  0x8 = WINDOW_UPDATE (flow control)
  0x9 = CONTINUATION  (continued HEADERS)
```

### Streams, Messages, and Frames

```
Single TCP Connection
+----------------------------------------------------------+
|  Stream 1: HEADERS frame -> DATA frame -> DATA frame     |
|  Stream 3: HEADERS frame -> DATA frame                   |
|  Stream 5: HEADERS frame -> DATA frame -> DATA frame     |
+----------------------------------------------------------+

On the wire (interleaved):
  [HEADERS s1] [HEADERS s3] [DATA s1] [DATA s3] [DATA s5] [HEADERS s5] [DATA s1]
```

Key concepts:
- **Stream**: A bidirectional flow of frames within a connection. Each request/response pair uses one stream. Odd-numbered streams are client-initiated; even-numbered are server-initiated (push).
- **Message**: A logical HTTP request or response, made up of one HEADERS frame and zero or more DATA frames.
- **Frame**: The smallest unit of communication. Frames from different streams are interleaved on the wire.

This eliminates application-layer HOL blocking: a slow response on stream 1 does not prevent frames from stream 3 from being sent and received. However, **TCP-level HOL blocking still exists** -- a single lost TCP segment blocks all streams until retransmission completes. This is what QUIC (HTTP/3) solves.

### HPACK Header Compression

HTTP headers are repetitive across requests (same Host, same User-Agent, same cookies). HPACK compresses them using:

1. **Static table**: 61 pre-defined common header name-value pairs (`:method: GET`, `:status: 200`, etc.)
2. **Dynamic table**: Connection-specific table built up as headers are sent. Both sides maintain a synchronized copy.
3. **Huffman encoding**: Literal values are Huffman-coded for further compression.

In practice HPACK reduces header overhead by 85-90% compared to HTTP/1.1.

### Flow Control

HTTP/2 has **two levels of flow control**, both using WINDOW_UPDATE frames:

```
Per-Connection Flow Control:
  Total window shared by all streams (default: 65,535 bytes)

Per-Stream Flow Control:
  Each stream has its own window (default: 65,535 bytes)

Sender tracks:  remaining_window = initial_window - bytes_sent + window_updates_received
Receiver sends: WINDOW_UPDATE(stream_id, increment) when it has consumed data
```

Flow control applies only to DATA frames, not HEADERS. It prevents a fast sender from overwhelming a slow receiver. In gRPC, misconfigured flow control windows are a common source of throughput issues -- the default 64KB window is too small for high-bandwidth streams; gRPC typically sets it to 16MB+.

### Server Push

The server can proactively push resources the client has not requested yet, using PUSH_PROMISE frames. In practice, server push was rarely used correctly and was **removed from Chrome in 2022**. HTTP 103 Early Hints is the modern replacement.

## Go HTTP Client Internals

Understanding `net/http`'s connection pooling is essential for debugging timeout issues in Go services.

### Transport and Connection Pooling

```go
// Default transport (simplified)
var DefaultTransport = &http.Transport{
    MaxIdleConns:          100,           // total idle connections across all hosts
    MaxIdleConnsPerHost:   2,             // idle connections per host (THE KEY KNOB)
    MaxConnsPerHost:       0,             // 0 = unlimited active connections
    IdleConnTimeout:       90 * time.Second,
    TLSHandshakeTimeout:  10 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}
```

**The default `MaxIdleConnsPerHost: 2` is almost always too low for service-to-service communication.** If your service makes 50 concurrent requests to another service, only 2 connections will be reused from the pool -- the other 48 will open new TCP+TLS connections, then be closed immediately after use because the pool is full.

```go
// Production-grade transport for internal services
transport := &http.Transport{
    MaxIdleConnsPerHost: 100,
    MaxConnsPerHost:     100,
    IdleConnTimeout:     30 * time.Second,
    // For HTTP/2: ForceAttemptHTTP2 is true by default when TLS is used
}
client := &http.Client{
    Transport: transport,
    Timeout:   10 * time.Second,
}
```

### Connection Lifecycle

```
Request comes in
       |
       v
  Pool has idle conn for this host?
       |                    |
      YES                  NO
       |                    |
       v                    v
  Check: is conn          Dial new TCP conn
  still alive?            + TLS handshake
  (read 1 byte with       |
   very short deadline)    v
       |              Use new conn
      YES   NO
       |     |
       v     v
  Reuse   Discard, dial new
       |
       v
  Send request, read response
       |
       v
  Was response body fully read?  <-- CRITICAL
       |                    |
      YES                  NO
       |                    |
       v                    v
  Return conn to pool    Connection is LEAKED
  (if pool not full)     (cannot be reused)
```

**Critical gotcha**: You **must** read the entire response body and close it, or the connection cannot be returned to the pool:

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()

// MUST drain the body even if you don't need it
io.Copy(io.Discard, resp.Body)
```

If you only read part of the body or forget `resp.Body.Close()`, the connection is leaked. Under load, this exhausts file descriptors and causes `dial: too many open files`.

### When Connections Get Closed

- **Idle timeout**: After `IdleConnTimeout` (default 90s) with no activity.
- **Server closes**: Server sends `Connection: close` or the TCP connection is reset.
- **Pool full**: The connection is not returned to the pool if `MaxIdleConnsPerHost` connections are already idle.
- **Body not drained**: Connection cannot be reused (leaked).
- **HTTP/2 GOAWAY**: Server sends GOAWAY to gracefully drain the connection.

## gRPC over HTTP/2

### Why gRPC Chose HTTP/2

gRPC needs multiplexing (many concurrent RPCs on one connection), bidirectional streaming, header compression, and flow control. HTTP/2 provides all of these natively. Building on HTTP/2 also means gRPC traffic can traverse standard proxies, load balancers, and CDNs that speak HTTP/2.

### How RPCs Map to Streams

Each gRPC call is one HTTP/2 stream:

```
gRPC Unary Call -> 1 HTTP/2 Stream:

Client                                        Server
  |                                             |
  |-- HEADERS (stream 1) ---------------------->|
  |   :method: POST                             |
  |   :path: /mypackage.MyService/MyMethod      |
  |   content-type: application/grpc            |
  |   grpc-timeout: 5S                          |
  |                                             |
  |-- DATA (stream 1) ------------------------->|
  |   [gRPC length-prefixed message]            |
  |                                             |
  |<- HEADERS (stream 1) -----------------------|
  |   :status: 200                              |
  |   content-type: application/grpc            |
  |                                             |
  |<- DATA (stream 1) --------------------------|
  |   [gRPC length-prefixed message]            |
  |                                             |
  |<- HEADERS (stream 1, END_STREAM) -----------|   <-- trailers
  |   grpc-status: 0                            |
  |   grpc-message: OK                          |
```

### Protobuf Framing (Length-Prefixed Messages)

gRPC wraps each Protobuf message in a 5-byte prefix:

```
gRPC Message Frame:
+--+--+--+--+--+--+--+--+--+--+--+--+
|C |  Message Length (4 bytes, BE)   |
+--+--+--+--+--+--+--+--+--+--+--+--+
|          Protobuf Payload          |
+------------------------------------+

C = Compression flag (1 byte):
    0x00 = uncompressed
    0x01 = compressed (using algorithm from grpc-encoding header)
```

This framing allows multiple gRPC messages to be carried in a single HTTP/2 DATA frame, or one message to span multiple DATA frames.

### gRPC Status Codes vs HTTP Status Codes

gRPC has its own status code space, sent in **trailers** (not the initial HTTP status):

| gRPC Status | Code | HTTP Equivalent | Meaning |
|---|---|---|---|
| OK | 0 | 200 | Success |
| CANCELLED | 1 | 499 | Client cancelled |
| UNKNOWN | 2 | 500 | Unknown error |
| INVALID_ARGUMENT | 3 | 400 | Bad request |
| DEADLINE_EXCEEDED | 4 | 504 | Timeout |
| NOT_FOUND | 5 | 404 | Not found |
| ALREADY_EXISTS | 6 | 409 | Conflict |
| PERMISSION_DENIED | 7 | 403 | Forbidden |
| RESOURCE_EXHAUSTED | 8 | 429 | Rate limited |
| UNAVAILABLE | 14 | 503 | Service unavailable |

The initial HTTP `:status` is almost always `200` for gRPC -- the real status is in the `grpc-status` trailer. This is why L7 load balancers that only look at HTTP status codes see all gRPC traffic as "200 OK" even when the RPC failed. You need gRPC-aware load balancers and observability.

### Metadata, Headers, and Trailers

gRPC metadata maps to HTTP/2 headers and trailers:

- **Request headers**: Sent in the initial HEADERS frame. Includes `:path`, `content-type`, `grpc-timeout`, and custom metadata (`x-request-id`, etc.)
- **Response headers**: Sent in the initial HEADERS frame of the response. Includes `:status`, `content-type`.
- **Response trailers**: Sent in a HEADERS frame with the END_STREAM flag. Contains `grpc-status`, `grpc-message`, and any trailing metadata. Trailers are **the only mechanism** in HTTP/2 to send metadata after the body.

## gRPC Streaming

### The Four RPC Patterns

```protobuf
service DataService {
  // Unary: 1 request, 1 response
  rpc GetItem(GetItemRequest) returns (Item);

  // Server streaming: 1 request, N responses
  rpc ListItems(ListItemsRequest) returns (stream Item);

  // Client streaming: N requests, 1 response
  rpc UploadItems(stream Item) returns (UploadSummary);

  // Bidirectional streaming: N requests, N responses
  rpc SyncItems(stream Item) returns (stream Item);
}
```

### How Each Pattern Maps to HTTP/2

```
Unary (1:1):
  Client: HEADERS + 1 DATA (END_STREAM) -->
  Server: HEADERS + 1 DATA + TRAILERS (END_STREAM) <--

Server Streaming (1:N):
  Client: HEADERS + 1 DATA (END_STREAM) -->
  Server: HEADERS + DATA + DATA + ... + TRAILERS (END_STREAM) <--

Client Streaming (N:1):
  Client: HEADERS + DATA + DATA + ... + DATA (END_STREAM) -->
  Server: HEADERS + 1 DATA + TRAILERS (END_STREAM) <--

Bidirectional Streaming (N:N):
  Client: HEADERS + DATA + DATA + ... -->    (END_STREAM when done)
  Server: HEADERS + DATA + DATA + ... <--    (END_STREAM when done)
  (Frames from both directions are interleaved independently)
```

For bidirectional streaming, both sides can send and receive independently. The client does not need to finish sending before the server starts responding. This is the basis for real-time communication patterns in gRPC (chat, multiplayer state sync, telemetry pipelines).

### Flow Control Implications

Each streaming RPC is one HTTP/2 stream with its own flow control window. If the consumer is slow:

1. The stream's receive window fills up.
2. The sender blocks, applying backpressure.
3. If many streams are slow, the connection-level window can also fill up, blocking **all** streams on that connection.

This is why gRPC services need careful tuning of `InitialWindowSize` and `InitialConnWindowSize` in high-throughput streaming scenarios:

```go
server := grpc.NewServer(
    grpc.InitialWindowSize(1 << 20),     // 1MB per-stream window
    grpc.InitialConnWindowSize(1 << 20), // 1MB per-connection window
    grpc.MaxRecvMsgSize(64 << 20),       // 64MB max message size
)
```

## Server-Sent Events (SSE)

SSE is a simple, HTTP-based protocol for server-to-client streaming. It uses a long-lived HTTP/1.1 (or HTTP/2) response with `Content-Type: text/event-stream`.

### Wire Format

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: First message\n\n

event: update
data: {"temperature": 72}\n\n

id: 42
event: heartbeat
data: ping\n\n

retry: 5000\n\n

data: Multi-line message\n
data: spans two data fields\n\n
```

Each event is terminated by a blank line (`\n\n`). Fields:
- **data**: The event payload. Multiple `data:` lines are concatenated with newlines.
- **event**: Event type (default is `"message"`). The client dispatches to `addEventListener(type, ...)`.
- **id**: Sets the last event ID. Sent back to the server on reconnection via `Last-Event-ID` header.
- **retry**: Tells the client how long (ms) to wait before reconnecting.

### Built-in Reconnection

The `EventSource` browser API **automatically reconnects** on failure:

```
Client                              Server
  |-- GET /events ------------------>|
  |<-- 200 text/event-stream -------|
  |<-- id: 1\ndata: hello\n\n ------|
  |<-- id: 2\ndata: world\n\n ------|
  |                                  |
  |         (connection drops)       |
  |                                  |
  |-- GET /events ------------------>|
  |   Last-Event-ID: 2              |   <-- automatic!
  |<-- 200 text/event-stream -------|
  |<-- id: 3\ndata: missed\n\n -----|   <-- server resumes from ID 3
```

The server can use `Last-Event-ID` to replay missed events. This is a massive advantage over WebSockets, which have no built-in reconnection or replay semantics.

### Why LLMs Use SSE for Token Streaming

LLM inference is a unidirectional stream: 1 prompt in, N tokens out. SSE is the ideal fit because:

1. **Unidirectional**: The client sends one request; the server streams tokens. No need for bidirectional communication.
2. **HTTP-native**: Works through any HTTP proxy, CDN, or load balancer. No special protocol support needed.
3. **Auto-reconnection**: If the connection drops, the client can resume (though most LLM APIs do not implement `Last-Event-ID` replay).
4. **Simple**: No framing complexity. Just text lines over HTTP.
5. **Browser-native**: `EventSource` API is built into every browser. No library needed.

OpenAI, Anthropic, and most LLM providers use SSE with `text/event-stream` for their streaming APIs:

```
data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":"Hello"}}]}

data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":" world"}}]}

data: [DONE]
```

### SSE Limitations

- **Unidirectional only**: Server to client. If you need the client to send data mid-stream, SSE is not enough.
- **Text-only**: Binary data must be Base64-encoded (inefficient).
- **HTTP/1.1 connection limit**: Browsers allow only **6 connections per origin**. Each SSE stream consumes one. In HTTP/2, this is not an issue (streams are multiplexed).
- **No built-in binary framing**: Unlike WebSockets, there are no opcodes or frame types.

## WebSockets

### HTTP Upgrade Handshake

WebSockets start as an HTTP/1.1 request that "upgrades" the connection:

```
Client -> Server:
  GET /chat HTTP/1.1
  Host: server.example.com
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  Sec-WebSocket-Version: 13

Server -> Client:
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

  (TCP connection is now a WebSocket -- no more HTTP framing)
```

The `Sec-WebSocket-Key` / `Sec-WebSocket-Accept` exchange is not for security -- it is to ensure that HTTP intermediaries do not accidentally treat WebSocket frames as HTTP. The accept value is `Base64(SHA1(key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))`.

### WebSocket Frame Structure

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |            (16/64)            |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
|                    Masking-key (0 or 4 bytes)                 |
+---------------------------------------------------------------+
|                    Payload Data                               |
+---------------------------------------------------------------+

Opcodes:
  0x0 = Continuation frame
  0x1 = Text frame (UTF-8)
  0x2 = Binary frame
  0x8 = Connection close
  0x9 = Ping
  0xA = Pong
```

Key details:
- **FIN bit**: 1 if this is the final fragment of a message. Large messages can be fragmented across multiple frames.
- **Masking**: Client-to-server frames MUST be masked (XOR with a 4-byte key). Server-to-client frames MUST NOT be masked. This prevents cache poisoning attacks on intermediary proxies.
- **Ping/Pong**: Either side can send a Ping; the other must respond with a Pong containing the same payload. Used for keepalive and latency measurement.

### WebSocket vs SSE vs gRPC Streaming

| Aspect | WebSocket | SSE | gRPC Streaming |
|---|---|---|---|
| Direction | Full-duplex | Server -> Client | All 4 patterns |
| Transport | TCP (after HTTP upgrade) | HTTP | HTTP/2 |
| Binary support | Native (opcode 0x2) | No (base64 needed) | Native (Protobuf) |
| Browser support | Yes | Yes | No (needs grpc-web proxy) |
| Auto-reconnect | No (manual) | Yes (EventSource) | No (manual) |
| Proxy traversal | Can be problematic | Excellent | Good (HTTP/2) |
| Multiplexing | No (1 conn per socket) | HTTP/2: yes | Yes (HTTP/2) |
| Schema/contract | None | None | Protobuf IDL |
| Backpressure | None built-in | None built-in | HTTP/2 flow control |

## Long Polling

Long polling is the simplest server push technique and works everywhere, including through the most restrictive proxies and firewalls.

### How It Works

```
Client                              Server
  |                                  |
  |-- GET /events?since=0 --------->|
  |                                  |  (server holds request open)
  |                                  |  ... waiting for new data ...
  |                                  |  (new data arrives)
  |<-- 200 OK + [event1, event2] ---|
  |                                  |
  |-- GET /events?since=2 --------->|  (immediately re-request)
  |                                  |  (server holds again...)
  |                                  |
```

The server holds the request open until either:
1. New data is available (returns it immediately).
2. A timeout is reached (returns empty, client re-requests).

### Comparison with SSE and WebSockets

| Aspect | Long Polling | SSE | WebSocket |
|---|---|---|---|
| Latency | 1 RTT per batch | Near-zero | Near-zero |
| Connection overhead | New request per batch | One persistent | One persistent |
| Proxy compatibility | Excellent | Good | Moderate |
| Implementation complexity | Low | Low | Moderate |
| Server resource usage | High (many pending requests) | Moderate | Low |

### When Long Polling Is Still Appropriate

- Behind corporate proxies that strip `Upgrade` headers or kill long-lived connections.
- When you need wide compatibility with very old HTTP clients.
- Low-frequency updates (once every few seconds) where the overhead is negligible.
- As a **fallback** when SSE or WebSocket connections fail.

## Interview Prep

### Q: When would you use gRPC vs REST?

**Use gRPC when:**
- Internal service-to-service communication where both sides are under your control.
- You need streaming (bidirectional or server-push).
- Performance matters: Protobuf is 3-10x smaller than JSON; HTTP/2 multiplexing reduces connection overhead.
- You want a strongly-typed contract (Protobuf IDL) with code generation.
- You need deadline propagation (gRPC timeouts propagate through the call chain via `grpc-timeout` header).

**Use REST when:**
- Public APIs consumed by browsers, third-party clients, or mobile apps.
- You need human-readable payloads for debugging (JSON).
- Your infrastructure (CDNs, API gateways, WAFs) is built around HTTP/1.1.
- The team is more familiar with REST conventions.

### Q: Why does L4 load balancing break with gRPC?

An L4 (TCP) load balancer distributes connections, not requests. gRPC multiplexes many RPCs over a **single long-lived HTTP/2 connection**. Once the L4 balancer assigns a connection to a backend, ALL RPCs on that connection go to the same backend. If one client opens one connection, all its RPCs hit one server.

```
L4 load balancer (connection-level):
  Client --[1 TCP conn]--> LB --[1 TCP conn]--> Server A
                                                (100% of RPCs go here)
                                  Server B      (gets nothing)
                                  Server C      (gets nothing)

L7 load balancer (request-level):
  Client --[1 HTTP/2 conn]--> LB --[stream 1]--> Server A
                                  --[stream 3]--> Server B
                                  --[stream 5]--> Server C
```

**Fix**: Use an L7 load balancer that understands HTTP/2 and can route individual streams (Envoy, Linkerd, Istio). Alternatively, use client-side load balancing (gRPC has built-in support for `dns:///`, `xds:///` resolvers and pick_first, round_robin, etc. policies).

### Q: SSE vs WebSocket for LLM token streaming?

**SSE wins for LLM streaming** because:
- LLM inference is inherently unidirectional: one prompt in, many tokens out.
- SSE works through every HTTP proxy, CDN, and load balancer with zero special configuration.
- Automatic reconnection is built into the EventSource API.
- No upgrade handshake means simpler connection setup.
- Sufficient for the use case -- there is no need for the client to send data mid-stream.

WebSockets would be warranted if the application needed **full-duplex** communication (e.g., a collaborative editor or multiplayer game), but for request-response-stream patterns, SSE is simpler and more robust.

### Q: How does gRPC handle deadlines and cancellation?

gRPC propagates deadlines through the call chain:

```
Client (deadline: 5s)
  -> Service A (remaining: 4.8s, sets grpc-timeout: 4800m)
       -> Service B (remaining: 4.5s, sets grpc-timeout: 4500m)
            -> Service C (remaining: 4.2s)
```

If the client cancels or the deadline expires:
1. The client sends RST_STREAM on the HTTP/2 stream.
2. The server's context is cancelled.
3. The cancellation propagates to any downstream RPCs the server initiated with that context.

This prevents **cascading resource waste**: if the caller is gone, all downstream work is abandoned.

### Q: What happens when an HTTP/2 connection experiences a TCP packet loss?

All streams on that connection are blocked until the lost packet is retransmitted (TCP HOL blocking). This is the fundamental limitation of HTTP/2 and the primary motivation for HTTP/3 (QUIC), which runs over UDP and implements independent stream-level loss recovery.

In gRPC, a single dropped TCP packet can stall dozens of concurrent RPCs. This is especially problematic on lossy networks (mobile, cross-region). Mitigations:
- Use multiple gRPC connections (gRPC channel pooling).
- Deploy services in the same region to minimize packet loss.
- Consider gRPC over QUIC (experimental, not widely supported yet).

### Q: How do you debug gRPC connection issues in production?

1. **Check gRPC status codes in trailers**, not HTTP status codes. A `200 OK` HTTP response can contain `grpc-status: 14 (UNAVAILABLE)`.
2. **Enable gRPC channelz** (`grpc.EnableChannelz()` in Go) for per-channel and per-subchannel diagnostics.
3. **Check connection-level flow control**: `WINDOW_UPDATE` frames not being sent (slow consumer) will stall all streams.
4. **Look for GOAWAY frames**: The server is draining connections (during rolling deploys, for instance).
5. **Monitor `grpc.client.attempt.duration` and `grpc.client.call.duration`**: The difference reveals retry overhead.
6. **Verify keepalive settings**: If the client's keepalive interval is more aggressive than what the server allows (`GOAWAY` with `ENHANCE_YOUR_CALM`), the server will kill connections.

### Q: What is the gRPC keepalive mechanism?

gRPC uses HTTP/2 PING frames for keepalive, not TCP keepalive:

```go
// Client-side keepalive
conn, _ := grpc.Dial(addr,
    grpc.WithKeepaliveParams(keepalive.ClientParameters{
        Time:                10 * time.Second,  // send PING every 10s if idle
        Timeout:             5 * time.Second,   // wait 5s for PING ACK
        PermitWithoutStream: false,             // only ping if there are active RPCs
    }),
)

// Server-side enforcement
server := grpc.NewServer(
    grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
        MinTime:             5 * time.Second,   // minimum time between client PINGs
        PermitWithoutStream: false,             // reject pings with no active RPCs
    }),
)
```

If the client pings too aggressively, the server responds with GOAWAY + `ENHANCE_YOUR_CALM`. This is a common misconfiguration that causes intermittent connection resets.

### Q: How does HTTP/2 flow control differ from TCP flow control?

- **TCP flow control** (receive window): Prevents the sender from overwhelming the receiver's kernel buffer. Operates at the byte-stream level. Managed by the OS.
- **HTTP/2 flow control** (WINDOW_UPDATE): Prevents a fast HTTP/2 sender from overwhelming a slow application-level consumer. Operates per-stream and per-connection. Managed by the application (or HTTP/2 library).

They are **independent and layered**: HTTP/2 flow control sits on top of TCP flow control. A stream can be flow-controlled at the HTTP/2 level even though the TCP window is open, and vice versa.

## Related Notes

- [[notes/Networking/tcp-socket-internals|TCP Socket Internals]]
- [[notes/Networking/tls-1.3-handshake|TLS 1.3 Handshake Deep Dive]]
- [[notes/K8s/service-mesh-multi-cluster-and-advanced-patterns|Service Mesh, Multi-Cluster & Advanced Patterns]]
