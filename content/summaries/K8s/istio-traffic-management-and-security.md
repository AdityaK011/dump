---
title: "Summary: Istio Traffic Management & Security"
---

> **Full notes:** [[notes/K8s/istio-traffic-management-and-security|Istio Traffic Management & Security -->]]

## Key Concepts

### CRD-to-Envoy Mapping
- VirtualService -> RDS (Route Discovery)
- DestinationRule -> CDS (Cluster Discovery)
- Gateway -> LDS (Listener Discovery)
- ServiceEntry -> CDS + EDS
- Sidecar -> LDS + CDS scope
- K8s Services/Endpoints -> EDS

### VirtualService (RDS)
- Defines how requests to a hostname are routed to backends
- `hosts` field: short names, FQDNs, wildcards, IPs
- Match logic: conditions within a single match block are **ANDed**; multiple match blocks are **ORed**
- Rules evaluated top-to-bottom, first match wins
- **Always include a catch-all route** (no match) as last rule -- otherwise 404 from Envoy
- VirtualService takes full ownership of host's routing -- no K8s default fallback

### DestinationRule (CDS)
- Configures how traffic reaches a destination AFTER routing
- **Subsets**: partition endpoints by label selector (typically `version`), each becomes separate Envoy cluster
- LB algorithms: LEAST_REQUEST (default, power-of-2), ROUND_ROBIN, RANDOM, consistent hash
- Consistent hash: by header, cookie, source IP, or query param (ring hash / Maglev)
- Without DestinationRule defining subsets, VirtualService cannot reference `subset: v2`

### Traffic Splitting (Canary)
- VirtualService `weight` field on routes -> Envoy `weighted_clusters`
- DestinationRule subsets map labels to clusters with EDS endpoints
- Gradually shift: 90/10 -> 75/25 -> 50/50 -> 0/100

### Resilience Features
- **Timeouts**: default disabled; set on VirtualService; Envoy returns 504 on expiry
- **Retries**: default 2 attempts; `retryOn`: 5xx, reset, connect-failure, gateway-error
  - `attempts: 3` = 1 original + 2 retries (mapped as `num_retries: 2` in Envoy)
- **Circuit breaking** (DestinationRule): caps maxConnections, maxPendingRequests; returns 503 (UO flag)
- **Outlier detection** (DestinationRule): ejects unhealthy endpoints from LB pool based on consecutive errors
- Circuit breaking = protect upstream from overload; Outlier detection = remove bad endpoints

### Fault Injection
- **Delay**: add latency before forwarding (percentage-based)
- **Abort**: return error code without forwarding
- Happens before upstream -- tests caller's handling of failures
- Gotcha: fault injection + timeout on same route = delayed requests always timeout

### Ingress: Istio Gateway CRD vs Kubernetes Gateway API
- **Istio Gateway CRD**: manual deployment of `istio-ingressgateway`, VirtualService binds by name
- **K8s Gateway API** (recommended since Istio 1.22+): 3-tier role model
  - GatewayClass (infra provider) -> Gateway (cluster operator) -> HTTPRoute (app developer)
  - Auto-provisions Envoy Deployment + Service + ServiceAccount
  - Cross-namespace requires explicit `allowedRoutes` + `ReferenceGrant`
  - Portable across Istio, Envoy Gateway, Cilium
- Ambient mode waypoints also managed via Gateway API (`gatewayClassName: istio-waypoint`)

### ServiceEntry
- Registers external services into Istio's registry for full traffic management
- Without it: passthrough cluster, no retries/timeouts/metrics
- `location`: MESH_EXTERNAL vs MESH_INTERNAL
- `resolution`: DNS, STATIC, or NONE

### Sidecar Resource
- Limits proxy scope to only services the workload actually calls
- `egress.hosts` in `namespace/host` format
- Can reduce Envoy memory 10-50x in large meshes
- One namespace-wide Sidecar per namespace; workload-scoped overrides

### Egress Gateways
- Dedicated Envoy proxy for outbound external traffic
- Use cases: security auditing, TLS origination, network topology constraints

## Quick Reference

```
Canary Deployment Pattern:
  DestinationRule: define subsets (v1, v2 by label)
  VirtualService: weight: [{v1: 90}, {v2: 10}]
  Envoy: weighted_clusters -> separate CDS clusters -> EDS endpoints

Resilience Defaults:
  Timeout: disabled (no timeout)
  Retries: 2 attempts, conditions: connect-failure,refused-stream,unavailable
  Circuit breaker: no limits by default (must configure DestinationRule)

Circuit Breaking vs Outlier Detection:
  Circuit Breaking (connectionPool): caps concurrency -> 503 (UO) immediately
  Outlier Detection: ejects bad endpoints -> exponential backoff ejection

iptables Flow (per pod namespace):
  Outbound: OUTPUT -> ISTIO_OUTPUT -> (uid!=1337) -> REDIRECT to 15001
  Inbound:  PREROUTING -> ISTIO_INBOUND -> REDIRECT to 15006
  Envoy->app: OUTPUT -> (uid==1337) -> RETURN (bypasses redirect)

Match Condition Logic:
  Single match block: conditions ANDed
  Multiple match blocks: ORed
  (header=jason AND uri=/api) OR (header=admin)
```
