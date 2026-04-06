# Inside iCloud Private Relay: a complete networking deep dive

**Apple’s iCloud Private Relay splits internet traffic across two non-colluding proxies so that no single entity — not Apple, not the CDN partner, not your ISP — can see both who you are and what you’re browsing.**  This architectural choice reverberates through every layer of the networking stack, from cell tower radio identifiers to HPKE-encrypted DNS queries to QUIC connection migration on a moving train. Understanding Private Relay requires understanding the full vertical: how your phone gets an IP address, why that address stays stable across tower handoffs, how DNS evolved from plaintext to cryptographically partitioned, how QUIC and MASQUE create tunnels within tunnels, and why you stay logged into ChatGPT through all of it. This report traces that complete path, protocol by protocol, from radio waves to session cookies.

-----

## How your phone gets — and keeps — an IP address

Before any privacy relay enters the picture, a phone must attach to a cellular network. This process involves a cascade of identifiers, tunnels, and anchor points that most users never see but that fundamentally determine when IP addresses change.

### From radio waves to the core network

When a phone powers on, it performs **cell search**: scanning for Primary and Secondary Synchronization Signals (PSS/SSS) broadcast by nearby cell towers, then decoding the Master Information Block (MIB)  and System Information Block 1 (SIB1) to learn the network’s identity and access parameters. It then executes a **4-step Random Access (RACH) procedure** — transmitting a random preamble on the Physical Random Access Channel, receiving a timing advance correction and temporary radio identifier (TC-RNTI), sending an RRC connection request, and completing contention resolution. The result is an RRC (Radio Resource Control) connection with the base station: an eNodeB in 4G LTE or a gNB in 5G NR.

The base station then forwards the phone’s NAS (Non-Access Stratum) registration message into the **core network**. In 4G, this means the eNodeB sends an Attach Request to the MME (Mobility Management Entity) via the S1-MME interface, which authenticates the subscriber through the HSS (Home Subscriber Server) using the Diameter protocol. In 5G, the gNB sends a Registration Request to the AMF (Access and Mobility Management Function) via the N2 interface, with authentication flowing through the AUSF and UDM. After authentication, the network establishes a data session — a **PDN connection** in 4G or a **PDU session** in 5G — and assigns the phone an IP address.

### Three identifiers and what they protect

The cellular network uses a layered identity system designed to minimize how often the phone’s permanent identity is exposed over the air:

**IMSI / SUPI** is the permanent subscriber identity — up to 15 digits structured as MCC (country) + MNC (operator) + MSIN (subscriber).  In 4G, the IMSI is transmitted in **plaintext** during the initial attach,  which is exactly what IMSI-catchers (Stingray devices) exploit. 5G fixes this: the IMSI is renamed SUPI (Subscription Permanent Identifier) and is **never sent in cleartext**. Instead, the phone transmits a SUCI (Subscription Concealed Identifier), encrypting the MSIN portion using ECIES with the home network’s public key.  Only the MCC+MNC are visible, for routing purposes.

**TMSI / GUTI** is the temporary identity used for all signaling after initial attach. In 4G, the GUTI (Globally Unique Temporary Identifier) consists of a GUMMEI (which identifies the specific MME) plus a 32-bit M-TMSI.  In 5G, the 5G-GUTI uses a GUAMI plus a 32-bit 5G-TMSI,  with the standard mandating unpredictable reallocation after each network-triggered service request to resist tracking.   The GUTI may or may not change during a handover — it depends on whether the phone enters a new Tracking Area.

**RNTI** (Radio Network Temporary Identifier) operates only at the radio layer, identifying the phone’s scheduling allocation within a single cell. The **C-RNTI** (Cell-RNTI) is a 16-bit identifier assigned by the base station  and used to scramble downlink control information for that specific phone. Critically, **C-RNTI always changes during a handover** because it has meaning only within a single cell. 

### GTP tunnels: the reason your IP survives tower hops

The mechanism that keeps your IP address stable across cell towers is **GTP (GPRS Tunnelling Protocol)**. User traffic flows through GTP-U tunnels (UDP port 2152) that encapsulate the phone’s original IP packets inside outer IP/UDP/GTP headers. In 4G, the data path is:

