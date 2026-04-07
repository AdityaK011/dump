---
title: "TLS 1.3 Handshake Deep Dive"
---

TLS 1.3 (RFC 8446) is a ground-up rework of the Transport Layer Security protocol. It removes legacy cruft, hardens the cryptographic foundation, and cuts a full round trip from connection setup. This note covers the protocol at the byte-and-message level, targeting platform engineers who need to debug mTLS in service meshes, reason about forward secrecy, and answer hard interview questions.

## TLS 1.3 vs TLS 1.2 -- What Changed

### Removed in TLS 1.3

| Feature | Why It Was Removed |
|---|---|
| **RSA key exchange** | No forward secrecy. If the server's RSA private key leaks, every past session recorded by an adversary is decryptable. |
| **Static Diffie-Hellman** | Same problem. The DH parameters were reused across sessions, so compromising them breaks all sessions. |
| **Compression** | Enabled CRIME/BREACH attacks. Compressed ciphertext leaks plaintext length information that can be exploited. |
| **Renegotiation** | Complex state machine, source of multiple CVEs (triple handshake attack, renegotiation injection). |
| **ChangeCipherSpec message** | Vestigial message that added complexity but no security. In TLS 1.3, encryption transitions are implicit. |
| **RC4, 3DES, CBC-mode ciphers** | Known weaknesses (BEAST, Lucky13, Sweet32). TLS 1.3 mandates AEAD ciphers only. |
| **Custom DH groups** | Misconfigurations (Logjam attack with 512-bit DH). Only named groups allowed now. |
| **SHA-1 in signatures** | Collision attacks proven practical (SHAttered, 2017). |

### Added in TLS 1.3

| Feature | Purpose |
|---|---|
| **1-RTT handshake** | Client sends key_share in ClientHello itself, server responds with its share. Keys are derived before the second flight. |
| **0-RTT early data** | Resumption with pre-shared keys allows sending application data in the very first flight, at the cost of replay protection. |
| **Encrypted handshake** | After ServerHello, every message is encrypted with handshake traffic keys. Certificate, extensions, and Finished are never sent in plaintext. |
| **Mandatory (EC)DHE** | Every full handshake uses ephemeral Diffie-Hellman. Forward secrecy is not optional. |
| **AEAD-only ciphers** | AES-128-GCM, AES-256-GCM, ChaCha20-Poly1305. No more separate MAC. |
| **HKDF-based key schedule** | Clean, formally analyzed key derivation replaces the ad-hoc PRF of TLS 1.2. |

### Cipher Suite Simplification

TLS 1.2 had 300+ cipher suites encoding key exchange + authentication + cipher + MAC. TLS 1.3 reduces this to 5 cipher suites that specify only AEAD + hash:

```
TLS_AES_128_GCM_SHA256          (0x13,0x01)
TLS_AES_256_GCM_SHA384          (0x13,0x02)
TLS_CHACHA20_POLY1305_SHA256    (0x13,0x03)
TLS_AES_128_CCM_SHA256          (0x13,0x04)
TLS_AES_128_CCM_8_SHA256        (0x13,0x05)
```

Key exchange and authentication are negotiated separately via extensions (`supported_groups`, `signature_algorithms`, `key_share`).

---

## The 1-RTT Full Handshake

This is the core of TLS 1.3. It completes in a single round trip.

```
   Client                                           Server

   ClientHello
     + supported_versions (0x0304)
     + key_share (x25519 pub key)
     + signature_algorithms
     + psk_key_exchange_modes
     + server_name (SNI)
   --------------------------------->
                                                ServerHello
                                                  + supported_versions
                                                  + key_share (x25519 pub key)
                                     <---------------------------------

          === Handshake keys derived (ECDHE shared secret) ===
          === All messages below are ENCRYPTED ===

                                     <--- {EncryptedExtensions}
                                     <--- {Certificate}
                                     <--- {CertificateVerify}
                                     <--- {Finished}

   {Finished}  ------------------->

          === Application keys derived ===
          === Application data can flow ===

   [Application Data]  <---------->  [Application Data]
```

Messages in `{}` braces are encrypted with handshake traffic keys. Messages in `[]` brackets are encrypted with application traffic keys.

