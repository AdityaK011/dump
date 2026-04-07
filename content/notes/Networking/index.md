---
title: Networking
---

Notes on networking protocols, privacy architectures, and transport layers.

## TCP & Sockets
- [[notes/Networking/tcp-socket-internals|TCP Socket Internals]] — Listening vs connected sockets, SYN/accept queues, socket buffers, kernel packet path, kube-proxy DNAT transparency
- [[notes/Networking/container-networking-internals|Container Networking Internals]] — Network namespaces, veth pairs, bridges, iptables DNAT chains, conntrack, Calico vs Cilium

## TLS & Security
- [[notes/Networking/tls-1.3-handshake|TLS 1.3 Handshake Deep Dive]] — ECDHE key exchange, certificate chains, forward secrecy, 0-RTT resumption, mTLS

## Application Protocols
- [[notes/Networking/http-grpc-and-streaming|HTTP, gRPC & Streaming]] — HTTP/2 multiplexing, gRPC over HTTP/2, SSE, WebSockets, connection pooling

## Privacy & DNS
- [[notes/Networking/dns-privacy-and-oblivious-protocols|DNS Privacy & Oblivious Protocols]] — From plaintext DNS to ODoH and OHTTP, with HPKE encryption details

## Cellular & IP
- [[notes/Networking/cellular-networks-and-ip-addresses|Cellular Networks & IP Addresses]] — How phones attach to networks, GTP tunnels, and when IP addresses change

## Transport
- [[notes/Networking/quic-and-masque|QUIC & MASQUE]] — QUIC's unified transport/encryption, connection migration, and MASQUE tunneling

## Architecture
- [[notes/Networking/icloud-private-relay-architecture|iCloud Private Relay Architecture]] — Dual-hop privacy partitioning, SSE streaming, and session persistence