```
UE → [air] → eNodeB → [S1-U GTP tunnel] → S-GW → [S5 GTP tunnel] → P-GW → Internet
```

Each tunnel segment uses a **TEID (Tunnel Endpoint Identifier)** — a 32-bit dynamically allocated value that identifies the tunnel at the receiving node. Separate TEIDs exist for uplink and downlink on each interface.  The phone’s IP address is the **inner** IP inside the GTP encapsulation; the **outer** IPs belong to the network nodes. During a handover, only the outer tunnel endpoints change — the target base station allocates a new S1-U TEID, the MME sends a Modify Bearer Request to the S-GW, and the S-GW updates its forwarding table. The **P-GW remains completely unaware** of the handover and continues anchoring the phone’s IP address.

In 5G, the architecture is simplified: the S-GW is eliminated, and the **UPF (User Plane Function)** combines both gateway roles. Traffic flows UE → gNB → [N3 GTP-U tunnel] → UPF → Internet. The SMF (Session Management Function) programs the UPF via PFCP (Packet Forwarding Control Protocol) with Packet Detection Rules and Forwarding Action Rules. During handover, the SMF updates these rules to point at the new gNB — but the UPF’s N6 interface to the internet is untouched.

### When IP addresses actually change

Your IP stays stable during tower handoffs because the PDN connection (4G) or PDU session (5G) persists. IP addresses change when that session is disrupted:

- **Wi-Fi ↔ cellular transitions** assign completely different IPs because Wi-Fi uses local DHCP while cellular IPs come from the carrier’s P-GW/UPF pool  (exception: 3GPP’s ePDG and N3IWF mechanisms can maintain the same session across access types using IPSec tunnels)
- **Airplane mode toggling** releases the PDN/PDU session entirely, yielding a new IP on reconnect 
- **International roaming** with home-routed traffic tunnels data back to the home P-GW via the S8 interface, preserving the home-country IP;  local breakout instead routes through the visited network’s P-GW, producing a new IP
- **5G SSC Mode 2 and 3** intentionally change IPs: Mode 2 (“break-before-make”) releases the session for UPF reselection; Mode 3 (“make-before-break”) establishes a new UPF connection with a new IPv6 prefix before releasing the old one

Most carriers also deploy **Carrier-Grade NAT (CGNAT)**, assigning private IPv4 addresses  (10.x.x.x or 100.64.x.x) that map to shared public IPs. The public IP can change independently of the phone’s private address. 

-----

## DNS resolution evolved from plaintext to cryptographically partitioned

Traditional DNS, specified in RFC 1035 (1987), sends queries as **unencrypted UDP datagrams on port 53**. Every entity on the network path — your ISP, the coffee shop WiFi operator, a government firewall — can see every domain you resolve. This isn’t a theoretical concern: ISPs monetize DNS logs, censorship regimes inject forged responses, and IMSI-catcher-level attacks on WiFi can trivially capture DNS traffic with tcpdump.

### DoT and DoH encrypted the pipe but not the endpoint

**DNS over TLS (DoT, RFC 7858, 2016)** wraps DNS in TLS on dedicated **port 853**. This encrypts queries in transit but has two problems: port 853 is trivially identifiable and blockable by firewalls,  and the resolver still sees both the client’s IP and every query. **DNS over HTTPS (DoH, RFC 8484, 2018)** improved on this by sending DNS queries as HTTPS POST or GET requests on **port 443**, making them indistinguishable from normal web traffic.  The query payload uses the `application/dns-message` MIME type — the same binary wire format from RFC 1035, just tunneled inside HTTPS.  DoH blends with regular traffic and is much harder to block. 

But both DoT and DoH leave a fundamental gap: **the resolver still knows both who is asking (client IP) and what they’re asking (domain name)**. With DNS resolution increasingly centralized among a few large providers, this creates a single point of surveillance.

### ODoH splits identity from query using HPKE