### Step-by-Step Breakdown

**1. ClientHello** (plaintext)

The client generates an ephemeral ECDHE keypair and sends:
- `supported_versions`: Lists `0x0304` (TLS 1.3). The legacy `ClientHello.version` field is frozen at `0x0303` (TLS 1.2) for middlebox compatibility.
- `key_share`: Contains the client's ephemeral public key for one or more named groups (e.g., x25519, secp256r1). This is the key change from TLS 1.2 -- the client speculatively sends key material before knowing what the server supports.
- `signature_algorithms`: What signature schemes the client accepts for CertificateVerify (e.g., `ecdsa_secp256r1_sha256`, `rsa_pss_rsae_sha256`).
- `psk_key_exchange_modes`: If PSK resumption is supported, indicates `psk_dhe_ke` (PSK + ECDHE) or `psk_ke` (PSK only).
- `server_name` (SNI): Unencrypted hostname. This is the field that Encrypted Client Hello (ECH) aims to protect.

**2. ServerHello** (plaintext)

The server selects:
- The TLS 1.3 version from `supported_versions`.
- A cipher suite (e.g., `TLS_AES_256_GCM_SHA384`).
- A `key_share` entry with the server's ephemeral public key for the client's chosen group.

At this point, both sides have enough material to compute the ECDHE shared secret and derive handshake traffic keys.

**3. EncryptedExtensions** (encrypted with handshake keys)

Non-cryptographic extensions that do not need to be in ServerHello (e.g., `server_name` acknowledgment, ALPN protocol selection like `h2`). Separated from ServerHello because ServerHello must remain compatible with TLS 1.2 parsers.

**4. Certificate** (encrypted)

The server's certificate chain: leaf certificate, intermediate CA(s). Root CA is typically omitted because the client must already have it in its trust store.

**5. CertificateVerify** (encrypted)

The server signs a hash of the entire handshake transcript (ClientHello through Certificate) with its private key. This proves that the server holds the private key corresponding to the certificate and binds the handshake to this specific session, preventing replay and tampering.

The signed content is structured as:
```
64 bytes of 0x20 (spaces)
"TLS 1.3, server CertificateVerify"
0x00
Hash(Handshake Context)
```

**6. Server Finished** (encrypted)

HMAC over the entire handshake transcript using the server's finished key. This provides key confirmation -- proof that the server derived the same handshake keys.

**7. Client Finished** (encrypted)

HMAC over the full transcript including the server's Finished. After the server receives this, both sides derive application traffic keys and the handshake is complete.

### Where Encryption Starts

```
   ClientHello          ------>  plaintext
                        <------  ServerHello         plaintext
   - - - - - - - ENCRYPTION BOUNDARY - - - - - - -
                        <------  EncryptedExtensions encrypted (handshake keys)
                        <------  Certificate         encrypted (handshake keys)
                        <------  CertificateVerify   encrypted (handshake keys)
                        <------  Finished            encrypted (handshake keys)
   Finished             ------>  encrypted (handshake keys)
   - - - - - - - KEY CHANGE TO APPLICATION KEYS - - - - - -
   Application Data     <-----> encrypted (application keys)
```

In TLS 1.2, the Certificate and all extensions were sent in plaintext. An observer could see the server's certificate and determine the identity of the server beyond just SNI. TLS 1.3 encrypts everything after ServerHello.

---

## ECDHE Key Exchange

Ephemeral Elliptic Curve Diffie-Hellman (ECDHE) is the backbone of TLS 1.3 key agreement. The two most common curves are X25519 (Curve25519, the default in practice) and P-256 (secp256r1, NIST standard).

### How It Works (Conceptual)

```
   Client                                      Server
   
   1. Generate ephemeral private key: a        1. Generate ephemeral private key: b
   2. Compute public key: A = a * G            2. Compute public key: B = b * G
   3. Send A in key_share  -------->
                           <--------  4. Send B in key_share
   5. Compute shared secret: S = a * B         5. Compute shared secret: S = b * A
   
   Both arrive at S = a * b * G (commutativity of scalar multiplication)
```

