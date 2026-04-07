---
title: "QUIC and MASQUE"
---

QUIC and MASQUE represent a fundamental rethinking of internet transport and proxying. QUIC unifies transport reliability and encryption into a single protocol running over UDP, solving TCP's ossification, head-of-line blocking, and handshake latency problems. MASQUE extends HTTP/3 to tunnel arbitrary UDP and IP traffic through QUIC connections, enabling proxy architectures like Apple's iCloud Private Relay. This note covers QUIC's design decisions, its handshake mechanics, connection migration via Connection IDs, independent stream multiplexing, and MASQUE's CONNECT-UDP and CONNECT-IP extensions.

## QUIC: transport and encryption unified in one protocol

QUIC (RFC 9000, May 2021) exists because TCP had become unevolvable. Three problems drove its creation: the **2-RTT handshake tax** (TCP's 3-way handshake plus TLS 1.3's handshake before any data flows), **head-of-line blocking** (a single lost TCP segment blocks all multiplexed HTTP/2 streams), and **ossification** (one-third of internet paths encounter middleboxes that modify TCP metadata, making protocol evolution effectively impossible).

### Built on UDP to escape the kernel

QUIC runs on **UDP** -- not because UDP provides useful semantics, but because UDP packets pass through NATs, require no kernel changes, and are already understood by middleboxes. QUIC reimplements reliability, congestion control, and flow control entirely in **user-space**, enabling iteration at application-update timescales rather than OS-upgrade cycles. It uses **monotonically increasing packet numbers** (never reused, even for retransmissions) instead of TCP's byte-sequence numbers, eliminating retransmission ambiguity. Congestion control is per-connection, and senders can unilaterally choose their algorithm (New Reno, CUBIC, BBR). Flow control operates at both the stream level (`MAX_STREAM_DATA` frames) and connection level (`MAX_DATA` frames).

QUIC packets use two header formats. **Long headers** (used during connection establishment) carry Version, Source and Destination Connection IDs, and type-specific fields. **Short headers** (used for application data after the handshake) carry only the Destination Connection ID -- no version, no source CID -- minimizing per-packet overhead. A single UDP datagram can contain multiple coalesced QUIC packets.

### The 1-RTT and 0-RTT handshakes

QUIC merges the transport and TLS 1.3 cryptographic handshakes into a **single round trip**. The client sends an Initial packet containing a CRYPTO frame with the TLS ClientHello (including QUIC transport parameters as a TLS extension), padded to 1200 bytes to mitigate amplification attacks. The server responds with an Initial packet (ServerHello) and Handshake packets (EncryptedExtensions, Certificate, CertificateVerify, Finished) -- often coalesced into a single UDP datagram. The client completes with its Finished message and can immediately send **1-RTT application data**. Total: **1 RTT before data**, versus 2-3 RTTs for TCP+TLS.

For returning clients, QUIC supports **0-RTT**: the client uses a Pre-Shared Key from a previous session's NewSessionTicket to derive early encryption keys, sending application data in 0-RTT packets coalesced with the Initial ClientHello in the very first UDP datagram. The server can decrypt this data immediately. The tradeoff: 0-RTT data has **no replay protection** (an attacker can capture and replay it) and **no forward secrecy** (keys derive only from the PSK, not a fresh ECDHE exchange). Only idempotent operations should use 0-RTT. Google's deployment data showed the majority of QUIC connections complete with 0-RTT in practice.

### Connection IDs enable seamless IP changes

TCP identifies connections by the 4-tuple (source IP, source port, destination IP, destination port). Any element changing -- as happens when a phone switches from WiFi to cellular -- kills the connection. QUIC decouples connection identity from network addressing using **Connection IDs (CIDs)**: opaque, variable-length identifiers that each endpoint selects for its peer to use. Multiple CIDs can identify the same connection.

When a client's IP changes, it sends packets from the new address using a **fresh, previously unused Connection ID** (provided earlier via `NEW_CONNECTION_ID` frames). The server recognizes the CID and associates the packets with the existing connection. **Path validation** follows: the migrating endpoint sends a `PATH_CHALLENGE` frame containing 8 random bytes; the peer echoes them in `PATH_RESPONSE`. After validation, the endpoint resets its congestion controller and RTT estimates for the new path. Old CIDs are retired via `RETIRE_CONNECTION_ID` frames, preventing on-path observers from linking old and new network paths.