**Oblivious DNS over HTTPS (ODoH, RFC 9230, 2022)** — co-authored by engineers from Apple, Cloudflare, and Fastly  — provides a **technical guarantee** that no single entity sees both pieces of information. The architecture introduces a three-party model: **Client → Relay → Target Resolver**. The client encrypts its DNS query using the Target’s public key, then sends the opaque ciphertext through the Relay. The Relay knows the client’s IP but cannot decrypt the query. The Target decrypts and resolves the query but only sees the Relay’s IP. 

The encryption uses **HPKE (Hybrid Public Key Encryption, RFC 9180)**, which composes three cryptographic primitives: 

- **KEM (Key Encapsulation Mechanism)**: DHKEM(X25519, HKDF-SHA256)  — an elliptic curve Diffie-Hellman key exchange using Curve25519.  The client generates an ephemeral keypair, performs DH with the Target’s public key, and outputs an encapsulated key `enc` plus a shared secret.
- **KDF (Key Derivation Function)**: HKDF-SHA256 — derives encryption keys from the shared secret using Extract and Expand operations (RFC 5869). 
- **AEAD (Authenticated Encryption with Associated Data)**: AES-128-GCM — encrypts the DNS query payload and authenticates it against tampering.

The client calls `SetupBaseS(pkR, "odoh query")` to establish an HPKE sender context, then `context.Seal(aad, plaintext)` to encrypt.  The resulting blob — `enc || ciphertext` — is sent as an `ObliviousDoHMessage` with content type `application/oblivious-dns-message`. The Target decrypts using `SetupBaseR(enc, skR, "odoh query")`  and resolves the query. The response is encrypted back using a symmetric key derived from the HPKE context via `context.Export("odoh response", Nk)`, bound to the original query plaintext and a fresh random nonce — ensuring only the originating client can decrypt it.

**Apple’s Private Relay uses ODoH for all DNS resolution.**  The device encrypts each DNS query with HPKE, sends it through Apple’s ingress relay (which cannot read it), and the query reaches the DNS resolver (Cloudflare’s `odoh.cloudflare-dns.com`) stripped of the user’s identity.  To ensure geographically correct answers, the device includes its public IP subnet in the encrypted query via the EDNS0 Client Subnet option.

-----

## OHTTP: the protocol that makes oblivious requests possible

**Oblivious HTTP (RFC 9458, January 2024)**  generalizes the ODoH pattern to arbitrary HTTP requests. Its three roles — Client, Relay, and Gateway — mirror ODoH’s architecture but work for any HTTP exchange, not just DNS. 

### Encryption and encapsulation in detail

The client first obtains the Gateway’s **KeyConfig** (served as `application/ohttp-keys`),  which contains a Key Identifier, HPKE KEM/KDF/AEAD algorithm IDs, and the Gateway’s public key.  The client then serializes its HTTP request using **Binary HTTP (RFC 9292)** — a compact binary encoding of HTTP messages  using QUIC-style variable-length integers for all lengths, capturing method, scheme, authority, path, headers, body, and optional padding in a protocol-independent format. 

The encapsulated request is constructed by building a 7-byte header (`key_id || kem_id || kdf_id || aead_id`), calling `SetupBaseS(pkR, info)` to generate an HPKE sender context, then sealing the Binary HTTP payload with the header as AAD.  The result is `hdr || enc || ciphertext` — sent to the Relay as an HTTP POST with content type `application/ohttp-req`. The Relay forwards the opaque blob to the Gateway as `message/ohttp-req`. The Gateway decrypts using its private key, processes the request, and encrypts the response using a key derived from the HPKE context’s Export function, bound to the original query plaintext and a fresh nonce.  The encrypted response returns through the Relay as `message/ohttp-res` / `application/ohttp-res`.

### Why OHTTP only handles single request-response pairs

OHTTP is **stateless by design** — each request creates a fresh ephemeral HPKE context with new keying material. This is essential for **unlinkability**: if state persisted between requests, the Gateway could correlate them as coming from the same client.  But this makes OHTTP unsuitable for streaming, long-lived connections, or WebSocket-style communication. The entire Binary HTTP message must be encrypted as one unit before sending; there is no mechanism for incremental delivery. 

