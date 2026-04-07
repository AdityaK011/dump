---
title: "Summary: GKE Request Path and Load Balancing"
---

> **Full notes:** [[notes/K8s/gke-request-path-and-load-balancing|GKE Request Path and Load Balancing -->]]

## Key Concepts

### DNS and Anycast
- Single global anycast IP advertised via BGP from 200+ edge PoPs
- No multi-IP rotation needed -- set DNS TTLs high (300s-3600s)
- Maglev (L4 LB) at edge uses ECMP forwarding, preserves TCP sessions across BGP route changes

### GCP Load Balancer Resource Chain
- Global Forwarding Rule -> Target HTTPS Proxy -> URL Map -> Backend Service -> NEG
- Target HTTPS Proxy terminates TLS, appends 2 IPs to X-Forwarded-For
- Backend Service default timeout: **30 seconds**
- Health check probe source IPs: **35.191.0.0/16** and **130.211.0.0/22**

### TLS Termination
- Terminates at GFEs at edge (close to user)
- SSL Policy profiles: COMPATIBLE, MODERN, RESTRICTED, FIPS_202205
- Supports TLS 1.0-1.3, HTTP/3 over QUIC
- SNI selection: exact match -> wildcard -> primary fallback; ECDSA prioritized over RSA

### Container-Native Load Balancing (NEGs)
- LB sends traffic **directly to pod IPs** -- no double hop
- Pod IPs from alias IP ranges, natively routable in VPC
- NEG controller manages pod readiness gate: `cloud.google.com/load-balancer-neg-ready`
- Prevents premature old-pod termination during rolling updates

### Node-Level Packet Routing
- **iptables**: O(n) rule chains, probability-based selection (legacy)
- **IPVS**: O(1) hash table lookups, multiple algorithms (being deprecated in K8s 1.35)
- **Dataplane V2 (Cilium/eBPF)**: O(1) map lookups, no context switches, default for Autopilot

### Pod Networking
- Each pod gets own network namespace + veth pair
- CNI plugin: ADD creates namespace/veth/IP, DEL tears down
- Config: `/etc/cni/net.d/`, binaries: `/opt/cni/bin/`

## Quick Reference

```
Request Path:
Client -> DNS (anycast IP) -> Edge PoP (Maglev) -> GFE (TLS termination)
     -> URL Map routing -> Backend Service -> NEG -> Pod IP directly

Response Path:
Pod -> veth -> host routing/eBPF -> NIC -> GFE (edge) -> Client
(Long leg on Google backbone, short encrypted leg to client)

Dataplane V2 Scale Limits:
  260,000 endpoints across all services
  7,500 nodes
  200,000 pods per cluster

Packet Routing Comparison:
  iptables  -> O(n)  | legacy, slow at scale
  IPVS      -> O(1)  | hash tables, being deprecated
  eBPF      -> O(1)  | no userspace context switch, DSR support
```
