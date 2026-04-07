---
title: "DNS Privacy and Oblivious Protocols"
---

DNS resolution has evolved from plaintext queries visible to every network observer to cryptographically partitioned systems where no single entity can see both who is asking and what they are asking. This note covers that evolution -- from traditional unencrypted DNS through DoT and DoH, to Oblivious DNS over HTTPS (ODoH) and Oblivious HTTP (OHTTP) -- with particular attention to the HPKE encryption mechanics and the architectural constraints that make these protocols suitable for single request-response exchanges but not for streaming.

## DNS resolution evolved from plaintext to cryptographically partitioned

Traditional DNS, specified in RFC 1035 (1987), sends queries as **unencrypted UDP datagrams on port 53**. Every entity on the network path -- your ISP, the coffee shop WiFi operator, a government firewall -- can see every domain you resolve. This is not a theoretical concern: ISPs monetize DNS logs, censorship regimes inject forged responses, and attacks on WiFi can trivially capture DNS traffic.

### DoT and DoH encrypted the pipe but not the endpoint

**DNS over TLS (DoT, RFC 7858, 2016)** wraps DNS in TLS on dedicated **port 853**. This encrypts queries in transit but has two problems: port 853 is trivially identifiable and blockable by firewalls, and the resolver still sees both the client's IP and every query. **DNS over HTTPS (DoH, RFC 8484, 2018)** improved on this by sending DNS queries as HTTPS POST or GET requests on **port 443**, making them indistinguishable from normal web traffic. The query payload uses the `application/dns-message` MIME type -- the same binary wire format from RFC 1035, just tunneled inside HTTPS. DoH blends with regular traffic and is much harder to block.

But both DoT and DoH leave a fundamental gap: **the resolver still knows both who is asking (client IP) and what they're asking (domain name)**. With DNS resolution increasingly centralized among a few large providers, this creates a single point of surveillance.

### ODoH splits identity from query using HPKE

**Oblivious DNS over HTTPS (ODoH, RFC 9230, 2022)** -- co-authored by engineers from Apple, Cloudflare, and Fastly -- provides a **technical guarantee** that no single entity sees both pieces of information. The architecture introduces a three-party model: **Client -> Relay -> Target Resolver**. The client encrypts its DNS query using the Target's public key, then sends the opaque ciphertext through the Relay. The Relay knows the client's IP but cannot decrypt the query. The Target decrypts and resolves the query but only sees the Relay's IP.

The encryption uses **HPKE (Hybrid Public Key Encryption, RFC 9180)**, which composes three cryptographic primitives:

- **KEM (Key Encapsulation Mechanism)**: DHKEM(X25519, HKDF-SHA256) -- an elliptic curve Diffie-Hellman key exchange using Curve25519. The client generates an ephemeral keypair, performs DH with the Target's public key, and outputs an encapsulated key `enc` plus a shared secret.
- **KDF (Key Derivation Function)**: HKDF-SHA256 -- derives encryption keys from the shared secret using Extract and Expand operations (RFC 5869).
- **AEAD (Authenticated Encryption with Associated Data)**: AES-128-GCM -- encrypts the DNS query payload and authenticates it against tampering.

The client calls `SetupBaseS(pkR, "odoh query")` to establish an HPKE sender context, then `context.Seal(aad, plaintext)` to encrypt. The resulting blob -- `enc || ciphertext` -- is sent as an `ObliviousDoHMessage` with content type `application/oblivious-dns-message`. The Target decrypts using `SetupBaseR(enc, skR, "odoh query")` and resolves the query. The response is encrypted back using a symmetric key derived from the HPKE context via `context.Export("odoh response", Nk)`, bound to the original query plaintext and a fresh random nonce -- ensuring only the originating client can decrypt it.

**Apple's Private Relay uses ODoH for all DNS resolution.** The device encrypts each DNS query with HPKE, sends it through Apple's ingress relay (which cannot read it), and the query reaches the DNS resolver (Cloudflare's `odoh.cloudflare-dns.com`) stripped of the user's identity. To ensure geographically correct answers, the device includes its public IP subnet in the encrypted query via the EDNS0 Client Subnet option.

## OHTTP: the protocol that makes oblivious requests possible

**Oblivious HTTP (RFC 9458, January 2024)** generalizes the ODoH pattern to arbitrary HTTP requests. Its three roles -- Client, Relay, and Gateway -- mirror ODoH's architecture but work for any HTTP exchange, not just DNS.

### Encryption and encapsulation in detail

The client first obtains the Gateway's **KeyConfig** (served as `application/ohttp-keys`), which contains a Key Identifier, HPKE KEM/KDF/AEAD algorithm IDs, and the Gateway's public key. The client then serializes its HTTP request using **Binary HTTP (RFC 9292)** -- a compact binary encoding of HTTP messages using QUIC-style variable-length integers for all lengths, capturing method, scheme, authority, path, headers, body, and optional padding in a protocol-independent format.

The encapsulated request is constructed by building a 7-byte header (`key_id || kem_id || kdf_id || aead_id`), calling `SetupBaseS(pkR, info)` to generate an HPKE sender context, then sealing the Binary HTTP payload with the header as AAD. The result is `hdr || enc || ciphertext` -- sent to the Relay as an HTTP POST with content type `application/ohttp-req`. The Relay forwards the opaque blob to the Gateway as `message/ohttp-req`. The Gateway decrypts using its private key, processes the request, and encrypts the response using a key derived from the HPKE context's Export function, bound to the original query plaintext and a fresh nonce. The encrypted response returns through the Relay as `message/ohttp-res` / `application/ohttp-res`.

### Why OHTTP only handles single request-response pairs

OHTTP is **stateless by design** -- each request creates a fresh ephemeral HPKE context with new keying material. This is essential for **unlinkability**: if state persisted between requests, the Gateway could correlate them as coming from the same client. But this makes OHTTP unsuitable for streaming, long-lived connections, or WebSocket-style communication. The entire Binary HTTP message must be encrypted as one unit before sending; there is no mechanism for incremental delivery.

This limitation directly shapes [[notes/Networking/icloud-private-relay-architecture|Private Relay's architecture]]: **OHTTP handles DNS queries** (which are inherently single request-response pairs), while **[[notes/Networking/quic-and-masque|MASQUE]] handles web traffic** (which requires streaming, multiplexing, and persistent connections).

---

## Related notes

- [[notes/Networking/quic-and-masque|QUIC and MASQUE]] -- the transport and tunneling protocols that handle the streaming traffic OHTTP cannot
- [[notes/Networking/icloud-private-relay-architecture|iCloud Private Relay Architecture]] -- how ODoH and OHTTP fit into the complete Private Relay flow
- [[notes/Networking/cellular-networks-and-ip-addresses|Cellular Networks and IP Addresses]] -- the network layer that provides the IP addresses these protocols aim to hide