This limitation directly shapes Private Relay’s architecture: **OHTTP handles DNS queries** (which are inherently single request-response pairs), while **MASQUE handles web traffic** (which requires streaming, multiplexing, and persistent connections).

-----

## QUIC: transport and encryption unified in one protocol

QUIC (RFC 9000, May 2021)  exists because TCP had become unevolvable. Three problems drove its creation: the **2-RTT handshake tax** (TCP’s 3-way handshake plus TLS 1.3’s handshake before any data flows), **head-of-line blocking** (a single lost TCP segment blocks all multiplexed HTTP/2 streams), and **ossification** (one-third of internet paths encounter middleboxes that modify TCP metadata, making protocol evolution effectively impossible). 

### Built on UDP to escape the kernel

QUIC runs on **UDP**  — not because UDP provides useful semantics, but because UDP packets pass through NATs, require no kernel changes, and are already understood by middleboxes. QUIC reimplements reliability, congestion control, and flow control entirely in **user-space**, enabling iteration at application-update timescales rather than OS-upgrade cycles.  It uses **monotonically increasing packet numbers** (never reused, even for retransmissions) instead of TCP’s byte-sequence numbers, eliminating retransmission ambiguity. Congestion control is per-connection, and senders can unilaterally choose their algorithm (New Reno, CUBIC, BBR). Flow control operates at both the stream level (`MAX_STREAM_DATA` frames) and connection level (`MAX_DATA` frames). 

QUIC packets use two header formats. **Long headers** (used during connection establishment) carry Version, Source and Destination Connection IDs, and type-specific fields. **Short headers** (used for application data after the handshake) carry only the Destination Connection ID — no version, no source CID — minimizing per-packet overhead.  A single UDP datagram can contain multiple coalesced QUIC packets.

### The 1-RTT and 0-RTT handshakes

QUIC merges the transport and TLS 1.3 cryptographic handshakes into a **single round trip**.  The client sends an Initial packet containing a CRYPTO frame with the TLS ClientHello (including QUIC transport parameters as a TLS extension), padded to 1200 bytes to mitigate amplification attacks. The server responds with an Initial packet (ServerHello) and Handshake packets (EncryptedExtensions, Certificate, CertificateVerify, Finished) — often coalesced into a single UDP datagram. The client completes with its Finished message and can immediately send **1-RTT application data**. Total: **1 RTT before data**, versus 2-3 RTTs for TCP+TLS.

For returning clients, QUIC supports **0-RTT**: the client uses a Pre-Shared Key from a previous session’s NewSessionTicket to derive early encryption keys, sending application data in 0-RTT packets coalesced with the Initial ClientHello in the very first UDP datagram.  The server can decrypt this data immediately. The tradeoff: 0-RTT data has **no replay protection** (an attacker can capture and replay it) and **no forward secrecy** (keys derive only from the PSK, not a fresh ECDHE exchange). Only idempotent operations should use 0-RTT. Google’s deployment data showed the majority of QUIC connections complete with 0-RTT in practice.

### Connection IDs enable seamless IP changes

TCP identifies connections by the 4-tuple (source IP, source port, destination IP, destination port). Any element changing — as happens when a phone switches from WiFi to cellular — kills the connection. QUIC decouples connection identity from network addressing using **Connection IDs (CIDs)**: opaque, variable-length identifiers that each endpoint selects for its peer to use.  Multiple CIDs can identify the same connection.

When a client’s IP changes, it sends packets from the new address using a **fresh, previously unused Connection ID** (provided earlier via `NEW_CONNECTION_ID` frames). The server recognizes the CID and associates the packets with the existing connection. **Path validation** follows: the migrating endpoint sends a `PATH_CHALLENGE` frame containing 8 random bytes; the peer echoes them in `PATH_RESPONSE`.  After validation, the endpoint resets its congestion controller and RTT estimates for the new path. Old CIDs are retired via `RETIRE_CONNECTION_ID` frames, preventing on-path observers from linking old and new network paths.