- `G` is the generator point (base point) of the elliptic curve, a fixed public parameter.
- `a`, `b` are random 32-byte scalars (private keys), generated fresh for each handshake.
- `A`, `B` are points on the curve (public keys).
- The shared secret `S` is a point on the curve. Its x-coordinate is extracted as the raw shared secret (32 bytes for X25519).

### X25519 Specifics

- Montgomery curve: `y^2 = x^3 + 486662x^2 + x` over `F(2^255 - 19)`
- Uses only x-coordinates in computation (Montgomery ladder), making implementation simpler and resistant to side-channel attacks.
- Private key: 32 random bytes, clamped (clear bits 0, 1, 2 of first byte; set bit 254; clear bit 255).
- Public key: 32 bytes. Computed as `X25519(private_key, 9)` where 9 is the base point.
- Shared secret: `X25519(my_private, their_public)`.

### Why Forward Secrecy

The ephemeral keys `a` and `b` are generated for each connection and discarded after the shared secret is derived. Even if an attacker records the entire ciphertext and later compromises the server's long-term private key (the one in the certificate), they cannot recover `a` or `b`, and therefore cannot recompute `S`.

Contrast with TLS 1.2 RSA key exchange: the client encrypts the premaster secret with the server's RSA public key. If the server's RSA private key is later compromised, every recorded session is decryptable.

---

## Key Schedule

TLS 1.3 uses HKDF (HMAC-based Key Derivation Function, RFC 5869) as its sole key derivation mechanism. The key schedule is a tree of `HKDF-Extract` and `HKDF-Expand-Label` operations.

### HKDF Primitives

```
HKDF-Extract(salt, IKM) -> PRK
   Takes input keying material (IKM) and a salt, produces a pseudorandom key (PRK).
   Internally: HMAC-Hash(salt, IKM)

HKDF-Expand-Label(Secret, Label, Context, Length) -> output
   Expands a secret with a label and context (usually a transcript hash).
   Internally: HKDF-Expand(Secret, HkdfLabel, Length)
   where HkdfLabel = length || "tls13 " || Label || Context
```

### Full Key Schedule Diagram

```
                 0 (all-zero PSK if no PSK)
                 |
                 v
   PSK -----> HKDF-Extract = Early Secret
                 |
                 +---> Derive-Secret(., "ext binder" | "res binder", "")
                 |         = binder_key
                 |
                 +---> Derive-Secret(., "c e traffic", ClientHello)
                 |         = client_early_traffic_secret  [0-RTT keys]
                 |
                 v
           Derive-Secret(., "derived", "")
                 |
                 v
   ECDHE --> HKDF-Extract = Handshake Secret
                 |
                 +---> Derive-Secret(., "c hs traffic", ClientHello..ServerHello)
                 |         = client_handshake_traffic_secret
                 |
                 +---> Derive-Secret(., "s hs traffic", ClientHello..ServerHello)
                 |         = server_handshake_traffic_secret
                 |
                 v
           Derive-Secret(., "derived", "")
                 |
                 v
   0 -------> HKDF-Extract = Master Secret
                 |
                 +---> Derive-Secret(., "c ap traffic", ClientHello..server Finished)
                 |         = client_application_traffic_secret_0
                 |
                 +---> Derive-Secret(., "s ap traffic", ClientHello..server Finished)
                 |         = server_application_traffic_secret_0
                 |
                 +---> Derive-Secret(., "res master", ClientHello..client Finished)
                          = resumption_master_secret  [used for PSK in resumption]
```

### From Secrets to Actual Keys

Each traffic secret is expanded into the actual symmetric keys used for AEAD encryption:

```
   traffic_secret
       |
       +---> HKDF-Expand-Label(., "key", "", key_length) = write key
       |
       +---> HKDF-Expand-Label(., "iv", "", 12) = write IV (nonce)
```

The nonce for each record is computed as `IV XOR sequence_number` (64-bit counter). This avoids nonce reuse without transmitting a per-record nonce.

### Why This Matters

