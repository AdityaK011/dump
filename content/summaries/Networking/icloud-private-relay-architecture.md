---
title: "Summary: iCloud Private Relay Architecture"
---

> **Full notes:** [[notes/Networking/icloud-private-relay-architecture|iCloud Private Relay Architecture -->]]

## Key Concepts

### Anonymous Authentication (RSA Blind Signatures)
- Device contacts Apple's Basic Attestation Authority, which verifies hardware genuineness + iCloud+ subscription
- Issues one-time-use Privacy Pass tokens (RFC 9474) that cannot be linked back to the requesting device
- Tokens rotated daily, rate-limited per device, double-spend prevention
- Relay confirms token legitimacy without learning identity

### Two-Hop Privacy Partitioning
- **Ingress proxy** (Apple-operated): knows user's real IP + subscription validity, cannot see destination
- Performs geo-IP lookup, converts real IP to 4-character geohash (~800 km^2), shares only this with egress
- **Egress proxy** (Cloudflare/Akamai/Fastly): knows destination server + coarse geo, cannot see real IP
- Assigns Relay IP from dedicated regional pool (city-level or country+timezone precision)
- Relay IPs published at `mask-api.icloud.com/egress-ip-ranges.csv`
- ~1,725 ingress IPs across AS714 and Akamai-PR AS

### Privacy Matrix
- ISP/Access Network: sees client IP only
- Ingress Proxy (Apple): sees client IP only
- Egress Proxy (CDN): sees destination only
- Destination Server: sees destination + content
- No single entity sees both client IP and destination

### Nested Tunnel Architecture (3 Layers)
- **Layer 1** (Client <-> Ingress): QUIC/HTTP/3 to `mask.icloud.com`, SNI visible, carries anonymous token
- **Layer 2** (Client <-> Egress): QUIC inside CONNECT-UDP tunnel through ingress. Ingress sees only opaque DATAGRAMs
- **Layer 3** (Client <-> Destination): CONNECT/CONNECT-UDP through egress. TLS fully end-to-end
- Fallback: HTTP/2 CONNECT over TLS/TCP to `mask-h2.icloud.com`

### Coverage Scope
- **Covered**: all Safari browsing (HTTP+HTTPS), all DNS queries (via ODoH), insecure HTTP from apps
- **Not covered**: non-Safari app HTTPS, local network, VPN traffic, cellular services (MMS, Visual Voicemail), enterprise-managed devices
- Content filters / Screen Time see traffic before Private Relay
- iOS 17+: ProxyConfiguration API for custom MASQUE relays; MDM-deployed relay configs

### LLM Streaming Through Private Relay
- Uses Server-Sent Events (SSE): `Content-Type: text/event-stream`, server pushes `data:` lines separated by blank lines
- SSE preferred over WebSockets for LLM streaming: works through HTTP proxies, no upgrade handshake, built-in reconnection via `Last-Event-ID`
- QUIC connection migration keeps stream alive across IP changes
- TCP fallback: SSE auto-reconnects with `Last-Event-ID`

### Session Survival Across IP Changes
- Sessions identified by cookies/tokens, never by IP address
- Cookie attributes: Domain, Path, Secure, HttpOnly, SameSite (Lax default), Max-Age/Expires
- JWTs (RFC 7519): `header.payload.signature`, stateless verification via signature check
- IP-based session binding exists but is optional and breaks mobile usage patterns

## Quick Reference

```
Detection/Blocking:
  Initial DNS lookup for mask.icloud.com is plaintext
  -> Networks can detect and block Private Relay here

Three-Layer Tunnel:
  +------------------------------------------------------+
  | Layer 1: Client <-> Ingress (QUIC, mask.icloud.com)  |
  |  +------------------------------------------------+  |
  |  | Layer 2: Client <-> Egress (QUIC in CONNECT-UDP)|  |
  |  |  +--------------------------------------------+ |  |
  |  |  | Layer 3: Client <-> Dest (TLS end-to-end)  | |  |
  |  |  +--------------------------------------------+ |  |
  |  +------------------------------------------------+  |
  +------------------------------------------------------+

Protocol Split:
  DNS queries   -> ODoH (OHTTP-based, single req/resp)
  Web traffic   -> MASQUE (CONNECT-UDP, streaming)

Key Numbers:
  Ingress IPs:          ~1,725
  Geohash precision:    4 chars (~800 km^2)
  Token rotation:       daily
  Egress operators:     Cloudflare, Akamai, Fastly

Session Identity:
  IP changes -> new TCP/QUIC connections from new IP
             -> same cookies sent automatically
             -> server identifies by session token, not IP
             -> user stays logged in
```
