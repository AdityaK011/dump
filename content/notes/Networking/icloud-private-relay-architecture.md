---
title: "iCloud Private Relay Architecture"
---

iCloud Private Relay splits internet traffic across two non-colluding proxies so that no single entity -- not Apple, not the CDN partner, not your ISP -- can see both who you are and what you are browsing. This note covers the complete end-to-end flow: anonymous authentication via RSA blind signatures, the two-hop relay architecture with its privacy partitioning guarantees, the nested QUIC/MASQUE tunnel layers, what Private Relay does and does not protect, how LLM streaming via Server-Sent Events works through the relay, and why cookie-based sessions survive the IP address changes that the architecture necessarily introduces.

## The complete Private Relay flow, hop by hop

### Configuration and anonymous authentication

The device resolves `mask.icloud.com` (for QUIC/HTTP/3) or `mask-h2.icloud.com` (TCP/HTTP/2 fallback) via standard DNS -- notably, this initial lookup is in plaintext, which is how networks can detect and block Private Relay. APNIC researchers found approximately **1,725 ingress IP addresses** across Apple's own AS (AS714) and a dedicated Akamai-PR AS.

Authentication uses **RSA Blind Signatures** -- a Privacy Pass variant (RFC 9474). The device contacts Apple's Basic Attestation Authority, which verifies both hardware genuineness and iCloud+ subscription status, then issues one-time-use tokens. The "blind" property means tokens **cannot be linked back** to the requesting device. Tokens are rotated daily, rate-limited per device, and subject to asynchronous double-spend prevention. When connecting to a relay, the device presents these anonymous tokens -- the relay confirms legitimacy without learning identity.

### Two hops, two operators, zero overlap

The **ingress proxy** (Apple-operated) accepts the device's [[notes/Networking/quic-and-masque|QUIC]] connection on UDP port 443 with TLS 1.3, using **raw public keys** (not certificates) for proxy authentication. The device presents its blind signature token. The ingress proxy knows the user's **real IP address** and subscription validity -- but cannot see the destination because all subsequent traffic is encrypted to the egress proxy.

The ingress proxy performs a geo-IP lookup on the user's real IP, converts it to a **4-character geohash** (representing roughly 800 km-squared), and shares this coarse location -- not the real IP -- with the next hop.

The **egress proxy** (operated by Cloudflare, Akamai, or Fastly) receives traffic from the ingress proxy via a [[notes/Networking/quic-and-masque|MASQUE]] tunnel. It knows the **destination server** (decrypted from the CONNECT request) and the coarse geographic region -- but has no knowledge of the user's real IP. It assigns a **Relay IP address** from a dedicated pool mapped to the user's region (with city-level precision if "Maintain general location" is selected, or country+timezone if broader privacy is preferred). These Relay IPs are published at `mask-api.icloud.com/egress-ip-ranges.csv` and registered with geo-IP providers like MaxMind.

The privacy partitioning is summarized by what each entity can observe:

| Entity                | Client IP | Destination | Content |
|-----------------------|-----------|-------------|---------|
| ISP / Access Network  | Yes       | No          | No      |
| Ingress Proxy (Apple) | Yes       | No          | No      |
| Egress Proxy (CDN)    | No        | Yes         | No      |
| Destination Server    | No        | Yes         | Yes     |

### The tunnel-within-tunnel architecture

Three distinct connection layers nest inside each other:

**Layer 1 -- Client to Ingress Proxy**: A QUIC/HTTP/3 connection carrying the anonymous token and all multiplexed tunnels. SNI is `mask.icloud.com`; ALPN is HTTP/3.

**Layer 2 -- Client to Egress Proxy** (tunneled through Layer 1): A second QUIC connection runs inside a CONNECT-UDP tunnel through the ingress proxy. The TLS handshake is end-to-end between client and egress -- the ingress proxy sees only opaque DATAGRAM frames.

**Layer 3 -- Client to Destination Server** (tunneled through Layer 2): A CONNECT or CONNECT-UDP tunnel through the egress proxy carries the actual connection to the website. The TLS session with the destination server is **fully end-to-end** -- neither proxy can decrypt web content. As an optimization, the initial TLS handshake messages are sent **in the same data flight** as the proxy CONNECT request, eliminating extra round trips.

For fallback when QUIC is blocked, Private Relay uses HTTP/2 CONNECT over TLS/TCP to `mask-h2.icloud.com`, preserving the same dual-hop privacy architecture with reduced performance.

### What Private Relay protects and what it does not

Private Relay covers **all Safari browsing** (HTTP and HTTPS), **all DNS queries** (via [[notes/Networking/dns-privacy-and-oblivious-protocols|ODoH]]), and **insecure HTTP traffic from apps**. It does **not** cover non-Safari app HTTPS traffic, local network connections, VPN-routed traffic, cellular services (MMS, Visual Voicemail), or enterprise-managed devices. Content filters and parental controls using the NetworkExtension or Screen Time APIs still see traffic before it enters Private Relay. Starting with iOS 17, Apple exposed a **ProxyConfiguration API** allowing developers to configure their own MASQUE relays, and enterprise networks can deploy managed relay configurations via MDM as a VPN alternative.

