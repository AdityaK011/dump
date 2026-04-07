---
title: "Container Networking Internals"
---

This note covers the low-level Linux primitives that make container networking work -- from network namespaces and veth pairs up through kube-proxy iptables chains, conntrack, and CNI plugin architectures. Every abstraction Kubernetes exposes (Service, NetworkPolicy, DNS) ultimately reduces to these kernel mechanisms.

## 1. Linux Network Namespaces

A **network namespace** is a kernel construct that provides an isolated copy of the entire networking stack. Each namespace gets its own:

- Network interfaces (lo, eth0, etc.)
- Routing tables
- iptables/nftables rules
- ARP table
- `/proc/net` entries
- Socket port space (two containers can both bind port 80)
- Conntrack table entries

When Docker or containerd starts a container, the runtime calls `unshare(CLONE_NEWNET)` (or `clone()` with that flag) to create a new network namespace. The container's processes see only the interfaces inside their namespace.

### Hands-On with `ip netns`

```bash
# Create two network namespaces (simulating two containers)
sudo ip netns add container-a
sudo ip netns add container-b

# List namespaces
ip netns list

# Run a command inside a namespace
sudo ip netns exec container-a ip addr
# => only loopback, and it's DOWN

# Bring up loopback inside the namespace
sudo ip netns exec container-a ip link set lo up

# Each namespace has its own routing table
sudo ip netns exec container-a ip route
# => empty -- completely isolated

# Inspect from /proc -- every namespace has a unique inode
ls -la /proc/self/ns/net
ls -la /proc/$(docker inspect -f '{{.State.Pid}}' <container>)/ns/net
```

Docker containers do NOT appear under `ip netns list` by default because Docker does not create symlinks in `/var/run/netns/`. You can create them manually:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' my_container)
sudo ln -s /proc/$PID/ns/net /var/run/netns/my_container
# Now: sudo ip netns exec my_container ip addr
```

### What Namespaces Do NOT Isolate

Network namespaces share the same kernel. This means `sysctl` values under `net.*` are namespace-aware for some parameters (e.g., `net.ipv4.ip_forward`) but global for others (e.g., `nf_conntrack_max` was global until kernel 4.7). This is a common source of production surprises.

---

## 2. veth Pairs (Virtual Ethernet)

A **veth pair** is a pair of virtual network interfaces connected like a crossover cable. A packet sent into one end immediately appears at the other end. They are the primary mechanism for connecting a container's network namespace to the host (or to a bridge).

```
+---------------------------+       +---------------------------+
|   Container Namespace     |       |   Host (root) Namespace   |
|                           |       |                           |
|   eth0 (10.244.1.5)      |       |      vethXXXXXX           |
|        |                  |       |           |               |
+--------|------------------+       +-----------|---------------+
         |                                      |
         +---------- veth pair ----------------+
            (like a virtual cable)
```

### Creating a veth Pair Manually

```bash
# Create the pair
sudo ip link add veth-host type veth peer name veth-container

# Move one end into the container namespace
sudo ip link set veth-container netns container-a

# Configure the container side
sudo ip netns exec container-a ip addr add 10.244.1.5/24 dev veth-container
sudo ip netns exec container-a ip link set veth-container up
sudo ip netns exec container-a ip link set lo up

# Configure the host side
sudo ip addr add 10.244.1.1/24 dev veth-host
sudo ip link set veth-host up

# Now host can ping container
ping 10.244.1.5
```

### How Docker Uses veth Pairs

When Docker starts a container with the default bridge network:

1. Creates a new network namespace for the container
2. Creates a veth pair
3. Moves one end (`eth0`) into the container namespace
4. Attaches the other end (`vethXXXXXX`) to the `docker0` bridge
5. Assigns an IP from the bridge subnet via IPAM
6. Sets the bridge IP as the default gateway inside the container

You can trace veth pairs by matching interface indexes:

```bash
# Inside container: get the peer interface index
cat /sys/class/net/eth0/iflink
# => 42