This is transformative for mobile: a user on a train moving between cell towers, or walking from WiFi to cellular, experiences **zero connection interruption**.  Research from 2024 found approximately 50% of IPv4 QUIC servers and nearly 80% of IPv6 servers support connection migration.

### Independent streams eliminate head-of-line blocking

Each QUIC stream is an independent, ordered byte sequence. A lost packet affecting Stream 3 does **not** block delivery of data on Streams 1, 2, or 4 — QUIC reassembles per-stream using Stream IDs and byte offsets. Stream IDs encode initiator (client vs. server) and directionality (bidirectional vs. unidirectional) in their two least-significant bits. Up to **2^62 streams** can be created within a connection.

This directly solves HTTP/2’s fatal flaw: with HTTP/2 over TCP, all multiplexed streams share a single TCP byte stream, so one lost segment blocks everything.   With HTTP/3 over QUIC, each HTTP request-response pair maps to its own QUIC stream — loss isolation is built into the transport.

### What QUIC hides from middleboxes

After the Initial packets, **all QUIC traffic is encrypted with keys known only to the endpoints**. QUIC encrypts packet payloads using AEAD (AES-128-GCM, AES-256-GCM, or ChaCha20-Poly1305) and applies **header protection** — a separate encryption step that masks the packet number and certain flag bits using a sample of the encrypted payload. Middleboxes cannot read frame types, stream IDs, acknowledgment information, or flow control state.  The only elements guaranteed visible across all QUIC versions  (per the **Invariants specification, RFC 8999**) are the header form bit, Version field, and Connection IDs. QUIC Version 2 (RFC 9369) was created explicitly to exercise these invariants, using different salt values and packet type bits to prevent middlebox ossification on v1’s wire image.

**HTTP/3 (RFC 9114)**  maps HTTP semantics onto QUIC streams: each request-response uses a client-initiated bidirectional stream,  with dedicated unidirectional streams for control,  QPACK encoder, and QPACK decoder. QPACK replaces HTTP/2’s HPACK header compression because HPACK requires in-order delivery that QUIC’s independent streams don’t guarantee — QPACK instead sends dynamic table updates on a separate encoder stream and only references entries after they’ve been acknowledged.

-----

## MASQUE: tunneling protocols through HTTP/3

**MASQUE (Multiplexed Application Substrate over QUIC Encryption)** is  the IETF protocol suite that lets HTTP/3 proxies tunnel arbitrary UDP and IP traffic   — the mechanism Apple chose for Private Relay’s web traffic.

### CONNECT-UDP proxies QUIC through QUIC

RFC 9298  (August 2022, David Schinazi) extends the HTTP CONNECT method to proxy UDP flows.  The client sends an Extended CONNECT request  with `:protocol = connect-udp` and a URI template specifying the target host and port.  If the proxy succeeds in opening a UDP socket to the target, it responds with 2xx, and subsequent data flows as **QUIC DATAGRAM frames** — acknowledged but **not retransmitted** on loss. This unreliable delivery is critical: tunneling QUIC inside reliable TCP would cause head-of-line blocking and congestion control interference.  

CONNECT-UDP enables **QUIC-in-QUIC**: an inner QUIC connection to a destination server runs over DATAGRAM frames of an outer QUIC connection to the proxy. Multiple independent UDP tunnels can be multiplexed over a single QUIC connection.

### CONNECT-IP provides VPN-like capability

RFC 9484 (October 2023,  co-authored by Tommy Pauly of Apple) goes further, proxying **arbitrary IP packets** through HTTP. Using `:protocol = connect-ip`,  the proxy and client negotiate IP address assignment (`ADDRESS_ASSIGN` capsules) and route advertisement (`ROUTE_ADVERTISEMENT` capsules). This enables full VPN-like functionality — tunneling ICMP, IPsec, or any IP protocol — entirely within the HTTP/3 framework. 

-----

## The complete Private Relay flow, hop by hop

With all the building blocks in place, here is how an iCloud Private Relay connection actually works end to end.

### Configuration and anonymous authentication