## How LLM streaming works through all of this

When you use ChatGPT or Claude through Safari with Private Relay active, the token-by-token streaming response traverses both relay hops.

### Server-Sent Events carry tokens one by one

LLM services stream responses using **Server-Sent Events (SSE)** -- a server push mechanism defined in the WHATWG HTML Living Standard. The client sends a single HTTP POST with the prompt; the server responds with `Content-Type: text/event-stream` and holds the connection open, pushing each generated token as a plaintext event:

```
data: {"choices": [{"delta": {"content": "Hello"}, "index": 0}]}

data: {"choices": [{"delta": {"content": " world"}, "index": 0}]}

data: [DONE]
```

Each event uses the `data:` field for payload, `id:` for a resumable event identifier, `event:` for named event types, and `retry:` to control reconnection timing. Messages are separated by blank lines. SSE is strictly **unidirectional** (server to client) -- which is exactly what LLM streaming needs, since the prompt is sent once and only the response streams back.

SSE is preferred over WebSockets for LLM streaming because it works through HTTP proxies and CDNs without special configuration, requires no protocol upgrade handshake, and provides **built-in reconnection**: when the connection drops, the browser automatically reopens it and sends a `Last-Event-ID` header so the server can resume from the interruption point.

**WebSockets (RFC 6455)**, by contrast, require an HTTP Upgrade handshake to establish a full-duplex bidirectional channel with its own frame format (opcodes for text, binary, ping, pong, close). WebSockets excel for chat and gaming but add unnecessary complexity for the fundamentally unidirectional task of LLM response streaming. They also require application-managed reconnection logic and can cause difficulties with load balancers that expect standard HTTP traffic.

### Surviving IP changes during a stream

When your phone switches from WiFi to cellular mid-stream, the outcome depends on the transport layer. With **QUIC/HTTP/3**, [[notes/Networking/quic-and-masque|connection migration via Connection IDs]] can keep the underlying connection alive transparently -- the stream continues without interruption. With **TCP-based connections**, the connection breaks. SSE's built-in reconnection kicks in: the browser waits the `retry:` interval, reopens the connection, sends `Last-Event-ID`, and the server resumes. WebSocket applications must implement equivalent logic manually.

Crucially, the application-layer session survives regardless of [[notes/Networking/cellular-networks-and-ip-addresses|IP changes]] because **sessions are identified by cookies and tokens, not by IP addresses**.

## Cookies and tokens: why IP changes don't log you out

HTTP is **stateless by design** (RFC 6265) -- each request is independent, and the server retains no memory of previous requests. The **cookie mechanism** adds statefulness: the server sends a `Set-Cookie` header with a name-value pair and attributes, and the browser **automatically** includes matching cookies in every subsequent request via the `Cookie` header.

Cookie attributes control scope and security: `Domain` and `Path` determine which requests receive the cookie, `Secure` restricts to HTTPS, `HttpOnly` blocks JavaScript access (mitigating XSS), `SameSite` controls cross-site sending (Lax by default in modern browsers, mitigating CSRF), and `Max-Age`/`Expires` set persistence. Cookies without expiration attributes are **session cookies**, deleted when the browser closes.

For API authentication, **bearer tokens** in the `Authorization` header serve a similar purpose but must be explicitly attached by application code (unlike cookies, which are automatic). **JWTs (JSON Web Tokens, RFC 7519)** are self-contained tokens structured as `header.payload.signature` -- three base64url-encoded segments containing algorithm metadata, claims (issuer, subject, expiration, custom data), and a cryptographic signature (HMAC, RSA, or ECDSA). JWTs enable **stateless session management** because the server can verify the token's signature without a database lookup, though revocation before expiration requires maintaining a server-side blacklist.

**The fundamental insight for Private Relay**: session identity is scoped by domain, path, and scheme -- **never by IP address**. When your IP changes (WiFi to cellular, or when Private Relay rotates the egress Relay IP), the browser establishes new TCP/QUIC connections from the new IP but continues sending the same cookies. The server identifies you by your session token, not your source address. This is why you remain logged into every website despite Private Relay's egress IP being different from your real IP -- and why IP changes from network transitions do not trigger logouts.

Some security-conscious services implement optional **IP-based session binding** (rejecting cookies from IPs different from the one that created the session) as a session hijacking countermeasure. This is not default behavior and can break legitimate mobile usage where IPs change frequently.

---

## Related notes

- [[notes/Networking/quic-and-masque|QUIC and MASQUE]] -- the transport and tunneling protocols that form the backbone of Private Relay's nested tunnel architecture
- [[notes/Networking/dns-privacy-and-oblivious-protocols|DNS Privacy and Oblivious Protocols]] -- ODoH and OHTTP handle Private Relay's DNS resolution layer
- [[notes/Networking/cellular-networks-and-ip-addresses|Cellular Networks and IP Addresses]] -- how the phone acquires and maintains the IP address that Private Relay hides
