---
title: "Summary: Container Networking Internals"
---

> **Full notes:** [[notes/Networking/container-networking-internals|Container Networking Internals -->]]

## Key Concepts

### Network Namespaces
- Kernel construct providing isolated copy of the entire networking stack
- Each namespace gets its own: interfaces, routing tables, iptables, ARP table, port space, conntrack
- Created via `unshare(CLONE_NEWNET)` or `clone()` with CLONE_NEWNET flag
- Docker containers don't appear in `ip netns list` -- need manual symlink to `/var/run/netns/`
- Some `net.*` sysctls are namespace-aware, others are global (source of production surprises)

### veth Pairs
- Virtual Ethernet pair connected like a crossover cable -- packet in one end appears at the other
- Primary mechanism connecting container namespace to host/bridge
- Trace pairs: `cat /sys/class/net/eth0/iflink` inside container, match on host with `ip link`

### Linux Bridge (docker0)
- L2 virtual switch in kernel; learns MACs, forwards frames like a physical switch
- Same-host container-to-container: pure L2 switching via bridge FDB, no NAT
- Bridge-nf-call-iptables=1: allows iptables to filter bridged traffic

### Container-to-Internet Flow
- Outbound: container -> veth -> bridge -> routing (ip_forward=1) -> iptables MASQUERADE -> host NIC
- MASQUERADE rewrites src IP from container IP to host IP
- Return: conntrack reverse-NAT restores original container dst IP

### kube-proxy iptables DNAT
- Three-layer chain architecture: KUBE-SERVICES -> KUBE-SVC-xxx -> KUBE-SEP-xxx
- Probability-based load balancing: rule k uses probability `1/(N-k+1)` (conditional)
- Each KUBE-SEP does the actual DNAT to pod IP
- **Scaling problem**: 10K services x 10 endpoints = ~100K rules, O(n) linear scan per packet
- Alternatives: IPVS (hash table, O(1)), nftables (verdict maps), Cilium (eBPF)

### Conntrack
- Tracks every connection for stateful firewalling and NAT reversal
- Each entry: original 5-tuple + expected reply 5-tuple (~300 bytes)
- **Table exhaustion**: when full, new connections silently dropped -- intermittent failures
- **UDP conntrack race condition** (K8s issue #56903): two DNS queries same 5-tuple, same time -> one dropped -> 5-second DNS timeout
- Fix: NodeLocal DNSCache, or Cilium (own eBPF conntrack)

### EndpointSlices vs Endpoints
- Endpoints: single object with all IPs; any pod change = full rewrite to all watchers -> O(N*M)
- EndpointSlices: pages of 100; pod change updates one slice -> O(N*100)
- EndpointSlices add: dual-stack, topology hints, per-endpoint conditions (ready/serving/terminating)

### Calico vs Cilium
- **Calico**: Pure L3 routing, no bridge, per-pod /32 routes, proxy ARP, BGP via BIRD, iptables for NetworkPolicy. Works on older kernels (3.10+)
- **Cilium**: eBPF at TC/XDP hooks, replaces kube-proxy entirely, O(1) hash map lookups, own conntrack (no race condition), L7-aware policies, identity-based security. Needs kernel 4.19+

## Quick Reference

```
Conntrack Tuning:
  sysctl net.netfilter.nf_conntrack_max         # default 65536-131072
  sysctl net.netfilter.nf_conntrack_count        # current usage
  conntrack -S | grep insert_failed              # DNS race condition check
  dmesg | grep conntrack                         # table full messages

DNAT probability math (3 endpoints):
  Rule 1: 1/3 = 0.333   Rule 2: 1/2 = 0.500   Rule 3: unconditional

DNS 5s timeout fix priority:
  1. NodeLocal DNSCache (bypass conntrack)
  2. dnsPolicy: Default
  3. NOTRACK for DNS in iptables raw table
  4. TCP for DNS (use-vc in resolv.conf)

Debugging:
  iptables -t nat -L KUBE-SERVICES -n   # check kube-proxy rules
  conntrack -L                           # view connection tracking table
  bridge fdb show dev docker0            # MAC forwarding database
  sysctl net.ipv4.ip_forward             # must be 1

Choose Calico: BGP peering, older kernels, Windows support
Choose Cilium: modern kernels, no iptables overhead, L7 policies, deep observability
```
