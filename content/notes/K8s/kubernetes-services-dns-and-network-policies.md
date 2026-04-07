---
title: "Kubernetes Services, DNS, and Network Policies"
---

## Service types and how they actually work

**ClusterIP** allocates a virtual IP from the cluster's Service CIDR (e.g., `10.96.0.0/12`). When a pod sends a packet to this VIP, it hits the PREROUTING chain in netfilter. In iptables mode, the `KUBE-SERVICES` chain matches the destination ClusterIP:port, dispatches to a `KUBE-SVC-*` chain, which uses probability-based random selection across `KUBE-SEP-*` chains that perform the actual DNAT, rewriting the destination to `PodIP:targetPort`. Conntrack ensures reverse SNAT restores the ClusterIP as the source on return traffic. In Dataplane V2, an eBPF program performs the same DNAT from a BPF load balancer map with connection tracking in a BPF conntrack map.

**NodePort** builds on ClusterIP by opening a port in the 30000–32767 range on every node. The `KUBE-NODEPORTS` chain matches incoming packets and jumps to the same `KUBE-SVC-*` chain. The critical configuration here is `externalTrafficPolicy`: with the default `Cluster`, packets are SNATed with the node IP and may hop to another node's pod. With `Local`, traffic only forwards to pods on the receiving node, preserving the source IP but dropping traffic on nodes with no local pods.

**LoadBalancer** builds on NodePort and triggers the cloud controller manager to provision an external load balancer. On GKE, this creates a passthrough Network Load Balancer for L4 traffic. With NEG annotations, the LB can target pod IPs directly, bypassing NodePort entirely.

**ExternalName** creates a CNAME DNS record mapping `<svc>.<ns>.svc.cluster.local` to the specified external hostname. No proxying occurs, no ClusterIP is assigned, and no iptables rules are created. It is DNS-only, useful for integrating external services into the cluster DNS namespace.

## DNS resolution: CoreDNS, ndots, and the hidden query tax

CoreDNS runs as a Deployment in `kube-system`, exposed via the `kube-dns` Service. It resolves cluster service names (`<service>.<namespace>.svc.cluster.local`) to ClusterIPs, returns all pod IPs for headless services, and forwards non-cluster queries upstream.

The default pod `/etc/resolv.conf` contains `options ndots:5` and four search domains. This means any DNS name with fewer than 5 dots triggers search domain expansion before the absolute query. Resolving `api.example.com` (2 dots) generates **4–5 wasted queries** — appending each search domain — before finally trying the absolute name. For high-throughput services making frequent external DNS lookups, this is a significant overhead. Mitigation strategies include using FQDNs with a trailing dot (`api.example.com.`), reducing ndots per-pod via `dnsConfig`, or setting `ndots: 2` for pods that primarily call external services.

**NodeLocal DNSCache** (enabled by default on GKE Autopilot and Standard v1.34.1+) runs a CoreDNS cache on every node listening on link-local `169.254.20.10`. It intercepts DNS queries via iptables, serves cache hits in sub-millisecond latency, and eliminates cross-node DNS hops. This typically reduces CoreDNS load by **70–90%** and avoids the UDP conntrack race condition that can cause intermittent DNS failures.

## Network policies: from pod selectors to eBPF enforcement

Kubernetes NetworkPolicy resources use label selectors to control ingress and egress traffic to pods. Without any policy, all traffic is allowed. Once a policy selects a pod, only explicitly allowed traffic passes. The best practice is to start with a default-deny policy for both ingress and egress, then allow specific flows — but you must also allow DNS egress (port 53 to kube-system), or all service name resolution breaks.

**Calico** enforces policies using its Felix agent on each node, which programs iptables rules and ipsets. It adds roughly 60 iptables rules per node at installation and uses ipsets for efficient matching of large IP sets. **Cilium** (and GKE Dataplane V2) takes a fundamentally different approach: it assigns numeric **identities** to pods based on their labels and enforces policies via eBPF programs. Because identities persist across pod IP changes, this eliminates the dynamic-IP problem that iptables-based enforcement struggles with. Cilium also supports L7 policies (HTTP path/method filtering, gRPC, Kafka) using an embedded Envoy proxy.

## VPC-native networking: why pod IPs are first-class VPC citizens

In GKE VPC-native clusters, pod IPs come from alias IP ranges on the subnet's secondary range. The VPC routing tables natively know how to reach each node's pod CIDR — no custom static routes, no overlay encapsulation. This means pod-to-pod traffic across nodes flows through the VPC directly: source pod -> veth -> host routing table -> node NIC -> VPC network -> destination node -> veth -> destination pod. Pod IPs are accessible from on-prem via VPN/Interconnect, and firewall rules can target pod CIDR ranges directly. This is a prerequisite for container-native load balancing with NEGs.

## Related notes

- [[notes/K8s/gke-request-path-and-load-balancing|GKE Request Path and Load Balancing]]
- [[notes/K8s/ingress-vs-gateway-api|Ingress vs Gateway API]]
