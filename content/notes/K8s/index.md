---
title: Kubernetes
---

Notes on Kubernetes internals, GKE platform engineering, networking, autoscaling, and tooling.

## GKE & Load Balancing
- [[notes/K8s/gke-request-path-and-load-balancing|GKE Request Path & Load Balancing]] — How a request travels from browser to pod through GCP's global infrastructure
- [[notes/K8s/ingress-vs-gateway-api|Ingress vs Gateway API]] — Two models for L7 load balancing on GKE, plus TLS certificate management

## Service Mesh & Networking
- [[notes/K8s/kubernetes-services-dns-and-network-policies|Services, DNS & Network Policies]] — Service types, CoreDNS internals, ndots, and eBPF-based policy enforcement
- [[notes/K8s/istio-and-envoy-internals|Istio & Envoy Internals]] — Control plane, xDS protocol, sidecar injection, Envoy request pipeline, threading model, filter chains
- [[notes/K8s/istio-traffic-management-and-security|Istio Traffic Management & Security]] — VirtualService, DestinationRule, mTLS, SPIFFE, AuthorizationPolicy, circuit breaking
- [[notes/K8s/service-mesh-multi-cluster-and-advanced-patterns|Service Mesh, Multi-Cluster & Advanced Patterns]] — Sidecar vs ambient mode, multi-cluster networking, gRPC load balancing, WebSocket handling

## Autoscaling & Scheduling
- [[notes/K8s/kubernetes-autoscaling|Kubernetes Autoscaling]] — HPA, VPA, KEDA, and the minute-by-minute anatomy of a traffic surge
- [[notes/K8s/vpa-eviction-loops|The VPA Eviction Loop]] — Why VPA right-sizing can evict a pod forever, how multiple VPAs on one Deployment jam admission (oldest wins, Off-mode skipped), and why single-replica long-running workers are the classic victims
- [[notes/K8s/server-side-apply-replicas-collapse|SSA Deletes Your Replicas]] — How a CD migration to Server-Side Apply collapsed an HPA-managed fleet to 1 via the `before-first-apply` ownership migration, why the *second* apply breaks not the first, and why a PDB can't protect a replica count

## Storage
- [[notes/K8s/waitforfirstconsumer-bound-vs-attached|Bound Is Not Attached]] — Why `WaitForFirstConsumer` won't keep a PVC `Pending` for a *compatible* node, the four-stage Provision/Bind/Attach/Mount split and who owns each, why PV topology is zone-shaped and machine family is invisible to the scheduler, and how a machine-family + disk-type migration strands a disk on the wrong node

## Observability & Metrics
- [[notes/K8s/when-gauge-sums-lie|When Gauge Sums Lie]] — Three independent ways query-time aggregation corrupts gauge metrics: a wrong declared submission interval silently scaling every value (×0.75, ×0.25), ghost series double-counting churned pods at coarse rollups, and duplicate emitters from leader-elected exporters — plus why ratios cancel the error, why counters don't have the problem, and when to stop reconstructing facts at query time and pre-aggregate at the source

## Operators & Extension APIs
- [[notes/K8s/kubebuilder-controllers-and-webhooks|Kubebuilder Controllers, Webhooks & Extension APIs]] — Reconcile loops, CRD versioning, admission webhooks, extension API servers

## Fundamentals
- [[notes/K8s/interactive-containers-piping-and-ttys|Interactive Containers, Piping & TTYs]] — How kubectl exec/attach works, Unix pipes, and the TTY abstraction

## Tooling
- [[notes/K8s/k8scope-mcp-server|k8scope MCP Server]] — Building a Kubernetes MCP server with OAuth 2.0 proxy, Dynamic Client Registration, StreamableHTTP transport, and structured read-only tools

## Stateful Platforms
Moved to their own topic: [[notes/Elasticsearch/index|Elasticsearch]] — platform anatomy, the write path, and tenancy models.
