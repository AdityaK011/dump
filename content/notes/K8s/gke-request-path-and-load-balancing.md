---
title: "GKE Request Path and Load Balancing"
---

**Every HTTP request to a GKE-hosted application traverses at least seven distinct layers of infrastructure, each with its own failure modes, latency characteristics, and configuration surface.** Understanding this full stack — from DNS resolution at the edge through eBPF packet routing inside the kernel — is what separates routine Kubernetes operation from true platform engineering. This document traces a request end-to-end through GKE, covering DNS resolution, Google's global anycast network, TLS termination, L7 routing, container-native load balancing, and kernel-level packet routing to the correct pod.

## How a request travels from browser to pod and back

The journey of a single HTTPS request through GKE infrastructure involves DNS resolution, Google's global anycast network, TLS termination, L7 routing, container-native load balancing, and finally kernel-level packet routing to the correct pod. Each step introduces latency and has specific failure modes worth understanding.

### DNS resolution and the anycast advantage

When a client resolves your domain, Cloud DNS (or whatever authoritative DNS you use) returns a **single global anycast IP address**. Unlike DNS-based load balancing, Google's Global External Application Load Balancer advertises this IP via BGP from over 200 edge Points of Presence worldwide. The client's ISP routes packets to the topologically nearest Google edge PoP. This means you can set DNS TTLs aggressively high (300s–3600s) because there is only one IP to cache — no multi-IP rotation needed.

At the edge PoP, Google's custom L4 load balancer **Maglev** distributes packets across a pool of Google Front End (GFE) servers using ECMP (Equal-Cost Multi-Path) forwarding. Maglev provides stabilized anycast: if BGP route changes cause a packet to arrive at the wrong PoP, Maglev internally forwards it to the correct site, preserving TCP session state. This eliminates the connection-reset problem that plagues naive anycast deployments.

### The GCP load balancer resource chain

Inside Google's infrastructure, the request flows through a precise chain of Compute Engine resources, each performing one job:

**Global Forwarding Rule** binds the anycast IP and port (typically 443) to a Target HTTPS Proxy. This is the entry point, distributed across GFE locations globally. **Target HTTPS Proxy** terminates the client's TLS connection — this is where SSL certificates attach. It parses the decrypted HTTP request, inspects the Host header and URL path, and appends two IPs to the `X-Forwarded-For` header (the client's IP and the forwarding rule's IP). It then consults the **URL Map**, which is the route-level decision engine mapping `(host, path)` combinations to specific backend services. URL Maps support prefix, exact, and regex path matching, plus advanced features like weighted traffic splitting and header-based routing.

The **Backend Service** represents a collection of backends (Network Endpoint Groups or instance groups) and configures the protocol (HTTP, HTTPS, HTTP/2), timeout (default **30 seconds**), health checks, session affinity, and load balancing algorithm. Health check probes originate from GCP infrastructure IPs (**35.191.0.0/16** and **130.211.0.0/22**) and go directly to pod IPs when using container-native load balancing. Finally, **zonal NEGs** (Network Endpoint Groups) of type `GCE_VM_IP_PORT` contain the actual `PodIP:ContainerPort` endpoints that receive traffic.

### TLS termination at the edge

TLS terminates at the Target HTTPS Proxy, running on GFEs at the network edge. Google's strategy is to terminate SSL as close to the user as possible, then forward requests deeper into their network over long-lived encrypted connections. The load balancer supports **TLS 1.0 through 1.3**, controllable via SSL Policies with predefined profiles (COMPATIBLE, MODERN, RESTRICTED, FIPS_202205) or custom cipher suite selection. TLS 1.3 with **0-RTT early data** for resumed sessions is fully supported, reducing handshake latency. HTTP/3 over IETF QUIC is also available on external Application Load Balancers, providing faster connection initiation and eliminating head-of-line blocking.

A Target HTTPS Proxy can hold multiple SSL certificates for **SNI-based selection**. When using Certificate Manager certificate maps, the selection logic works as: exact hostname match first, then wildcard match (`*.example.com` for first-level subdomains), then fallback to the primary certificate map entry. ECDSA certificates are prioritized over RSA when multiple matches exist.

### Container-native load balancing eliminates the double hop

