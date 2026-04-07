---
title: "Summary: Kubernetes Services, DNS, and Network Policies"
---

> **Full notes:** [[notes/K8s/kubernetes-services-dns-and-network-policies|Kubernetes Services, DNS, and Network Policies -->]]

## Key Concepts

### Service Types
- **ClusterIP**: VIP from Service CIDR, DNAT via iptables/eBPF to PodIP:targetPort
- **NodePort**: Extends ClusterIP, opens port in **30000-32767** on every node
  - `externalTrafficPolicy: Cluster` -- SNATs, may double-hop (default)
  - `externalTrafficPolicy: Local` -- preserves source IP, drops traffic on nodes without local pods
- **LoadBalancer**: Extends NodePort, triggers cloud LB provisioning; with NEG annotations bypasses NodePort
- **ExternalName**: DNS CNAME only, no proxying, no ClusterIP, no iptables rules

### DNS: CoreDNS and the ndots Tax
- CoreDNS in `kube-system`, exposed as `kube-dns` Service
- Default `ndots:5` causes **4-5 wasted queries** for external names (e.g., `api.example.com` has 2 dots)
- Mitigations:
  - Trailing dot FQDNs: `api.example.com.`
  - Reduce ndots per-pod via `dnsConfig`
  - Set `ndots: 2` for external-heavy pods
- **NodeLocal DNSCache**: link-local `169.254.20.10`, reduces CoreDNS load by **70-90%**, sub-ms cache hits, avoids UDP conntrack race condition

### Network Policies
- No policy = all traffic allowed; any policy selecting a pod = default deny for that direction
- Best practice: default-deny ingress+egress, then allowlist (must allow DNS egress to port 53)
- **Calico**: iptables + ipsets (~60 rules per node at install)
- **Cilium/Dataplane V2**: numeric identities per label set, eBPF enforcement, survives pod IP changes, supports L7 policies (HTTP, gRPC, Kafka)

### VPC-Native Networking
- Pod IPs from alias IP ranges (secondary subnet ranges)
- Natively routable in VPC -- no overlay encapsulation
- Reachable from on-prem via VPN/Interconnect
- Prerequisite for container-native load balancing with NEGs

## Quick Reference

```
Service Type Chain:
  ClusterIP (base) -> NodePort (adds node ports) -> LoadBalancer (adds cloud LB)
  ExternalName is standalone (DNS only)

DNS Query Expansion (ndots:5, name with <5 dots):
  api.example.com ->
    api.example.com.<ns>.svc.cluster.local    (miss)
    api.example.com.svc.cluster.local          (miss)
    api.example.com.cluster.local              (miss)
    api.example.com.<domain>                   (miss)
    api.example.com                            (hit!)  = 4 wasted queries

Network Policy Enforcement:
  Calico:  labels -> iptables + ipsets (L3/L4)
  Cilium:  labels -> numeric identity -> eBPF (L3/L4/L7)
```