# On the host: find interface with that index
ip link | grep "^42:"
# => 42: veth8a3b1c2@if41: ...
```

---

## 3. Linux Bridge (docker0 / cbr0)

A **Linux bridge** is a Layer 2 (data link) virtual switch inside the kernel. It learns MAC addresses from frames it receives and forwards frames to the correct port, just like a physical switch.

```
+------------+    +------------+    +------------+
| Container1 |    | Container2 |    | Container3 |
| 10.244.1.2 |    | 10.244.1.3 |    | 10.244.1.4 |
|   eth0     |    |   eth0     |    |   eth0     |
+-----|------+    +-----|------+    +-----|------+
      |                 |                 |
  veth1111          veth2222          veth3333
      |                 |                 |
+-----|-----------------|-----------------|------+
|                  docker0 / cbr0                |
|              (Linux Bridge)                    |
|              10.244.1.1/24                     |
|                                                |
|  MAC Forwarding Table (FDB):                   |
|   aa:bb:cc:01 -> veth1111                      |
|   aa:bb:cc:02 -> veth2222                      |
|   aa:bb:cc:03 -> veth3333                      |
+------------------------------------------------+
                     |
                   eth0 (host NIC)
                 192.168.1.100
```

### Container-to-Container Communication (Same Host)

When Container1 (10.244.1.2) pings Container2 (10.244.1.3):

1. Container1 checks: is 10.244.1.3 in my subnet (10.244.1.0/24)? Yes.
2. Container1 sends an **ARP request**: "Who has 10.244.1.3? Tell 10.244.1.2"
3. The ARP broadcast goes through veth1111 to the bridge
4. The bridge floods the ARP to all attached ports (veth2222, veth3333)
5. Container2 responds with its MAC address
6. The bridge **learns** that MAC aa:bb:cc:02 is on port veth2222
7. Container1 sends the ICMP packet with dst MAC aa:bb:cc:02
8. Bridge looks up the MAC in FDB, forwards directly to veth2222
9. Packet arrives at Container2's eth0

### Inspecting the Bridge

```bash
# Show bridge interfaces
bridge link show

# Show the MAC forwarding database
bridge fdb show dev docker0

# Show ARP table
arp -n

# Show bridge details
brctl show docker0   # (bridge-utils package)
ip link show type bridge
```

Key detail: traffic between two containers on the same bridge **never touches iptables FORWARD chain** if `net.bridge.bridge-nf-call-iptables = 0`. Docker sets this to 1 so that iptables rules can filter inter-container traffic (controlled by `--icc=true/false`).

---

## 4. Container-to-Internet Packet Flow

This is the full path a packet takes from inside a container to the internet and back.

### Outbound Packet Flow (Container to Internet)

```
Container (10.244.1.5)
    |
    | 1. App sends packet: src=10.244.1.5:43210, dst=93.184.216.34:443
    |
    v
eth0 (inside container netns)
    |
    | 2. Packet traverses veth pair
    |
    v
veth7abc (host side, attached to bridge)
    |
    | 3. Bridge receives frame, checks FDB
    |    dst MAC is bridge MAC (default gw) -> goes to bridge itself
    |
    v