This is transformative for mobile: a user on a train moving between cell towers, or walking from WiFi to cellular, experiences **zero connection interruption**. Research from 2024 found approximately 50% of IPv4 QUIC servers and nearly 80% of IPv6 servers support connection migration.

### Independent streams eliminate head-of-line blocking

Each QUIC stream is an independent, ordered byte sequence. A lost packet affecting Stream 3 does **not** block delivery of data on Streams 1, 2, or 4 -- QUIC reassembles per-stream using Stream IDs and byte offsets. Stream IDs encode initiator (client vs. server) and directionality (bidirectional vs. unidirectional) in their two least-significant bits. Up to **2^62 streams** can be created within a connection.

This directly solves HTTP/2's fatal flaw: with HTTP/2 over TCP, all multiplexed streams share a single TCP byte stream, so one lost segment blocks everything. With HTTP/3 over QUIC, each HTTP request-response pair maps to its own QUIC stream -- loss isolation is built into the transport.

### What QUIC hides from middleboxes

After the Initial packets, **all QUIC traffic is encrypted with keys known only to the endpoints**. QUIC encrypts packet payloads using AEAD (AES-128-GCM, AES-256-GCM, or ChaCha20-Poly1305) and applies **header protection** -- a separate encryption step that masks the packet number and certain flag bits using a sample of the encrypted payload. Middleboxes cannot read frame types, stream IDs, acknowledgment information, or flow control state. The only elements guaranteed visible across all QUIC versions (per the **Invariants specification, RFC 8999**) are the header form bit, Version field, and Connection IDs. QUIC Version 2 (RFC 9369) was created explicitly to exercise these invariants, using different salt values and packet type bits to prevent middlebox ossification on v1's wire image.

**HTTP/3 (RFC 9114)** maps HTTP semantics onto QUIC streams: each request-response uses a client-initiated bidirectional stream, with dedicated unidirectional streams for control, QPACK encoder, and QPACK decoder. QPACK replaces HTTP/2's HPACK header compression because HPACK requires in-order delivery that QUIC's independent streams don't guarantee -- QPACK instead sends dynamic table updates on a separate encoder stream and only references entries after they have been acknowledged.

## MASQUE: tunneling protocols through HTTP/3

**MASQUE (Multiplexed Application Substrate over QUIC Encryption)** is the IETF protocol suite that lets HTTP/3 proxies tunnel arbitrary UDP and IP traffic -- the mechanism Apple chose for [[notes/Networking/icloud-private-relay-architecture|Private Relay's]] web traffic.

### CONNECT-UDP proxies QUIC through QUIC

RFC 9298 (August 2022, David Schinazi) extends the HTTP CONNECT method to proxy UDP flows. The client sends an Extended CONNECT request with `:protocol = connect-udp` and a URI template specifying the target host and port. If the proxy succeeds in opening a UDP socket to the target, it responds with 2xx, and subsequent data flows as **QUIC DATAGRAM frames** -- acknowledged but **not retransmitted** on loss. This unreliable delivery is critical: tunneling QUIC inside reliable TCP would cause head-of-line blocking and congestion control interference.

CONNECT-UDP enables **QUIC-in-QUIC**: an inner QUIC connection to a destination server runs over DATAGRAM frames of an outer QUIC connection to the proxy. Multiple independent UDP tunnels can be multiplexed over a single QUIC connection.

### CONNECT-IP provides VPN-like capability

RFC 9484 (October 2023, co-authored by Tommy Pauly of Apple) goes further, proxying **arbitrary IP packets** through HTTP. Using `:protocol = connect-ip`, the proxy and client negotiate IP address assignment (`ADDRESS_ASSIGN` capsules) and route advertisement (`ROUTE_ADVERTISEMENT` capsules). This enables full VPN-like functionality -- tunneling ICMP, IPsec, or any IP protocol -- entirely within the HTTP/3 framework.

---

## Related notes

- [[notes/Networking/icloud-private-relay-architecture|iCloud Private Relay Architecture]] -- how MASQUE tunnels compose with ODoH and blind signatures to create the two-hop relay system
- [[notes/Networking/cellular-networks-and-ip-addresses|Cellular Networks and IP Addresses]] -- the IP changes that QUIC's Connection IDs are designed to survive
- [[notes/Networking/dns-privacy-and-oblivious-protocols|DNS Privacy and Oblivious Protocols]] -- OHTTP/ODoH handle DNS while MASQUE handles streaming web traffic