In VPC-native GKE clusters, the load balancer sends traffic **directly to pod IP addresses**, bypassing node-level routing entirely. Pod IPs come from alias IP ranges (secondary subnet ranges) that are natively routable within the VPC, so GFE proxies and health check infrastructure can reach pods without any intermediate translation.

This is a fundamental improvement over the legacy instance-group approach. The traditional path was: LB -> Node (instance group member) -> kube-proxy iptables DNAT -> Pod (possibly on a different node). This "double hop" caused increased latency, imbalanced load distribution (iptables distributes randomly, not based on pod load), and inaccurate health checking at the node level rather than per-pod. Container-native load balancing with NEGs eliminates all of these problems.

The **GKE NEG controller** watches pod lifecycle events and automatically adds or removes endpoints from the NEG as pods are created, destroyed, or rescheduled. It also manages a critical **pod readiness gate** (`cloud.google.com/load-balancer-neg-ready`) — during rolling updates, a new pod's readiness gate is set to True only after the LB health check confirms the endpoint is healthy. This prevents premature old-pod termination.

### Inside the node: kube-proxy, IPVS, and eBPF

Once a packet reaches a node (either directly via NEG or through NodePort), it must be routed to the correct pod. Three mechanisms exist:

**iptables mode** (legacy default) programs NAT rules for every service and endpoint. Each service gets `KUBE-SVC-*` chains; each endpoint gets `KUBE-SEP-*` chains with probability-based random selection. For 3 backends, the first rule matches with probability 0.333, the second with 0.500, and the third unconditionally. This achieves equal distribution but has **O(n) complexity** — with thousands of services generating tens of thousands of rules, update latency becomes a bottleneck.

**IPVS mode** uses the Linux kernel's IP Virtual Server subsystem with hash tables for **O(1) lookups**. It creates a dummy `kube-ipvs0` interface where ClusterIPs are bound and supports multiple scheduling algorithms (round-robin, least-connections, various hashing). Note that Kubernetes 1.35 is deprecating IPVS mode via KEP-5495.

**GKE Dataplane V2** (built on **Cilium**) replaces kube-proxy entirely with **eBPF programs** attached to kernel hook points (TC ingress/egress, XDP). Service routing happens via eBPF map lookups — **constant-time O(1)** regardless of service count, with no context switches between userspace and kernel. Dataplane V2 is the default for all new GKE Autopilot clusters. It supports Direct Server Return (DSR), where the response pod bypasses the ingress node entirely, and Maglev consistent hashing for external traffic. Scale limits include **260,000 endpoints** across all services, up to **7,500 nodes**, and **200,000 pods per cluster**.

### Pod networking: veth pairs and CNI

Each pod gets its own Linux network namespace. A **virtual Ethernet (veth) pair** acts as a cable between namespaces: one end (`eth0`) lives inside the pod, the other (`vethXXXXXX`) in the host's root namespace. In GKE VPC-native clusters, pod IPs are alias IPs known to the VPC, so standard Linux routing on the node directs traffic — no overlay encapsulation needed. With Dataplane V2, eBPF programs often bypass conventional bridges entirely, managing packet flows directly in the kernel.

When kubelet creates a pod, it invokes the CNI plugin with an `ADD` command. The plugin creates a network namespace, a veth pair, assigns a pod IP from the node's allocated range via IPAM, configures routes inside the pod, and returns the result. On deletion, `DEL` tears everything down. CNI configuration lives at `/etc/cni/net.d/` with plugin binaries at `/opt/cni/bin/`.

### The response path

The response traverses these layers in reverse. The pod writes the HTTP response to its socket, the packet exits through the veth pair to the host namespace, where routing or eBPF programs direct it to the node's physical interface. For NEG-based load balancing, the pod responds directly to the GFE proxy that forwarded the request over the existing persistent connection. The GFE encrypts the response using the TLS session established with the client and sends it from the edge PoP nearest to the client. The key latency optimization: the encrypted leg between GFE and client is minimized because GFEs sit at the edge, while the longer GFE-to-backend leg travels over Google's private backbone.

## Related notes

- [[notes/K8s/kubernetes-services-dns-and-network-policies|Kubernetes Services, DNS, and Network Policies]]
- [[notes/K8s/ingress-vs-gateway-api|Ingress vs Gateway API]]
