---
title: "Summary: TLS 1.3 Handshake Deep Dive"
---

> **Full notes:** [[notes/Networking/tls-1.3-handshake|TLS 1.3 Handshake Deep Dive -->]]

## Key Concepts

### TLS 1.3 vs 1.2
- **Removed**: RSA key exchange (no forward secrecy), static DH, compression (CRIME), renegotiation, CBC ciphers, SHA-1, custom DH groups
- **Added**: 1-RTT handshake, 0-RTT early data, encrypted handshake (after ServerHello), mandatory ECDHE, AEAD-only ciphers, HKDF key schedule
- Cipher suites reduced from 300+ to 5 (only specify AEAD + hash)
- Key exchange and auth negotiated separately via extensions

### 1-RTT Full Handshake
- Client sends `key_share` (ephemeral ECDHE pub key) speculatively in ClientHello
- Server responds with its key_share in ServerHello
- Everything after ServerHello is encrypted with handshake traffic keys
- CertificateVerify: server signs transcript hash with private key (proves identity)
- Finished: HMAC over transcript (key confirmation + integrity)
- If server doesn't support offered groups: HelloRetryRequest -> falls back to 2-RTT

### ECDHE Key Exchange
- Both sides generate ephemeral keypairs, compute shared secret `S = a * b * G`
- X25519: Montgomery curve over F(2^255-19), 32-byte keys
- Ephemeral keys discarded after shared secret derived -> forward secrecy

### Forward Secrecy
- Certificate private key used only for authentication (CertificateVerify), NOT key exchange
- Even if server key compromised later, past sessions unrecoverable
- Cannot decrypt captured TLS 1.3 traffic with server private key -- use `SSLKEYLOGFILE`

### Key Schedule (HKDF-based)
- Three stages: Early Secret (PSK) -> Handshake Secret (ECDHE) -> Master Secret
- Each secret includes transcript hash -- any tampering produces different keys
- Traffic secrets expanded to write key + IV; nonce = IV XOR sequence_number

### 0-RTT Resumption
- Server sends NewSessionTicket with PSK after handshake
- Client uses PSK to send early data in first flight
- **Fundamental replay risk**: no server nonce to bind to
- Mitigations: only idempotent requests, strike registers, disable 0-RTT (`ssl_early_data off`)
- PSK+DHE (`psk_dhe_ke`) preserves forward secrecy; PSK-only does not

### mTLS (Mutual TLS)
- Server sends CertificateRequest; client sends Certificate + CertificateVerify
- In Istio: SPIFFE SVIDs as certs, SAN = `spiffe://cluster.local/ns/<ns>/sa/<sa>`
- Istiod acts as CA, certs short-lived (24h), rotated via SDS
- PeerAuthentication modes: STRICT (mTLS only), PERMISSIVE (both), DISABLE
- mTLS > network policies: crypto identity bound to service account, not IP

### Downgrade Prevention
- Server embeds sentinel `DOWNGRD` + version byte in last 8 bytes of ServerHello.random
- TLS 1.3 client detects forced downgrade and aborts

## Quick Reference

```
Handshake Flow:
  ClientHello (plaintext) -------->
                          <-------- ServerHello (plaintext)
  ============ ENCRYPTION STARTS ============
                          <-------- {EncryptedExtensions, Certificate,
                                     CertificateVerify, Finished}
  {Finished}              -------->
  ========== APPLICATION KEYS ==========
  [App Data]              <-------> [App Data]

TLS 1.3 Cipher Suites (only 5):
  TLS_AES_128_GCM_SHA256       TLS_AES_256_GCM_SHA384
  TLS_CHACHA20_POLY1305_SHA256 TLS_AES_128_CCM_SHA256
  TLS_AES_128_CCM_8_SHA256

Debugging:
  openssl s_client -connect host:443 -tls1_3 -msg -debug
  curl -v --tlsv1.3 https://host
  SSLKEYLOGFILE=keys.log curl https://host   # then use Wireshark
  istioctl proxy-config secret <pod>          # check mesh certs

Certificate Chain: Root CA -> Intermediate CA -> Leaf cert
  Validation: chain build -> sig verify -> validity period -> OCSP -> SAN match -> key usage
  SAN (Subject Alternative Name) replaces CN for hostname matching

SPIFFE ID format: spiffe://cluster.local/ns/<namespace>/sa/<service-account>
Istio cert rotation: SDS (Secret Discovery Service), hot-swap, ~12h rotation cycle
```
