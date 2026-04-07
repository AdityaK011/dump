---
title: "Summary: Istio & Envoy Internals"
---

> **Full notes:** [[notes/K8s/istio-and-envoy-internals|Istio & Envoy Internals -->]]

## Key Concepts

### Architecture: Control Plane (istiod) vs Data Plane (Envoy sidecars)
- **istiod**: single Go binary consolidating Pilot (config translation), Citadel (CA/certs), Galley (validation), xDS server
- Stateless -- reconstructs all state from K8s API on startup; run 2-3 replicas for HA
- **Data plane**: `istio-proxy` container = pilot-agent + Envoy
- pilot-agent: generates bootstrap config, fetches certs via SDS, manages Envoy lifecycle

### xDS Protocol (6 APIs)
- **LDS** (Listener): how to accept connections (bind address, port, filter chains)
- **RDS** (Route): HTTP routing rules (virtual hosts, match rules, rewrites)
- **CDS** (Cluster): upstream groups (LB policy, circuit breakers, TLS context)
- **EDS** (Endpoint): actual pod IP:port for each cluster
- **SDS** (Secret): TLS certs/keys for mTLS, CA bundles
- **ADS** (Aggregated): single gRPC stream multiplexing all types with ordering
- Safe ordering: **CDS -> EDS -> LDS -> RDS** (prevents traffic blackholes)
- Delta xDS (incremental) default since Istio 1.22 -- only changed resources sent

### Sidecar Injection
- Mutating Admission Webhook on pod creation (namespace label `istio-injection=enabled`)
- istiod (port 15017) mutates pod spec: adds istio-init + istio-proxy containers + volumes
- **istio-init**: runs `istio-iptables`, sets up NAT rules, exits (rules persist in namespace)
- Alternative: Istio CNI plugin (no NET_ADMIN needed, eliminates init container race)

### Traffic Interception (iptables)
- **Inbound**: PREROUTING -> ISTIO_INBOUND -> REDIRECT to port 15006 (VirtualInbound)
- **Outbound**: OUTPUT -> ISTIO_OUTPUT -> REDIRECT to port 15001 (VirtualOutbound)
- **Loop prevention**: UID 1337 check -- Envoy traffic bypasses redirect
- Never run app as UID 1337 (traffic bypasses sidecar entirely)
- Envoy reads original destination via `SO_ORIGINAL_DST` getsockopt

### Request Lifecycle (8 Steps)
1. App A sends to `reviews:8080`
2. Kernel iptables redirects to Envoy port 15001
3. Envoy outbound: route lookup -> endpoint selection -> mTLS -> send
4. Packet travels over pod network (CNI)
5. Kernel iptables redirects inbound to Envoy port 15006
6. Envoy inbound: terminate mTLS, verify SPIFFE identity, AuthorizationPolicy
7. Envoy forwards to localhost:8080 (as UID 1337, bypasses iptables)
8. App B processes request

### Ambient Mode (Sidecar-less)
- **ztunnel**: Rust DaemonSet, L4 only (mTLS, L4 authz, telemetry), uses HBONE tunneling
- **Waypoint proxy**: Envoy Deployment per namespace/service, L7 features only when needed
- 90%+ memory reduction vs sidecars; no pod restart needed to mesh
- Traffic flow: Pod -> ztunnel -> (optional waypoint) -> ztunnel -> Pod

### Common Gotchas
- Port naming: must use recognized prefix (`http-`, `grpc-`, etc.) or `appProtocol` field
- App must bind `0.0.0.0`, not `127.0.0.1`
- Init container race: app starts before Envoy -> use `holdApplicationUntilProxyStarts: true`
- istiod down: existing proxies work (cached config), but no updates, cert rotation fails, no new sidecars

## Quick Reference

```
Well-Known Ports:
  15001  Envoy VirtualOutbound (outbound capture)
  15006  Envoy VirtualInbound (inbound capture)
  15000  Envoy Admin API (/config_dump, /clusters, /stats)
  15012  istiod xDS over mTLS (production)
  15017  istiod webhook (injection, validation)
  15021  pilot-agent health check (/healthz/ready)
  15090  Envoy Prometheus metrics

Debugging Commands:
  istioctl proxy-status                          # sync status (SYNCED/STALE)
  istioctl proxy-config listeners deploy/my-app  # LDS
  istioctl proxy-config routes deploy/my-app     # RDS
  istioctl proxy-config clusters deploy/my-app   # CDS
  istioctl proxy-config endpoints deploy/my-app  # EDS
  istioctl proxy-config all deploy/my-app -o json
  kubectl exec deploy/my-app -c istio-proxy -- iptables -t nat -S

Sidecar vs Ambient:
  Sidecar: Envoy per pod, always L7, high resource overhead
  Ambient: ztunnel per node (L4), waypoint per ns (L7), 90%+ less memory
```