The device resolves `mask.icloud.com` (for QUIC/HTTP/3) or `mask-h2.icloud.com` (TCP/HTTP/2 fallback)  via standard DNS  — notably, this initial lookup is in plaintext, which is how networks can detect and block Private Relay.  APNIC researchers found approximately **1,725 ingress IP addresses** across Apple’s own AS (AS714) and a dedicated Akamai-PR AS. 

Authentication uses **RSA Blind Signatures** — a Privacy Pass variant  (RFC 9474). The device contacts Apple’s Basic Attestation Authority, which verifies both hardware genuineness and iCloud+ subscription status, then issues one-time-use tokens.  The “blind” property means tokens **cannot be linked back** to the requesting device. Tokens are rotated daily,  rate-limited per device, and subject to asynchronous double-spend prevention.  When connecting to a relay, the device presents these anonymous tokens — the relay confirms legitimacy without learning identity. 

### Two hops, two operators, zero overlap

The **ingress proxy** (Apple-operated) accepts the device’s QUIC connection on UDP port 443 with TLS 1.3,  using **raw public keys** (not certificates) for proxy authentication.  The device presents its blind signature token. The ingress proxy knows the user’s **real IP address** and subscription validity — but cannot see the destination because all subsequent traffic is encrypted to the egress proxy.  

The ingress proxy performs a geo-IP lookup on the user’s real IP, converts it to a **4-character geohash** (representing roughly 800 km²), and shares this coarse location — not the real IP — with the next hop.  

The **egress proxy** (operated by Cloudflare, Akamai, or Fastly)  receives traffic from the ingress proxy via a MASQUE tunnel.  It knows the **destination server** (decrypted from the CONNECT request) and the coarse geographic region — but has no knowledge of the user’s real IP.   It assigns a **Relay IP address** from a dedicated pool mapped to the user’s region  (with city-level precision if “Maintain general location” is selected, or country+timezone if broader privacy is preferred).  These Relay IPs are published at `mask-api.icloud.com/egress-ip-ranges.csv` and registered with geo-IP providers like MaxMind.

The privacy partitioning is summarized by what each entity can observe:

|Entity               |Client IP|Destination|Content|
|---------------------|---------|-----------|-------|
|ISP / Access Network |✅        |❌          |❌      |
|Ingress Proxy (Apple)|✅        |❌          |❌      |
|Egress Proxy (CDN)   |❌        |✅          |❌      |
|Destination Server   |❌        |✅          |✅      |

### The tunnel-within-tunnel architecture

Three distinct connection layers nest inside each other:

**Layer 1 — Client ↔ Ingress Proxy**: A QUIC/HTTP/3 connection carrying the anonymous token and all multiplexed tunnels. SNI is `mask.icloud.com`; ALPN is HTTP/3. 

**Layer 2 — Client ↔ Egress Proxy** (tunneled through Layer 1): A second QUIC connection runs inside a CONNECT-UDP tunnel through the ingress proxy. The TLS handshake is end-to-end between client and egress — the ingress proxy sees only opaque DATAGRAM frames.

**Layer 3 — Client ↔ Destination Server** (tunneled through Layer 2): A CONNECT or CONNECT-UDP tunnel through the egress proxy carries the actual connection to the website. The TLS session with the destination server is **fully end-to-end** — neither proxy can decrypt web content. As an optimization, the initial TLS handshake messages are sent **in the same data flight** as the proxy CONNECT request, eliminating extra round trips.  

For fallback when QUIC is blocked, Private Relay uses HTTP/2 CONNECT over TLS/TCP to `mask-h2.icloud.com`,  preserving the same dual-hop privacy architecture with reduced performance. 

### What Private Relay protects and what it doesn’t

Private Relay covers **all Safari browsing** (HTTP and HTTPS),  **all DNS queries** (via ODoH),  and **insecure HTTP traffic from apps**.  It does **not** cover non-Safari app HTTPS traffic, local network connections, VPN-routed traffic,  cellular services (MMS, Visual Voicemail), or enterprise-managed devices.   Content filters and parental controls using the NetworkExtension or Screen Time APIs still see traffic before it enters Private Relay.  Starting with iOS 17, Apple exposed a **ProxyConfiguration API** allowing developers to configure their own MASQUE relays, and enterprise networks can deploy managed relay configurations via MDM as a VPN alternative. 

