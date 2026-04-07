---
title: "Summary: DNS Privacy and Oblivious Protocols"
---

> **Full notes:** [[notes/Networking/dns-privacy-and-oblivious-protocols|DNS Privacy and Oblivious Protocols -->]]

## Key Concepts

### Evolution of DNS Encryption
- **Traditional DNS** (RFC 1035, 1987): plaintext UDP on port 53, visible to entire network path
- **DoT** (RFC 7858, 2016): DNS over TLS on port 853. Encrypted in transit but port is trivially blockable
- **DoH** (RFC 8484, 2018): DNS over HTTPS on port 443. Blends with normal web traffic, harder to block. Uses `application/dns-message` MIME type
- **Problem with DoT/DoH**: resolver still sees both client IP and query (single point of surveillance)

### ODoH -- Cryptographic Privacy Partitioning
- **Oblivious DoH** (RFC 9230, 2022): three-party model -- Client -> Relay -> Target Resolver
- Relay knows client IP but cannot decrypt query
- Target decrypts and resolves query but only sees Relay's IP
- Apple Private Relay uses ODoH for all DNS resolution (via `odoh.cloudflare-dns.com`)
- Geo-correct answers preserved via EDNS0 Client Subnet in encrypted query

### HPKE Encryption (RFC 9180)
- **KEM**: DHKEM(X25519, HKDF-SHA256) -- ephemeral DH key exchange, outputs `enc` + shared secret
- **KDF**: HKDF-SHA256 -- derives encryption keys from shared secret
- **AEAD**: AES-128-GCM -- encrypts and authenticates the DNS payload
- Client calls `SetupBaseS(pkR, "odoh query")` then `context.Seal(aad, plaintext)`
- Response encrypted with key from `context.Export("odoh response", Nk)` + fresh nonce

### OHTTP -- Generalized Oblivious HTTP (RFC 9458, Jan 2024)
- Extends ODoH pattern to arbitrary HTTP requests (Client -> Relay -> Gateway)
- Request serialized using Binary HTTP (RFC 9292) with QUIC-style variable-length integers
- Encapsulated as: `hdr(7 bytes) || enc || ciphertext`, content type `application/ohttp-req`
- **Stateless by design**: fresh ephemeral HPKE context per request for unlinkability
- Cannot handle streaming or persistent connections -- entire message encrypted as one unit

### Protocol Selection in Private Relay
- **OHTTP/ODoH** for DNS queries (single request-response pairs)
- **MASQUE** for web traffic (requires streaming, multiplexing, persistent connections)

## Quick Reference

```
DNS Privacy Evolution:
  Plaintext (port 53) -> DoT (port 853) -> DoH (port 443) -> ODoH (port 443, 3-party)

ODoH Three-Party Model:
  Client -----> Relay -----> Target Resolver
  (encrypts     (knows IP,   (decrypts query,
   query to      can't read   sees only
   Target)       query)       Relay IP)

HPKE Composition:
  KEM: X25519          (key agreement)
  KDF: HKDF-SHA256     (key derivation)
  AEAD: AES-128-GCM    (encryption + authentication)

OHTTP vs MASQUE:
  +------------------+------------------+
  | OHTTP            | MASQUE           |
  +------------------+------------------+
  | Single req/resp  | Streaming        |
  | Stateless        | Stateful         |
  | DNS queries      | Web traffic      |
  | Fresh keys/req   | Persistent conn  |
  +------------------+------------------+
```