- **Separation of key material**: Handshake keys are cryptographically independent from application keys. Compromising one does not compromise the other.
- **Transcript binding**: Every derived secret includes a hash of the handshake transcript up to that point. This means any tampering with handshake messages produces different keys on each side, and the Finished messages will not verify.
- **Key update**: Application traffic secrets can be rotated using `HKDF-Expand-Label(current_secret, "traffic upd", "", Hash.length)` without a new handshake.

---

## Certificate Chain of Trust

### The Chain

```
   +---------------------------+
   |  Root CA Certificate      |  <-- Pre-installed in client trust store
   |  (self-signed)            |      (/etc/ssl/certs, NSS DB, JVM cacerts)
   |  Subject: DigiCert Root   |
   |  Issuer: DigiCert Root    |
   +---------------------------+
              |
              | signs (CA:TRUE, pathLen constraint)
              v
   +---------------------------+
   |  Intermediate CA Cert     |  <-- Sent by server in Certificate message
   |  Subject: DigiCert G2     |
   |  Issuer: DigiCert Root    |
   +---------------------------+
              |
              | signs (CA:FALSE or CA:TRUE pathLen:0)
              v
   +---------------------------+
   |  Leaf Certificate         |  <-- Sent by server in Certificate message
   |  Subject: *.example.com   |
   |  Issuer: DigiCert G2      |
   |  SAN: *.example.com       |
   |  Public Key: (ECDSA P-256)|
   +---------------------------+
```

### Validation Steps

1. **Chain building**: The client receives the leaf cert and intermediate(s). It chains each certificate's `Issuer` field to the next certificate's `Subject` field until it reaches a root CA in its trust store.
2. **Signature verification**: Each certificate's signature is verified using the issuer's public key. The root CA is self-signed and verified against the trust store copy.
3. **Validity period**: `notBefore` and `notAfter` are checked against current time.
4. **Revocation**: OCSP stapling (sent in EncryptedExtensions via `status_request`) or CRL checked.
5. **Name matching**: The requested hostname (from SNI) must match a SAN (Subject Alternative Name) entry in the leaf cert. The CN (Common Name) field is deprecated for this purpose.
6. **Key usage**: The leaf cert must have appropriate key usage extensions (e.g., `digitalSignature` for ECDSA).

### CertificateVerify

After sending the Certificate, the server must prove it possesses the corresponding private key. It does this by signing the handshake transcript hash:

```
content = 0x20 * 64                                    // 64 space characters
        + "TLS 1.3, server CertificateVerify\x00"      // context string + null
        + Hash(handshake_messages_so_far)               // transcript hash

signature = Sign(server_private_key, content)
```

The client verifies this signature using the public key from the leaf certificate. This binds the server's identity to this specific handshake -- an attacker who obtains a valid certificate but not the private key cannot produce a valid CertificateVerify.

---

## 0-RTT Resumption

After a successful handshake, the server can send a `NewSessionTicket` message containing a PSK (Pre-Shared Key) identity. On subsequent connections, the client can use this PSK to send encrypted application data in its very first flight.

### 0-RTT Flow

```
   Client                                            Server

   ClientHello
     + key_share
     + pre_shared_key (PSK identity + binder)
     + early_data
   {Application Data}   -------->                    0-RTT data (encrypted
                                                      with early traffic keys)
                                                ServerHello
                                                  + pre_shared_key
                                                  + key_share
                         <--------
                         <--------  {EncryptedExtensions}
                                      + early_data (accepted/rejected)
                         <--------  {Finished}
   {EndOfEarlyData}
   {Finished}            -------->
   [Application Data]   <--------> [Application Data]
```

### PSK and NewSessionTicket

After a full handshake, the server sends:

```
NewSessionTicket {
    ticket_lifetime: 7200,       // seconds
    ticket_age_add: 0x1a2b3c4d,  // random offset to obfuscate ticket age
    ticket_nonce: 0x00,
    ticket: <opaque blob>,       // server-encrypted session state
    extensions: {
        max_early_data_size: 16384
    }
}
```

The PSK is derived as:
```
resumption_master_secret (from the original handshake)
   |
   v
HKDF-Expand-Label(., "resumption", ticket_nonce, Hash.length) = PSK
```

### Replay Risk