-----

## How LLM streaming works through all of this

When you use ChatGPT or Claude through Safari with Private Relay active, the token-by-token streaming response traverses both relay hops. Understanding how this works — and survives network disruptions — requires understanding SSE and session management.

### Server-Sent Events carry tokens one by one

LLM services stream responses using **Server-Sent Events (SSE)** — a server push mechanism defined in the WHATWG HTML Living Standard.   The client sends a single HTTP POST with the prompt; the server responds with `Content-Type: text/event-stream`  and holds the connection open, pushing each generated token as a plaintext event:

```
data: {"choices": [{"delta": {"content": "Hello"}, "index": 0}]}

data: {"choices": [{"delta": {"content": " world"}, "index": 0}]}

data: [DONE]
```

Each event uses the `data:` field for payload, `id:` for a resumable event identifier, `event:` for named event types, and `retry:` to control reconnection timing. Messages are separated by blank lines.  SSE is strictly **unidirectional** (server → client) — which is exactly what LLM streaming needs, since the prompt is sent once and only the response streams back. 

SSE is preferred over WebSockets for LLM streaming because it works through HTTP proxies and CDNs without special configuration,  requires no protocol upgrade handshake, and provides **built-in reconnection**: when the connection drops, the browser automatically reopens it and sends a `Last-Event-ID` header so the server can resume from the interruption point. 

**WebSockets (RFC 6455)**, by contrast, require an HTTP Upgrade handshake to establish a full-duplex bidirectional channel  with its own frame format (opcodes for text, binary, ping, pong, close). WebSockets excel for chat and gaming but add unnecessary complexity for the fundamentally unidirectional task of LLM response streaming.  They also require application-managed reconnection logic and can cause difficulties with load balancers that expect standard HTTP traffic.

### Surviving IP changes during a stream

When your phone switches from WiFi to cellular mid-stream, the outcome depends on the transport layer. With **QUIC/HTTP/3**, connection migration via Connection IDs can keep the underlying connection alive transparently — the stream continues without interruption. With **TCP-based connections**, the connection breaks. SSE’s built-in reconnection kicks in: the browser waits the `retry:` interval, reopens the connection, sends `Last-Event-ID`, and the server resumes.  WebSocket applications must implement equivalent logic manually. 

Crucially, the application-layer session survives regardless of IP changes because **sessions are identified by cookies and tokens, not by IP addresses**.

-----

## Cookies and tokens: why IP changes don’t log you out

HTTP is **stateless by design** (RFC 6265 §1) — each request is independent, and the server retains no memory of previous requests. The **cookie mechanism** adds statefulness: the server sends a `Set-Cookie` header with a name-value pair and attributes, and the browser **automatically** includes matching cookies in every subsequent request via the `Cookie` header. 

