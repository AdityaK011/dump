---
title: "Summary: Service Mesh, Multi-Cluster, and Advanced Patterns"
---

> **Full notes:** [[notes/K8s/service-mesh-multi-cluster-and-advanced-patterns|Service Mesh, Multi-Cluster, and Advanced Patterns -->]]

## Key Concepts

### Container-Native LB vs Legacy
- Legacy: LB -> random node -> kube-proxy -> random pod (double hop, SNAT, imbalanced)
- NEG-based: LB -> Pod directly (single hop, per-pod health checks, source IP preserved)
- **Non-negotiable for production HTTP workloads on GKE**

### Load Balancer Selection
- **Global external ALB**: internet-facing, anycast from 200+ PoPs, Cloud Armor/CDN, Premium Tier
- **Regional ALB**: Envoy-based, lower cost, Standard Tier, data residency compliance
- **Internal ALB**: Envoy-based, regional, VPC-internal; needs proxy-only subnet; no FrontendConfig/CDN/managed certs
- **Passthrough NLB** (L4): preserves client IP, DSR, supports TCP/UDP/ESP/ICMP/GRE
- **Proxy NLB** (TCP/SSL Proxy): terminates connections, can be global, backends see LB IP

### Multi-Cluster Networking
- When needed: regional HA, blast radius isolation, regulatory locality, >15,000 nodes
- **Multi-cluster Gateway** (preferred): standard Gateway API + `gke-l7-global-external-managed-mc`
- **MCS**: export Service -> `ServiceImport` in all fleet clusters at `<svc>.<ns>.svc.clusterset.local`
- **Cloud Service Mesh**: Istio-based, locality-aware LB, auto endpoint discovery via Fleet
- **Karmada**: multi-cloud orchestrator, Propagation Policies (Duplicated/Divided strategies)

### Service Mesh: Sidecar vs Ambient
- **Sidecar mode**: Envoy in every pod, iptables redirect (15006 inbound, 15001 outbound), 2 proxy hops per request
  - Cost: **~0.63ms p90 latency**, **50-100MB + 0.20 vCPU per sidecar**
- **Ambient mode** (GA since Istio 1.22 single-cluster):
  - **ztunnel** (Rust DaemonSet): L4 mTLS, TCP auth, SPIFFE identity, HBONE tunneling; **~20-50MB per node**
  - **Waypoint proxy** (Envoy Deployment per namespace): L7 when needed, 1 proxy hop (vs 2)
  - **73% less CPU** than sidecars (Solo.io benchmark)
  - Mesh keys in ztunnel, not in pod (better security)
- SPIFFE identity: `spiffe://cluster.local/ns/<ns>/sa/<sa>`, X.509, auto-rotated (~24h TTL)
- Worth it when: mTLS everywhere, canary/fault injection, multi-cluster discovery, centralized observability, L7 authz

### Rate Limiting and Circuit Breaking
- **Cloud Armor** (edge): `throttle` or `rate-based-ban`, keyed by IP/header/cookie/path
- **Envoy circuit breaking**: connection pool limits (503 with `UO` flag on overflow), outlier detection (consecutive 5xx, exponential backoff, 50% panic threshold)
- Timeout layering: LB (30s) -> Ingress/Gateway (25s) -> mesh (20s) -> app client (15s)
- Retries: only idempotent ops, cap with `maxRetries` to prevent storms

### gRPC Load Balancing
- Problem: HTTP/2 = single TCP connection = L4 LB sends all RPCs to one pod
- **L7 LB**: GCP ALB with HTTP/2 backend or Istio (routes individual RPCs)
- **Client-side**: headless service + `dns:///` scheme + `round_robin` policy
- **xDS**: `xds:///` resolver with Traffic Director or custom xDS server
- Set `GRPC_MAX_CONNECTION_AGE` to force re-resolution on scale events

### WebSocket Handling
- Classic ALB: timeout is **hard cutoff** (default 30s kills WebSocket)
- Global/Regional ALB: timeout applies to **idle only**, active connections unaffected
- All WebSocket connections closed after **24 hours** (non-customizable)

### Termination Race Condition
- Pod removal from endpoints and preStop hook run **in parallel**
- Fix: `terminationGracePeriodSeconds: 210`, `preStop: sleep 120s`, ~90s for graceful shutdown
- Always set `maxUnavailable: 0` in rolling update strategy

### Spot Nodes
- **60-91% cost savings**, 30s termination notice, **15s grace period** for non-system pods (fixed in GKE)
- `terminationGracePeriodSeconds` must be **<=25s**, preStop max 5s
- PDBs do NOT protect against Spot preemption
- Run critical services on standard nodes with anti-affinity to Spot

## Quick Reference

```
Service Mesh Comparison:
  Sidecar:  2 hops | 50-100MB/pod  | 0.63ms p90  | full L7
  Ambient:  1 hop  | 20-50MB/node  | 73% less CPU | L4 default, L7 opt-in

Timeout Layering (outermost -> innermost):
  LB: 30s -> Gateway: 25s -> Mesh: 20s -> App: 15s

Termination Race Fix:
  terminationGracePeriodSeconds: 210
  preStop: sleep 120s
  maxUnavailable: 0

Spot Node Constraints:
  Termination notice: 30s
  Actual grace period: 15s (GKE-fixed)
  terminationGracePeriodSeconds: <=25s

gRPC LB Solutions:
  L7 LB (ALB/mesh)     -> per-RPC routing
  Client-side (dns:///) -> headless svc + round_robin
  xDS (xds:///)         -> proxyless, control plane driven
```