0-RTT data has a fundamental problem: it is not protected against replay. An attacker who captures the ClientHello + early data can retransmit it to the server, and the server has no built-in mechanism in the protocol itself to detect the duplicate.

**Why replay is inherent to 0-RTT**:
- In 1-RTT, the server's random contribution (ServerHello.random, key_share) ensures uniqueness. The client's Finished depends on the server's values.
- In 0-RTT, the client sends data before receiving anything from the server. There is no server-contributed nonce to bind to.

**Mitigations** (application level):
- Only send idempotent requests as early data (GET, not POST with side effects).
- Servers can implement single-use ticket databases or strike registers, but these are operationally expensive, especially across distributed server fleets.
- `max_early_data_size` limits the damage window.
- Many deployments simply disable 0-RTT. Cloudflare and AWS ALB support it selectively.

---

## Forward Secrecy

Forward secrecy (also called perfect forward secrecy, PFS) means that compromise of long-term keys does not compromise past session keys.

### TLS 1.2 RSA Key Exchange (No Forward Secrecy)

```
   Client                                         Server

   ClientHello  -------->
                <--------  ServerHello
                <--------  Certificate (RSA public key)
   
   premaster_secret = random 48 bytes
   encrypted_pms = RSA_Encrypt(server_RSA_pubkey, premaster_secret)
   
   ClientKeyExchange
     (encrypted_pms) -------->
   
   Server decrypts: premaster_secret = RSA_Decrypt(server_RSA_privkey, encrypted_pms)
   Both derive: master_secret = PRF(premaster_secret, "master secret",
                                     ClientHello.random + ServerHello.random)
```

If the server's RSA private key is compromised at any future time (stolen, subpoenaed, leaked), an adversary who recorded the ciphertext can decrypt `encrypted_pms`, recover `premaster_secret`, derive all session keys, and decrypt all recorded traffic. This is a passive, retroactive attack.

### TLS 1.3 ECDHE (Mandatory Forward Secrecy)

The ephemeral private keys (`a`, `b`) exist only in memory during the handshake. They are never written to disk and are zeroed after the shared secret is derived. Even if the server's certificate private key is later compromised, the attacker cannot recover the per-session ECDHE keys.

The certificate private key is only used for **authentication** (CertificateVerify), not for **key exchange**. These roles are cleanly separated in TLS 1.3.

### Practical Implications for SREs

- **Key rotation**: You can rotate certificates without worrying about past traffic. But you should still rotate promptly because a compromised signing key allows impersonation of future connections.
- **Traffic capture**: NSA-style "record everything, decrypt later" attacks are defeated by forward secrecy. This is one of the primary motivations for TLS 1.3's mandatory ECDHE.
- **Debugging**: Forward secrecy means you cannot decrypt captured traffic using the server's private key. For debugging, use `SSLKEYLOGFILE` (supported by OpenSSL, BoringSSL, NSS) which logs per-session secrets. Wireshark can consume this file.

---

## mTLS (Mutual TLS)

In standard TLS, only the server authenticates. In mutual TLS, the client also presents a certificate to prove its identity.

### mTLS Handshake Flow

```
   Client                                           Server

   ClientHello          -------->
                        <--------  ServerHello
                        <--------  {EncryptedExtensions}
                        <--------  {CertificateRequest}    <-- server asks for
                        <--------  {Certificate}                client cert
                        <--------  {CertificateVerify}
                        <--------  {Finished}

   {Certificate}        -------->                          <-- client sends its cert
   {CertificateVerify}  -------->                          <-- client proves identity
   {Finished}           -------->

   [Application Data]   <--------> [Application Data]
```

The `CertificateRequest` message from the server includes:
- `certificate_authorities`: list of acceptable CA distinguished names.
- `signature_algorithms`: acceptable signature schemes for the client's CertificateVerify.
- `extensions`: OID filters, etc.

### mTLS in Service Meshes (Istio / Envoy / SPIFFE)

In a Kubernetes service mesh like Istio, mTLS is used for service-to-service authentication. The sidecar proxy (Envoy) handles all TLS termination and origination.

