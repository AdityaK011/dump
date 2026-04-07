---
title: "Summary: QUIC and MASQUE"
---

> **Full notes:** [[notes/Networking/quic-and-masque|QUIC and MASQUE -->]]

## Key Concepts

### Why QUIC Exists
- TCP's three problems: 2-RTT handshake tax, head-of-line blocking across multiplexed streams, ossification (1/3 of paths have middleboxes modifying TCP metadata)
- Runs on UDP to bypass kernel and middlebox constraints
- Reimplements reliability, congestion control, flow control entirely in user-space

### QUIC Transport Design
- **Packet numbers**: monotonically increasing, never reused (eliminates retransmission ambiguity)
- **Congestion control**: per-connection, sender chooses algorithm (New Reno, CUBIC, BBR)
- **Flow control**: stream-level (`MAX_STREAM_DATA`) + connection-level (`MAX_DATA`)
- **Long headers**: connection setup (Version, Src/Dst CIDs, type fields)
- **Short headers**: application data (Dst CID only, minimal overhead)
- Single UDP datagram can contain multiple coalesced QUIC packets

### Handshake Performance
- **1-RTT**: merges transport + TLS 1.3 handshake. Initial packet padded to 1200 bytes (amplification mitigation). 1 RTT vs TCP+TLS's 2-3 RTTs
- **0-RTT**: returning clients use PSK from previous session. Data sent alongside Initial ClientHello. Trade-offs: no replay protection, no forward secrecy. Only for idempotent operations. Majority of Google QUIC connections use 0-RTT

### Connection IDs -- Surviving IP Changes
- TCP uses 4-tuple (breaks on any element change). QUIC uses opaque Connection IDs
- On IP change: client uses fresh unused CID (pre-shared via `NEW_CONNECTION_ID` frames)
- Path validation: `PATH_CHALLENGE` (8 random bytes) -> `PATH_RESPONSE`
- Old CIDs retired via `RETIRE_CONNECTION_ID` to prevent linking old/new paths
- ~50% IPv4 servers and ~80% IPv6 servers support connection migration

### Independent Stream Multiplexing
- Each stream is independent ordered byte sequence -- loss on one stream does not block others
- Stream ID encodes: initiator (client/server) + directionality (bidi/unidi) in 2 LSBs
- Up to 2^62 streams per connection
- Directly solves HTTP/2-over-TCP head-of-line blocking

### Encryption and Anti-Ossification
- All post-handshake traffic encrypted (AES-128-GCM, AES-256-GCM, or ChaCha20-Poly1305)
- Header protection masks packet number and flag bits
- Invariants (RFC 8999): only header form bit, Version, and CIDs guaranteed visible
- QUIC v2 (RFC 9369): different salt/packet-type bits to prevent v1 ossification

### HTTP/3 (RFC 9114)
- Maps HTTP onto QUIC streams: each request-response = client-initiated bidi stream
- QPACK replaces HPACK (HPACK needs in-order delivery; QUIC streams are independent)
- Dedicated unidirectional streams for control, QPACK encoder, QPACK decoder

### MASQUE Tunneling
- **CONNECT-UDP** (RFC 9298): proxies UDP flows via Extended CONNECT with `:protocol = connect-udp`. Data as QUIC DATAGRAM frames (acknowledged, not retransmitted). Enables QUIC-in-QUIC
- **CONNECT-IP** (RFC 9484): proxies arbitrary IP packets via `:protocol = connect-ip`. Negotiates IP assignment (`ADDRESS_ASSIGN`) and routes (`ROUTE_ADVERTISEMENT`). Full VPN-like capability within HTTP/3

## Quick Reference

```
Handshake Comparison:
  TCP + TLS 1.3:  [SYN] [SYN-ACK] [ACK+ClientHello] [ServerHello] = 2-3 RTT
  QUIC 1-RTT:     [Initial(ClientHello)] [Initial+Handshake(Server)] = 1 RTT
  QUIC 0-RTT:     [Initial+0-RTT data] [Server response]            = 0 RTT

Connection Migration:
  Phone moves WiFi -> Cellular:
    TCP:  connection dies, full reconnect
    QUIC: new CID on new IP, PATH_CHALLENGE/RESPONSE, stream continues

MASQUE Stack:
  +---------------------------+
  | CONNECT-IP  (any IP)      |  RFC 9484
  | CONNECT-UDP (UDP flows)   |  RFC 9298
  +---------------------------+
  | HTTP/3                    |  RFC 9114
  +---------------------------+
  | QUIC                      |  RFC 9000
  +---------------------------+
  | UDP                       |
  +---------------------------+

Key Numbers:
  Initial packet min size:  1200 bytes
  Max streams/connection:   2^62
  PATH_CHALLENGE payload:   8 bytes
  GTP-U port:               2152
  QUIC DATAGRAM:            ack'd but NOT retransmitted
```
