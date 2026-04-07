---
title: "Summary: Ingress vs Gateway API"
---

> **Full notes:** [[notes/K8s/ingress-vs-gateway-api|Ingress vs Gateway API -->]]

## Key Concepts

### GKE Ingress Controller
- Runs on GKE master nodes (not user project)
- Creates: Forwarding Rule, Target HTTP(S) Proxy, URL Map, Backend Service(s), Health Checks, NEGs, Firewall Rules
- Initial provisioning: **several minutes**
- At scale: 20 Ingress x 20 NEGs = **30+ min** reconciliation; 100+ Ingress = **6+ hours**

### Critical Annotations
- `cloud.google.com/neg: '{"ingress": true}'` -- container-native LB
- `cloud.google.com/backend-config` -- references BackendConfig CRD
- `cloud.google.com/app-protocols` -- backend protocol (HTTP, HTTPS, HTTP/2)

### BackendConfig (attached to Service)
- `timeoutSec` (default 30s)
- `connectionDraining.drainingTimeoutSec` -- set to 1.5-2x longest request
- `healthCheck` -- override defaults
- `securityPolicy.name` -- Cloud Armor reference
- `sessionAffinity` -- CLIENT_IP, GENERATED_COOKIE, HEADER_FIELD
- **Always use API version `cloud.google.com/v1`** -- v1beta1 has bug that removes Cloud Armor policies

### FrontendConfig (attached to Ingress)
- `sslPolicy` -- TLS versions and ciphers
- `redirectToHttps` -- automatic HTTP->HTTPS redirect
- External Ingress only (not internal)

### Gateway API -- The Replacement
- Separates concerns into 3 resources:
  - **GatewayClass** (infra provider) -- defines LB capabilities
  - **Gateway** (cluster operator) -- listeners, TLS
  - **HTTPRoute** (app developer) -- routing rules
- Multiple HTTPRoutes from different namespaces can share one Gateway (multi-tenant)
- GKE Gateway Controller is Google-hosted (more scalable than Ingress controller)
- Merges all HTTPRoutes into a single URL Map
- Supports Certificate Manager (Ingress does not)
- **Google-recommended** over Ingress

### GKE GatewayClasses
- `gke-l7-global-external-managed` -- global external ALB
- `gke-l7-regional-external-managed` -- regional external ALB
- `gke-l7-rilb` -- regional internal ALB
- Multi-cluster variants available

### TLS Certificate Options
- **ManagedCertificate CRD**: simplest, auto-provisioned DV certs, **30-60 min** to provision, no wildcards, Ingress only
- **cert-manager**: ACME protocol, HTTP01 or DNS01 challenges, supports wildcards via DNS01
  - Use `http01-edit-in-place: "true"` annotation on GKE to prevent dual-IP issue
- **Certificate Manager (GCP)**: most scalable, thousands of entries per map, DNS authorization supports wildcards, shareable across LBs

## Quick Reference

```
Ingress Model (monolithic):
  Ingress resource = infra + routing + TLS (one persona, annotations for extras)

Gateway API Model (separated):
  GatewayClass  ->  Gateway  ->  HTTPRoute
  (provider)       (operator)    (developer)

TLS Certificate Decision Tree:
  Simple, Ingress only?        -> ManagedCertificate CRD
  Need wildcards?              -> cert-manager (DNS01) or Certificate Manager
  Scale (1000s of certs)?      -> Certificate Manager
  Gateway API?                 -> Certificate Manager (ManagedCert not supported)

SNI Selection Order:
  Exact hostname -> Wildcard (*.example.com) -> Primary entry fallback
  ECDSA prioritized over RSA
```