```
   Pod A                                              Pod B
   +------------------+                    +------------------+
   | App container    |                    | App container    |
   |  (plaintext)     |                    |  (plaintext)     |
   +--------+---------+                    +--------+---------+
            | localhost                              ^ localhost
   +--------v---------+                    +--------+---------+
   | Envoy sidecar    |  -- mTLS --------> | Envoy sidecar    |
   | (istio-proxy)    |     over the       | (istio-proxy)    |
   | Cert: spiffe://  |     pod network    | Cert: spiffe://  |
   |   cluster.local/ |                    |   cluster.local/ |
   |   ns/default/    |                    |   ns/default/    |
   |   sa/svc-a       |                    |   sa/svc-b       |
   +------------------+                    +------------------+
```

**SPIFFE (Secure Production Identity Framework for Everyone)**:
- Each workload gets an X.509 certificate (called an SVID -- SPIFFE Verifiable Identity Document).
- The Subject Alternative Name (SAN) contains a SPIFFE ID: `spiffe://cluster.local/ns/<namespace>/sa/<service-account>`.
- Istiod (the control plane) acts as the CA, signing CSRs from each Envoy proxy.
- Certificates are short-lived (default 24 hours) and automatically rotated.

**Authorization** (separate from authentication):
- mTLS proves identity. `AuthorizationPolicy` resources in Istio define who can talk to whom.
- Example: "Only pods with SPIFFE ID `spiffe://cluster.local/ns/frontend/sa/web` can call `spiffe://cluster.local/ns/backend/sa/api` on port 8080 with GET /health."

**PeerAuthentication modes in Istio**:
- `STRICT`: mTLS only. Plaintext connections are rejected.
- `PERMISSIVE`: Accepts both mTLS and plaintext. Used during migration.
- `DISABLE`: No mTLS.

### Why mTLS Over Network Policies Alone

Network policies (Calico, Cilium) operate at L3/L4 -- they control which IPs/ports can communicate. They do not authenticate identity. A compromised pod in the same namespace could impersonate another service. mTLS provides cryptographic identity bound to a service account, not an IP address.

---

## Interview Prep: Deep Q&A

### Q1: Why does TLS 1.3 complete in 1-RTT while TLS 1.2 needs 2-RTT?

**A**: In TLS 1.2, the client must wait for the server's ServerHello and Certificate to know the server's key exchange parameters before sending its own key material (ClientKeyExchange). This adds an extra round trip.

In TLS 1.3, the client speculatively sends its ECDHE public key in the ClientHello via the `key_share` extension. The server responds with its own key share in ServerHello. Both sides can derive keys immediately after ServerHello, so the server can send its encrypted messages (Certificate, Finished) in the same flight as ServerHello.

If the server does not support any of the client's offered groups, it responds with `HelloRetryRequest`, and the handshake falls back to 2-RTT.

### Q2: What happens if a middlebox strips the `supported_versions` extension?

**A**: The `supported_versions` extension is how TLS 1.3 is negotiated. The legacy version field in ClientHello is frozen at `0x0303` (TLS 1.2). If a middlebox strips the extension, the server sees a TLS 1.2 ClientHello and negotiates TLS 1.2. This is by design -- TLS 1.3 was carefully engineered to look like TLS 1.2 to broken middleboxes (the "ossification" problem that also motivated QUIC).

### Q3: Can you decrypt TLS 1.3 traffic with the server's private key?

**A**: No. The server's private key is only used to sign the CertificateVerify message (authentication). The encryption keys are derived from the ECDHE shared secret, which depends on ephemeral keys that are discarded after the handshake. To decrypt traffic, you need the per-session secrets, obtainable via `SSLKEYLOGFILE` or by instrumenting the application's TLS library.

### Q4: What is the difference between PSK-only and PSK+DHE resumption?