Cookie attributes control scope and security:  ` [![](claude-citation:/icon.png?validation=7C74B72B-396D-42E2-A632-587E2D35653D&citation=eyJlbmRJbmRleCI6MjkxOTQsIm1ldGFkYXRhIjp7Imljb25VcmwiOiJodHRwczpcL1wvd3d3Lmdvb2dsZS5jb21cL3MyXC9mYXZpY29ucz9zej02NCZkb21haW49bW96aWxsYS5vcmciLCJwcmV2aWV3VGl0bGUiOiJTZXQtQ29va2llIGhlYWRlciAtIEhUVFAgLSBNRE4gV2ViIERvY3MiLCJzb3VyY2UiOiJNRE4gV2ViIERvY3MiLCJ0eXBlIjoiZ2VuZXJpY19tZXRhZGF0YSJ9LCJzb3VyY2VzIjpbeyJpY29uVXJsIjoiaHR0cHM6XC9cL3d3dy5nb29nbGUuY29tXC9zMlwvZmF2aWNvbnM/c3o9NjQmZG9tYWluPW1vemlsbGEub3JnIiwic291cmNlIjoiTUROIFdlYiBEb2NzIiwidGl0bGUiOiJTZXQtQ29va2llIGhlYWRlciAtIEhUVFAgLSBNRE4gV2ViIERvY3MiLCJ1cmwiOiJodHRwczpcL1wvZGV2ZWxvcGVyLm1vemlsbGEub3JnXC9lbi1VU1wvZG9jc1wvV2ViXC9IVFRQXC9SZWZlcmVuY2VcL0hlYWRlcnNcL1NldC1Db29raWUifV0sInN0YXJ0SW5kZXgiOjI5MTQ3LCJ0aXRsZSI6Ik1ETiBXZWIgRG9jcyIsInVybCI6Imh0dHBzOlwvXC9kZXZlbG9wZXIubW96aWxsYS5vcmdcL2VuLVVTXC9kb2NzXC9XZWJcL0hUVFBcL1JlZmVyZW5jZVwvSGVhZGVyc1wvU2V0LUNvb2tpZSIsInV1aWQiOiI3MjlhZjVkZi0xN2RmLTRkMGMtYmJiNi1iYWRmNzY5YWZkOWUifQ%3D%3D "MDN Web Docs")](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie)Domain` and `Path` determine which requests receive the cookie, `Secure` restricts to HTTPS, `HttpOnly` blocks JavaScript access (mitigating XSS), `SameSite` controls cross-site sending (Lax by default in modern browsers, mitigating CSRF), and `Max-Age`/`Expires` set persistence. Cookies without expiration attributes are **session cookies**, deleted when the browser closes.

For API authentication, **bearer tokens** in the `Authorization` header serve a similar purpose but must be explicitly attached by application code (unlike cookies, which are automatic). **JWTs (JSON Web Tokens, RFC 7519)** are self-contained tokens structured as `header.payload.signature` — three base64url-encoded segments containing algorithm metadata, claims (issuer, subject, expiration, custom data), and a cryptographic signature (HMAC, RSA, or ECDSA). JWTs enable **stateless session management** because the server can verify the token’s signature without a database lookup, though revocation before expiration requires maintaining a server-side blacklist.

**The fundamental insight for Private Relay**: session identity is scoped by domain, path, and scheme — **never by IP address**. When your IP changes (WiFi to cellular, or when Private Relay rotates the egress Relay IP), the browser establishes new TCP/QUIC connections from the new IP but continues sending the same cookies. The server identifies you by your session token, not your source address. This is why you remain logged into every website despite Private Relay’s egress IP being different from your real IP — and why IP changes from network transitions don’t trigger logouts.

Some security-conscious services implement optional **IP-based session binding** (rejecting cookies from IPs different from the one that created the session) as a session hijacking countermeasure. This is not default behavior and can break legitimate mobile usage where IPs change frequently.

-----

## Conclusion: privacy through architectural separation

iCloud Private Relay’s core innovation isn’t any single protocol — it’s the **architectural principle of privacy partitioning** applied across every layer. At the DNS layer, ODoH uses HPKE to ensure the relay that knows your identity can’t read your query, while the resolver that reads your query doesn’t know your identity. At the transport layer, MASQUE tunnels within QUIC connections create a nested architecture where the ingress proxy sees your IP but not your destination, and the egress proxy sees your destination but not your IP. At the application layer, cookies and tokens ensure sessions persist through the IP changes this architecture necessarily introduces.

The protocol stack enabling all of this — GTP tunnels anchoring cellular IPs, QUIC’s Connection IDs surviving network transitions, HPKE’s hybrid encryption splitting knowledge between parties, MASQUE’s HTTP/3 tunneling carrying arbitrary traffic, and SSE’s built-in reconnection resuming streams after disruptions — represents a convergence of a decade of IETF standardization work. The key RFCs are **9000** (QUIC), **9001** (QUIC-TLS), **9114** (HTTP/3), **9180** (HPKE), **9230** (ODoH), **9292** (Binary HTTP), **9297** (HTTP Datagrams), **9298** (CONNECT-UDP), **9458** (OHTTP), and **9484** (CONNECT-IP). What makes Private Relay novel is not the invention of new protocols but the composition of open standards into a system where trust is distributed by cryptographic design rather than policy promises.