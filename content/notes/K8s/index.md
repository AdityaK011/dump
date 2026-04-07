---
title: Kubernetes
---

Notes on Kubernetes internals, GKE platform engineering, networking, autoscaling, and tooling.

## GKE & Load Balancing
- [[notes/K8s/gke-request-path-and-load-balancing|GKE Request Path & Load Balancing]] — How a request travels from browser to pod through GCP's global infrastructure
- [[notes/K8s/ingress-vs-gateway-api|Ingress vs Gateway API]] — Two models for L7 load balancing on GKE, plus TLS certificate management

## Networking
- [[notes/K8s/kubernetes-services-dns-and-network-policies|Services, DNS & Network Policies]] — Service types, CoreDNS internals, ndots, and eBPF-based policy enforcement
- [[notes/K8s/service-mesh-multi-cluster-and-advanced-patterns|Service Mesh, Multi-Cluster & Advanced Patterns]] — Istio sidecar vs ambient, multi-cluster networking, gRPC load balancing, WebSocket handling

## Autoscaling
- [[notes/K8s/kubernetes-autoscaling|Kubernetes Autoscaling]] — HPA, VPA, KEDA, and the minute-by-minute anatomy of a traffic surge

## Fundamentals
- [[notes/K8s/interactive-containers-piping-and-ttys|Interactive Containers, Piping & TTYs]] — How kubectl exec/attach works, Unix pipes, and the TTY abstraction

## Tooling
- [[notes/K8s/kubelens-mcp-server|KubeLens MCP Server]] — Building a Kubernetes MCP server with stateless auth, client caching, and structured tool design