**A**: In PSK-only (`psk_ke`), the session keys are derived solely from the pre-shared key. If the PSK is compromised (e.g., the server's ticket encryption key leaks), all resumed sessions are decryptable. There is no forward secrecy for resumed sessions.

In PSK+DHE (`psk_dhe_ke`), the client and server still perform an ephemeral ECDHE exchange during resumption. The session keys are derived from both the PSK and the ECDHE shared secret. This preserves forward secrecy. TLS 1.3 strongly recommends `psk_dhe_ke`.

### Q5: How does 0-RTT replay protection work in practice?

**A**: The protocol itself does not prevent replay. Defenses are application-level:
- Servers can maintain a strike register (a set of recently seen ClientHello hashes) to reject duplicates. This requires shared state across server instances, which is hard in distributed systems.
- Applications should only send idempotent data as 0-RTT early data (e.g., HTTP GET, not POST with a payment).
- Frameworks like Cloudflare's implementation associate 0-RTT with `cf-connecting-ip` and rate-limit replays.
- Many production systems simply disable 0-RTT (`ssl_early_data off` in nginx).

### Q6: How does Istio handle certificate rotation without downtime?

**A**: Istiod acts as the certificate authority. Each Envoy sidecar generates a private key locally, creates a CSR (Certificate Signing Request), and sends it to Istiod via SDS (Secret Discovery Service, an xDS API). Istiod signs the CSR and returns the certificate. Envoy receives the new cert via SDS and hot-swaps it without restarting or dropping connections. Default cert lifetime is 24 hours, and rotation happens well before expiry (at roughly 50% of lifetime).

### Q7: Why does TLS 1.3 encrypt the Certificate message but not SNI?

**A**: The Certificate is sent after ServerHello, when handshake keys are available. SNI is in the ClientHello, which is sent before any key exchange occurs -- there are no keys to encrypt it with. Encrypted Client Hello (ECH, formerly ESNI) solves this by encrypting the ClientHello's sensitive fields using a public key published in DNS (via HTTPS/SVCB records). The outer ClientHello contains a non-sensitive "cover" SNI, and the real SNI is inside the encrypted inner ClientHello.

### Q8: What is the purpose of the Finished message?

**A**: The Finished message is an HMAC over the handshake transcript using a key derived from the handshake traffic secret. It serves three purposes:
1. **Key confirmation**: Proves both sides derived the same keys from the ECDHE exchange.
2. **Transcript integrity**: Any tampering with handshake messages (by a MITM) results in different transcript hashes, different derived keys, and mismatched Finished MACs. The handshake fails.
3. **Authentication binding**: The server's Finished proves the server computed keys using the same ClientHello the client sent (preventing downgrade attacks).

### Q9: How do you debug a TLS handshake failure in production?

**A**: Layered approach:
1. **openssl s_client**: `openssl s_client -connect host:443 -tls1_3 -msg -debug` shows the handshake messages, certificate chain, and negotiated parameters.
2. **curl verbose**: `curl -v --tlsv1.3 https://host` shows TLS version, cipher, certificate info.
3. **Packet capture + Wireshark**: Capture with tcpdump, analyze in Wireshark. For TLS 1.3, you need `SSLKEYLOGFILE` to decrypt.
4. **Server logs**: Check for `ssl_error` in nginx/envoy logs. Common issues: expired cert, untrusted CA, SNI mismatch, cipher mismatch.
5. **Envoy admin**: In Istio, `istioctl proxy-config secret <pod>` shows the loaded certs and their expiry. `istioctl proxy-config listener <pod>` shows TLS context configuration.

### Q10: What prevents a downgrade attack from TLS 1.3 to 1.2?

**A**: TLS 1.3 servers embed a sentinel value in the last 8 bytes of `ServerHello.random` when they negotiate a lower version:
- Negotiating TLS 1.2: random ends with `44 4F 57 4E 47 52 44 01` ("DOWNGRD" + 0x01)
- Negotiating TLS 1.1 or below: random ends with `44 4F 57 4E 47 52 44 00`

A TLS 1.3-capable client checks for these sentinels. If it offered TLS 1.3 but the server responded with TLS 1.2 and the sentinel is present, the client knows the server supports TLS 1.3 and a downgrade is being forced (possibly by a MITM). The client aborts with an `illegal_parameter` alert.

---

## Related Notes

- [[notes/Networking/tcp-socket-internals|TCP Socket Internals]]
- [[notes/Networking/http-grpc-and-streaming|HTTP, gRPC & Streaming]]
- [[notes/K8s/istio-and-envoy-internals|Istio & Envoy Internals]]