docker0 bridge (10.244.1.1)
    |
    | 4. Kernel routing decision: dst 93.184.216.34 is not local
    |    -> forward via default route (host's eth0)
    |    Requires: net.ipv4.ip_forward = 1
    |
    v
iptables FORWARD chain
    |
    | 5. DOCKER-FORWARD / DOCKER-ISOLATION chains evaluated
    |
    v
iptables nat POSTROUTING chain
    |
    | 6. MASQUERADE rule:
    |    -A POSTROUTING -s 10.244.1.0/24 ! -o docker0 -j MASQUERADE
    |    Rewrites src IP: 10.244.1.5 -> 192.168.1.100 (host IP)
    |    Conntrack entry created: tracks this flow for return packets
    |
    v
eth0 (host physical NIC, 192.168.1.100)
    |
    | 7. Packet leaves the host
    |    src=192.168.1.100:43210, dst=93.184.216.34:443
    |
    v
~~~ Internet ~~~
```

### Return Packet Flow (Internet to Container)

```
~~~ Internet ~~~
    |
    | Response: src=93.184.216.34:443, dst=192.168.1.100:43210
    |
    v
eth0 (host NIC)
    |
    | 1. iptables nat PREROUTING: no matching DNAT rule
    |
    v
Conntrack lookup
    |
    | 2. Conntrack finds the entry from the outbound packet
    |    Reverse NAT: dst 192.168.1.100:43210 -> 10.244.1.5:43210
    |
    v
Routing decision -> forward to docker0
    |
    v
docker0 bridge -> veth7abc -> eth0 (container)
    |
    | 3. Packet delivered to container
    |    src=93.184.216.34:443, dst=10.244.1.5:43210
```

### The iptables Rules Docker Creates

```bash
# NAT table - POSTROUTING
sudo iptables -t nat -L POSTROUTING -n -v
# -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE

# Filter table - FORWARD
sudo iptables -L FORWARD -n -v
# -A FORWARD -i docker0 -o docker0 -j ACCEPT      (inter-container)
# -A FORWARD -i docker0 ! -o docker0 -j ACCEPT     (container -> outside)
# -A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

---

## 5. kube-proxy iptables DNAT Chains

kube-proxy in iptables mode programs three layers of chains to implement Kubernetes Services. Understanding these chains is essential for debugging service connectivity.

### Chain Architecture

```
Incoming packet to ClusterIP 10.96.0.100:80
         |
         v
    PREROUTING (nat table)
         |
         v
    KUBE-SERVICES                        (dispatch table)
         |
         | Match: -d 10.96.0.100/32 --dport 80
         v
    KUBE-SVC-XYZABC123                   (per-service chain)
         |
         | Probability-based selection:
         | 33% -> KUBE-SEP-AAA111
         | 33% -> KUBE-SEP-BBB222
         | 34% -> KUBE-SEP-CCC333
         v
    KUBE-SEP-AAA111                      (per-endpoint chain)
         |
         | DNAT: dst -> 10.244.1.5:8080  (actual pod IP)
         v
    Routing decision -> forward to pod
```

### Actual iptables Rules

For a Service `my-svc` (ClusterIP 10.96.0.100:80) with 3 backend pods:

```bash
# Step 1: KUBE-SERVICES dispatches to the per-service chain
-A KUBE-SERVICES -d 10.96.0.100/32 -p tcp --dport 80 \
    -j KUBE-SVC-XYZABC123

# Step 2: KUBE-SVC chain does probability-based load balancing
-A KUBE-SVC-XYZABC123 -m statistic --mode random \
    --probability 0.33333 -j KUBE-SEP-AAA111
-A KUBE-SVC-XYZABC123 -m statistic --mode random \
    --probability 0.50000 -j KUBE-SEP-BBB222
-A KUBE-SVC-XYZABC123 -j KUBE-SEP-CCC333

# Step 3: KUBE-SEP chains do the actual DNAT
-A KUBE-SEP-AAA111 -p tcp -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-BBB222 -p tcp -j DNAT --to-destination 10.244.2.8:8080
-A KUBE-SEP-CCC333 -p tcp -j DNAT --to-destination 10.244.3.2:8080
```

### The Probability Math

The probabilities are **conditional**, not absolute. The chain is evaluated sequentially:

- Rule 1: probability 1/3 = 0.33333 (1 out of 3 remaining)
- Rule 2: probability 1/2 = 0.50000 (1 out of 2 remaining)
- Rule 3: no probability needed (last one, unconditional jump)

Net effect: each endpoint gets exactly 1/3 of traffic. For N endpoints, rule k uses probability `1/(N-k+1)`.

### NodePort and LoadBalancer Flow

For NodePort services, kube-proxy adds an additional entry point:

```bash
# NodePort entry in KUBE-NODEPORTS chain
-A KUBE-NODEPORTS -p tcp --dport 30080 -j KUBE-SVC-XYZABC123

# KUBE-SERVICES jumps to KUBE-NODEPORTS for non-ClusterIP traffic
-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
```

### Scaling Problem with iptables Mode

With 10,000 Services x 10 endpoints each:
- ~100,000 iptables rules
- Rules are stored in a **linked list** -- O(n) linear scan per packet
- Full rule reload is O(n) and blocks packet processing during update
- Each Service update requires rewriting the entire iptables ruleset

This is why **IPVS mode** and **nftables mode** exist as alternatives. IPVS uses a hash table for O(1) lookups regardless of the number of services.

---

## 6. Conntrack (Connection Tracking)

The conntrack subsystem (`nf_conntrack`) tracks every connection flowing through the Linux kernel's netfilter framework. It is what makes stateful firewalling and NAT possible.

### Conntrack Table Structure

Each entry in the conntrack table records:

```
conntrack -L output:

tcp  6 117 TIME_WAIT
    src=10.244.1.5  dst=10.96.0.100  sport=43210 dport=80
    src=10.244.2.8  dst=10.244.1.5   sport=8080  dport=43210
    [ASSURED] mark=0 use=1
    |                                  |
    +-- original direction             +-- reply direction
        (before NAT)                       (expected return packet,
                                            with NAT reversed)
```

The kernel matches return packets against the reply tuple. When a return packet arrives, conntrack recognizes it, and netfilter automatically reverses the NAT without needing another explicit rule.

### How Reverse NAT Works

```
1. Outbound: Pod (10.244.1.5:43210) -> Service (10.96.0.100:80)
   DNAT rewrites dst: 10.96.0.100:80 -> 10.244.2.8:8080
   Conntrack records:
     Original:  src=10.244.1.5:43210  dst=10.96.0.100:80
     Reply:     src=10.244.2.8:8080   dst=10.244.1.5:43210

2. Return:   Backend (10.244.2.8:8080) -> Pod (10.244.1.5:43210)
   Conntrack matches reply tuple
   Reverse DNAT: src 10.244.2.8:8080 -> 10.96.0.100:80
   Pod sees response from 10.96.0.100:80 (the ClusterIP)
```

### Conntrack Table Exhaustion

The conntrack table has a finite size:

```bash
# Check current limit
sysctl net.netfilter.nf_conntrack_max
# Default: 65536 on many systems, 131072 on others

# Check current usage
sysctl net.netfilter.nf_conntrack_count

# Check in real-time
conntrack -C

# View entries
conntrack -L
conntrack -L -p udp
```

When the table is full, **new connections are silently dropped**. The kernel increments `/proc/net/stat/nf_conntrack` `drop` counter and logs `nf_conntrack: table full, dropping packet` to dmesg. This manifests as intermittent connection failures that are incredibly difficult to debug if you are not monitoring `nf_conntrack_count`.

**Production tuning:**

```bash
# Increase the table size (each entry costs ~300 bytes of kernel memory)
sysctl -w net.netfilter.nf_conntrack_max=1048576

# Reduce timeouts for faster entry cleanup
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=3600

# For UDP-heavy workloads (DNS!)
sysctl -w net.netfilter.nf_conntrack_udp_timeout=10
sysctl -w net.netfilter.nf_conntrack_udp_timeout_stream=30
```

### The UDP Conntrack Race Condition (Kubernetes DNS Failures)

This is a well-known issue (upstream Kubernetes issue #56903) that causes intermittent DNS resolution failures, particularly under load.

**The problem:**

```
Pod has two DNS requests nearly simultaneously:
  Thread A sends query to 10.96.0.10:53 (kube-dns ClusterIP)
  Thread B sends query to 10.96.0.10:53 (kube-dns ClusterIP)

Both use the same source socket -> same 5-tuple possible
Both hit the DNAT rules at nearly the same time

       Thread A                          Thread B
          |                                 |
          v                                 v
     DNAT to pod-X                    DNAT to pod-Y
          |                                 |
          v                                 v
   Create conntrack entry           Create conntrack entry
   src=pod:52000                    src=pod:52000
   dst=pod-X:53                     dst=pod-Y:53
          |                                 |
          +------ RACE CONDITION! ----------+
          |
    Only ONE can win the conntrack table insert.
    The loser gets -EEXIST.
    The kernel DROPS that packet silently.
    DNS query times out (5 second delay).
```

**The symptoms:**
- Intermittent 5-second DNS lookup delays
- `conntrack` stat shows `insert_failed` counter incrementing
- More common under high DNS query load
- Only affects UDP (TCP has sequence numbers that differentiate connections)

**Solutions (in order of preference):**

1. **Use NodeLocal DNSCache** -- a DNS caching daemon (DaemonSet) that runs on every node. Pods query the node-local cache over a loopback IP, bypassing conntrack/NAT entirely for cache hits.

2. **Set `dnsPolicy: Default`** and point at node's resolver where appropriate.

3. **Disable conntrack for DNS traffic:**
   ```bash
   iptables -t raw -A PREROUTING -p udp --dport 53 -j NOTRACK
   iptables -t raw -A OUTPUT -p udp --dport 53 -j NOTRACK
   ```

4. **Use TCP for DNS** -- set `use-vc` option in `/etc/resolv.conf` (ndots and options in pod spec). TCP connections have unique sequence numbers so the race does not occur.

---

## 7. EndpointSlices vs Endpoints

### The Problem with Endpoints

In Kubernetes, an `Endpoints` object stores the complete list of IP:port pairs for a Service. Every time a pod is added or removed, the **entire** Endpoints object is rewritten and broadcast to every node via the watch API.

```
Service "my-svc" with 1000 pods:

Endpoints object = 1 resource containing all 1000 IPs

When 1 pod rolls:
  -> Entire Endpoints object (all 1000 IPs) re-serialized
  -> Sent to every node's kube-proxy via API server watch
  -> Every kube-proxy rewrites its iptables rules

With N nodes and M pods:
  Network cost per rolling update = O(N * M)
  A cluster with 5000 nodes and 1000-pod service:
    5000 * 1000 = 5,000,000 endpoint entries transmitted per change
```

This creates API server memory pressure, etcd write amplification, and watch event storms during rolling deployments.

### How EndpointSlices Solve This

EndpointSlices split the endpoints into pages (slices), each containing a maximum of 100 endpoints by default.

```
Service "my-svc" with 1000 pods:

EndpointSlice "my-svc-abc12"  -> endpoints[0..99]
EndpointSlice "my-svc-def34"  -> endpoints[100..199]
EndpointSlice "my-svc-ghi56"  -> endpoints[200..299]
...
EndpointSlice "my-svc-xyz99"  -> endpoints[900..999]

When 1 pod rolls:
  -> Only 1 EndpointSlice (with ~100 IPs) is updated
  -> Only that slice is broadcast to watchers
  -> O(N * 100) instead of O(N * M)
```

### Additional Benefits of EndpointSlices

- **Dual-stack support**: EndpointSlices have an `addressType` field (IPv4, IPv6, FQDN), making dual-stack a first-class concept
- **Topology hints**: Each endpoint can carry topology information (zone, node) for topology-aware routing
- **Per-endpoint conditions**: Each endpoint has individual `ready`, `serving`, and `terminating` conditions rather than a single binary "is this address in the list or not"

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-svc-abc12
  labels:
    kubernetes.io/service-name: my-svc
addressType: IPv4
endpoints:
  - addresses: ["10.244.1.5"]
    conditions:
      ready: true
      serving: true
      terminating: false
    zone: "us-central1-a"
    nodeName: "node-01"
    targetRef:
      kind: Pod
      name: my-pod-abc
      namespace: default
ports:
  - name: http
    port: 8080
    protocol: TCP
```

### Endpoints vs EndpointSlices Summary

| Aspect               | Endpoints            | EndpointSlices              |
|-----------------------|---------------------|-----------------------------|
| Max endpoints/object  | ~1000 (soft limit)  | 100 (configurable)          |
| Update granularity    | Entire object       | Single slice (~100 entries) |
| Dual-stack            | Separate objects    | addressType field           |
| Topology info         | None                | Zone, node hints            |
| API cost per change   | O(N * M)            | O(N * slice_size)           |
| Default since         | v1.0                | GA in v1.21, default in v1.22+ |

---

## 8. Calico vs Cilium Networking

### Calico: Pure L3 Routing (No Bridge, No Encapsulation)

Calico takes a fundamentally different approach from Docker's bridge model. It does not use a bridge or overlay network. Instead, it programs **point-to-point routes** on every host so that pod IPs are directly routable.

```
Node 1 (192.168.1.10)                  Node 2 (192.168.1.11)
+-----------------------------+        +-----------------------------+
|                             |        |                             |
| Pod-A          Pod-B        |        | Pod-C          Pod-D        |
| 10.244.1.2     10.244.1.3  |        | 10.244.2.2     10.244.2.3  |
|   |              |          |        |   |              |          |
| caliXXX1      caliXXX2     |        | caliYYY1      caliYYY2     |
|   |              |          |        |   |              |          |
|   +-- routing table: ------+        |   +-- routing table: ------+
|                             |        |                             |
| 10.244.1.2 dev caliXXX1    |        | 10.244.2.2 dev caliYYY1    |
| 10.244.1.3 dev caliXXX2    |        | 10.244.2.3 dev caliYYY2    |
| 10.244.2.0/24 via           |        | 10.244.1.0/24 via           |
|   192.168.1.11 dev eth0    |        |   192.168.1.10 dev eth0    |
+-----------------------------+        +-----------------------------+
              |                                     |
              +--- Physical Network (L2/L3) --------+
                   BGP peering via BIRD daemon
```

**Key characteristics of Calico:**

- **No bridge** -- each pod veth is a point-to-point link with proxy ARP. The host responds to ARP for any pod IP (because `proxy_arp` is enabled on the veth). No bridge forwarding database, no ARP flooding.
- **No encapsulation by default** -- pod IPs are directly routable. On-premises, Calico uses **BGP** (via the BIRD daemon) to advertise pod CIDR routes to the physical network. In cloud, it can use the cloud's routing tables or fall back to VXLAN/IPIP encapsulation.
- **NetworkPolicy via iptables/eBPF** -- Calico implements Kubernetes NetworkPolicy (and its own more expressive CRDs like GlobalNetworkPolicy) using iptables rules programmed per-endpoint or via eBPF.
- **Encapsulation options** -- IPIP or VXLAN can be enabled for cross-subnet traffic when the underlying network does not support BGP or cannot route pod CIDRs natively (common in cloud VPCs).

### Cilium: eBPF-Native Networking

Cilium replaces kube-proxy, iptables, and parts of the kernel networking stack with **eBPF programs** attached at strategic hook points in the kernel.

```
+-----------------------------------------------------+
|                      Node                            |
|                                                      |
|  Pod-A                          Pod-B                |
|  10.244.1.2                     10.244.1.3           |
|    |                              |                  |
|  lxcXXX1                       lxcXXX2               |
|    |                              |                  |
|    v                              v                  |
|  +-------------------------------------------+       |
|  | eBPF programs (TC hook / XDP)             |       |
|  |                                           |       |
|  | - Service DNAT (replaces kube-proxy)      |       |
|  | - Network Policy enforcement              |       |
|  | - L7 policy (HTTP, gRPC, Kafka)           |       |
|  | - Load balancing (Maglev consistent hash) |       |
|  | - Conntrack (eBPF map, NOT nf_conntrack)  |       |
|  | - Bandwidth management                    |       |
|  +-------------------------------------------+       |
|                      |                               |
|                    eth0 / VXLAN / Geneve             |
+-----------------------------------------------------+
```

**Key characteristics of Cilium:**

- **Replaces kube-proxy entirely** -- Service load balancing is done in eBPF at the TC (traffic control) or XDP (eXpress Data Path) hook, before the packet enters the normal kernel networking stack. No iptables rules for services at all.
- **O(1) service lookups** -- eBPF uses hash maps, so service/endpoint lookup is constant-time regardless of cluster size. No linked-list traversal like iptables.
- **Bypasses conntrack (nf_conntrack)** -- Cilium maintains its own connection tracking in eBPF maps, avoiding the UDP conntrack race condition entirely and eliminating conntrack table exhaustion as a failure mode.
- **L7-aware policies** -- eBPF programs can inspect L7 protocol headers (HTTP method/path, gRPC service, Kafka topic) and enforce policies at the network level without a sidecar proxy.
- **Identity-based security** -- Cilium assigns a numeric identity to each endpoint based on its labels. Policies reference identities, not IPs. The eBPF datapath does identity lookups from packet metadata without needing per-pod iptables rules.
- **Overlay options** -- VXLAN or Geneve encapsulation for cross-node traffic, or native routing mode (similar to Calico) when the underlying network supports it.

### When to Use Which

| Consideration                  | Calico                              | Cilium                              |
|-------------------------------|--------------------------------------|--------------------------------------|
| Kernel requirement            | Works on older kernels (3.10+)      | Needs 4.19+, best on 5.10+          |
| BGP integration               | First-class (BIRD daemon)           | Limited (Cilium BGP CP, newer)       |
| Performance at scale           | Good; iptables can slow at 10k+ svc| Excellent; eBPF hash maps O(1)       |
| L7 policy                     | Needs sidecar (Envoy/Istio)         | Built-in via eBPF                    |
| Operational complexity         | Well-understood, mature             | Steeper learning curve, newer        |
| Conntrack race condition       | Affected (uses nf_conntrack)        | Not affected (own conntrack in eBPF) |
| Windows node support           | Yes (HNS)                           | No                                   |
| Encryption                     | WireGuard or IPsec                  | WireGuard or IPsec (transparent)     |
| Observability                  | Flow logs, Prometheus metrics       | Hubble (deep eBPF-based flow viz)    |

**Rule of thumb:**
- Choose Calico if you need BGP peering with physical network infrastructure, run older kernels, or need Windows support.
- Choose Cilium if you run modern kernels and want to eliminate iptables overhead, need L7 network policies, or want deep observability without sidecars.

---

## 9. Interview Prep: Container Networking Q&A

### Q1: "Explain how two containers on the same host communicate."

**A:** Each container runs in its own network namespace with a veth pair connecting it to a Linux bridge (e.g., docker0). When Container A sends a packet to Container B:
1. Container A checks its routing table. The destination is on the same subnet, so it ARPs for Container B's MAC.
2. The ARP broadcast goes through A's veth to the bridge.
3. The bridge floods the ARP to all ports. Container B responds.
4. The bridge learns B's MAC-to-port mapping in its forwarding database.
5. A sends the packet with B's destination MAC. The bridge does a direct L2 forward to B's veth.

No routing, no NAT. It is pure L2 switching inside the kernel.

---

### Q2: "Walk me through what happens when a pod makes an HTTP request to a ClusterIP service."

**A:** Let's say pod at 10.244.1.5 connects to Service at 10.96.0.100:80, which has three backend pods.

1. **DNS resolution** -- the pod's `/etc/resolv.conf` points to kube-dns (10.96.0.10). The pod resolves `my-svc.default.svc.cluster.local` to 10.96.0.100.
2. **TCP SYN sent** -- src=10.244.1.5:random, dst=10.96.0.100:80
3. **iptables PREROUTING/OUTPUT** -- the packet hits the nat table. The KUBE-SERVICES chain matches on `-d 10.96.0.100 --dport 80` and jumps to KUBE-SVC-XXX.
4. **Probability-based DNAT** -- KUBE-SVC-XXX has three rules with probabilities 0.333, 0.5, and 1.0. One is selected. Say it jumps to KUBE-SEP-BBB which DNATs the destination to 10.244.2.8:8080.
5. **Conntrack entry created** -- original tuple: src=10.244.1.5, dst=10.96.0.100:80. Reply tuple: src=10.244.2.8:8080, dst=10.244.1.5.
6. **Routing** -- kernel looks up 10.244.2.8 in the routing table. If same node, forward via bridge/veth. If different node, forward to the node via the overlay or direct route (depending on CNI).
7. **Return path** -- backend pod responds. Conntrack matches the reply tuple and reverses the DNAT: src is rewritten from 10.244.2.8:8080 back to 10.96.0.100:80. The client pod sees the response coming from the ClusterIP.

---

### Q3: "We're seeing intermittent 5-second DNS timeouts in Kubernetes. What would you investigate?"

**A:** The most likely culprit is the **UDP conntrack race condition**. Here's my investigation:

1. **Check conntrack insert failures:**
   ```bash
   conntrack -S | grep insert_failed
   ```
   If `insert_failed` is incrementing, this confirms the race condition.

2. **Check conntrack table utilization:**
   ```bash
   sysctl net.netfilter.nf_conntrack_count
   sysctl net.netfilter.nf_conntrack_max
   ```
   If count is near max, we have table exhaustion (a different but related problem).

3. **Check dmesg for drops:**
   ```bash
   dmesg | grep conntrack
   ```

4. **Immediate fix:** Deploy NodeLocal DNSCache. This runs a DNS caching agent on each node that pods query over a local link address, bypassing DNAT/conntrack entirely for cached entries.

5. **Longer-term:** Consider Cilium, which uses its own eBPF-based conntrack and is not affected by this race.

---

### Q4: "Why does kube-proxy iptables mode not scale well? What are the alternatives?"

**A:** iptables rules are stored as a **linked list** in the kernel. For every packet, the kernel walks the chain linearly until it finds a match. With 10,000 services and 10 endpoints each, that's ~100k rules and potentially thousands of comparisons per packet.

Additionally, kube-proxy uses `iptables-restore` to atomically replace the entire ruleset on every Service/Endpoints change. This is an O(n) operation that blocks packet processing during the swap.

Alternatives:
- **IPVS mode**: Uses a kernel-space hash table. O(1) lookup regardless of service count. Supports multiple load balancing algorithms (round-robin, least-connections, etc.). Has been stable since Kubernetes 1.11.
- **nftables mode** (Kubernetes 1.29+): Uses nftables verdict maps for O(1) lookups. More modern kernel subsystem than iptables. Not yet as battle-tested.
- **Cilium kube-proxy replacement**: Eliminates kube-proxy entirely. eBPF programs handle service dispatch with hash map lookups.

---

### Q5: "Explain the difference between Calico and Cilium at the datapath level."

**A:** Calico uses traditional Linux networking primitives. Each pod gets a veth pair, but instead of a bridge, Calico programs **per-pod /32 routes** in the host routing table with `proxy_arp` enabled on the veth. Cross-node traffic is either directly routed (via BGP) or encapsulated (IPIP/VXLAN). Network policies are enforced via iptables rules programmed per-endpoint.

Cilium attaches **eBPF programs** to the TC (traffic control) hook on each pod's veth interface. These programs handle service load balancing (replacing kube-proxy's iptables chains), network policy enforcement, and connection tracking -- all before the packet enters the kernel's normal networking stack. This means packets can be redirected between pods on the same node without ever hitting the routing table or iptables.

The practical difference: Calico's datapath uses well-understood kernel primitives that work on any kernel. Cilium's datapath bypasses most of those primitives for better performance and programmability but requires a modern kernel (4.19+ minimum, 5.10+ recommended).

---

### Q6: "What is a conntrack entry and why should I care about nf_conntrack_max?"

**A:** A conntrack entry is a kernel data structure that tracks the state of a network connection (TCP, UDP, or ICMP). Each entry records the original direction 5-tuple and the expected reply 5-tuple, enabling stateful firewalling and NAT reversal.

`nf_conntrack_max` caps the number of simultaneous entries. Each entry uses about 300 bytes of non-swappable kernel memory. When the table fills up, the kernel silently drops new connections -- no RST, no ICMP unreachable, just a timeout on the client side.

In Kubernetes, every pod-to-service connection, every DNS query (UDP), and every health check probe creates a conntrack entry. A busy node can easily exhaust the default limit (65k-131k). Monitoring `nf_conntrack_count` vs `nf_conntrack_max` is critical. If count exceeds ~75% of max, increase the limit or reduce timeouts for short-lived connections.

---

### Q7: "How do EndpointSlices improve over Endpoints?"

**A:** With Endpoints, every Service has a single Endpoints object containing all pod IPs. When any pod changes, the entire object is re-serialized and pushed to all watchers (every kube-proxy instance). For a Service with 1000 pods and 5000 nodes, that's 5 million endpoint entries transmitted per rolling update event.

EndpointSlices break the list into chunks of 100. A pod change only triggers an update to one slice. The API server watch event is 100 entries, not 1000. The total data transmitted drops by 10x. Additionally, EndpointSlices add per-endpoint topology hints (enabling topology-aware routing), dual-stack address types, and granular readiness conditions (ready, serving, terminating).

---

### Q8: "A pod can reach other pods but cannot reach any ClusterIP service. What do you check?"

**A:** This points to a kube-proxy or iptables/DNAT issue, not a CNI or routing issue (since pod-to-pod works).

1. **Is kube-proxy running?** Check `kubectl get pods -n kube-system -l k8s-app=kube-proxy`. Look for CrashLoopBackOff or OOMKilled.
2. **Are iptables rules present?** Run `iptables -t nat -L KUBE-SERVICES -n` on the node. If the chain is empty or missing, kube-proxy is not programming rules.
3. **Is the Service correctly configured?** Check `kubectl get endpoints <svc>` -- are there endpoints? A Service with no selector or no matching pods will have no endpoints.
4. **Is ip_forward enabled?** `sysctl net.ipv4.ip_forward` must be 1. Some hardened base images disable this.
5. **Are FORWARD chain policies blocking?** `iptables -L FORWARD -n -v` -- check for DROP policies or missing ACCEPT rules for pod CIDRs.
6. **DNS vs IP?** If the pod cannot resolve the service name but can reach the ClusterIP by IP, the issue is DNS (coredns), not kube-proxy.

---

## Related Notes

- [[notes/Networking/tcp-socket-internals|TCP Socket Internals]]
- [[notes/K8s/kubernetes-services-dns-and-network-policies|Services, DNS & Network Policies]]
- [[notes/K8s/gke-request-path-and-load-balancing|GKE Request Path & Load Balancing]]
