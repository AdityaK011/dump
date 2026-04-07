---
title: "Service Mesh, Multi-Cluster, and Advanced Patterns"
---

## System design decisions that determine how your platform behaves at scale

### Why container-native load balancing is non-negotiable

The legacy instance-group approach creates unmanaged instance groups containing all nodes, regardless of whether they host relevant pods. The LB picks a node, kube-proxy picks a pod (possibly on a different node), and you get two hops of load balancing with random selection at both levels. This leads to imbalanced load, inaccurate health signals (node-level, not pod-level), SNAT obscuring client IPs, and Compute Engine API conflicts since VMs cannot belong to more than one load-balanced instance group.

Container-native load balancing with NEGs eliminates all of these: **single-hop data path** (LB→Pod), per-pod health checking, optimal load distribution using the LB's algorithm directly against pods, source IP preservation, and native connection draining. Every production GKE cluster running HTTP workloads should use VPC-native networking with NEG-backed services.

### Regional vs global, internal vs external: choosing the right load balancer

**Global external Application LB** is the default choice for internet-facing HTTP services. Anycast IP from 200+ edge PoPs, automatic cross-region failover, Premium Tier backbone routing, and integration with Cloud Armor, CDN, and managed certificates. **Regional ALBs** (Envoy-based) cost less, work with Standard Tier, and suit compliance requirements mandating regional data residency. **Internal ALBs** (always Envoy-based, regional) serve VPC-internal traffic for service-to-service communication — they require a proxy-only subnet and don't support FrontendConfig, CDN, or managed certificates.

For L4 traffic, **passthrough Network Load Balancers** preserve the client IP and support TCP, UDP, ESP, ICMP, and GRE. They use Direct Server Return, so responses bypass the LB entirely. **Proxy-based Network Load Balancers** (TCP Proxy, SSL Proxy) terminate connections and can be global, but backends see the LB's IP instead of the client's.

### Multi-cluster networking: when one cluster isn't enough

Multi-cluster architectures become necessary for regional HA, blast radius isolation, regulatory locality, or scaling beyond single-cluster limits (~15,000 nodes max). GKE offers several approaches. **Multi-cluster Gateway** (preferred) implements standard Gateway API with multi-cluster GatewayClasses like `gke-l7-global-external-managed-mc`. It uses `ServiceImport` backends, with a config cluster hosting Gateway and HTTPRoute resources. **Multi-Cluster Services (MCS)** provides the discovery layer: exporting a service creates a `ServiceImport` in all fleet clusters, addressable at `<service>.<namespace>.svc.clusterset.local`.

**Cloud Service Mesh** (formerly Anthos Service Mesh) adds Istio-based multi-cluster service discovery with locality-aware load balancing. Endpoint discovery is automatically configured when clusters join the same Fleet. **Karmada** (CNCF incubation) takes a different approach as a multi-cloud orchestrator: a dedicated control plane distributes standard Kubernetes manifests to member clusters via Propagation Policies, supporting strategies like `Duplicated` (full copy everywhere) or `Divided` (split replicas with static or dynamic weights).

### Service mesh: sidecar vs ambient mode and when the complexity pays off

**Istio sidecar mode** injects an Envoy proxy into every pod via mutating webhook. iptables rules redirect all inbound traffic to Envoy's port 15006 and outbound to port 15001. Every request traverses two proxy hops (source sidecar → destination sidecar), adding roughly **0.63ms p90 latency** and costing approximately **50–100MB memory and 0.20 vCPU per sidecar**.

**Istio ambient mode** (GA since Istio 1.22 for single-cluster) splits the data plane into two layers. **ztunnel** (written in Rust) runs as a DaemonSet — one per node — handling L4 concerns: mTLS encryption, TCP authorization, L4 telemetry, and SPIFFE identity. It uses HBONE (HTTP-Based Overlay Network Enveloping) to tunnel TCP through HTTP/2 CONNECT with mTLS. Memory footprint: **~20–50MB per ztunnel instance** versus 50–100MB per sidecar multiplied by N pods. **Waypoint proxies** (full Envoy, deployed as Kubernetes Deployments per namespace) handle L7 concerns when needed: HTTP routing, VirtualServices, L7 authorization. Only one L7 proxy hop per request versus two in sidecar mode. Solo.io benchmarks show ambient mode uses **73% less CPU** than sidecars.

Both modes use SPIFFE identities (`spiffe://cluster.local/ns/<namespace>/sa/<service-account>`) with X.509 certificates issued by istiod and automatically rotated (default TTL ~24 hours). In ambient mode, a compromised pod cannot leak mesh keys because they reside in the ztunnel, not in the pod.

A service mesh is worth the complexity when you need: mTLS everywhere (zero-trust), complex traffic management (canary releases, fault injection), multi-cluster service discovery, centralized observability, or L7 authorization policies. Start with ambient mode for lower operational overhead.

### Rate limiting and circuit breaking across the stack

**Cloud Armor** operates at the edge before traffic reaches backends. It supports `throttle` (allow up to threshold, deny excess) and `rate-based-ban` (ban the key after threshold exceeded) actions, keyed by IP, HTTP header, cookie, path, or combinations. Limits are per backend service.

**Envoy/Istio circuit breaking** uses two mechanisms. Connection pool limits (in DestinationRule) cap `maxConnections`, `http1MaxPendingRequests`, `http2MaxRequests`, and `maxRetries` — exceeding any returns **503 with response flag `UO`** (upstream overflow). Outlier detection ejects unhealthy endpoints based on consecutive 5xx errors, with exponential backoff (`baseEjectionTime × number_of_ejections`) and a panic threshold (default 50% healthy) that disables ejection to prevent total service loss.

Timeouts should be configured from outermost to innermost, each layer slightly shorter: LB (30s) → Ingress/Gateway (25s) → mesh/VirtualService (20s) → application client (15s). Retry policies must consider idempotency — only retry on idempotent operations, and use Envoy's `maxRetries` connection pool limit to cap concurrent retries and prevent retry storms.

### Observability: tracing a request across every layer

The key to end-to-end observability is **trace context propagation**. GCP Application Load Balancers generate `X-Cloud-Trace-Context` headers. Applications must forward W3C TraceContext headers (`traceparent`, `tracestate`) through all downstream calls. Cloud Logging allows filtering by trace ID across LB logs, Kubernetes events, and application logs. Cloud Trace links automatically to Cloud Logging entries.

**Google Cloud Managed Service for Prometheus** provides fully managed Prometheus with 24-month retention and global PromQL querying. GKE's managed OpenTelemetry provides an in-cluster OTLP endpoint (`opentelemetry-collector.gke-managed-otel.svc.cluster.local:4318`) that routes traces to Cloud Trace, metrics to Managed Prometheus, and logs to Cloud Logging. For Datadog, the Agent DaemonSet collects node/container metrics, network performance monitoring via eBPF, and accepts APM traces on port 8126 with built-in OTLP receiver support.

## Advanced networking topics that catch teams by surprise

### gRPC load balancing: why L4 breaks everything

gRPC runs over HTTP/2, which multiplexes all requests over a **single long-lived TCP connection**. A standard Kubernetes ClusterIP Service performs L4 (connection-level) load balancing — it routes the single TCP connection to one pod, and 100% of subsequent RPCs go to that pod while others sit idle. This is empirically verified: 50 gRPC requests through a standard Service result in one pod receiving all traffic.

Three solutions exist. **L7 load balancing** with GCP's Application Load Balancer (configured with HTTP/2 backend protocol) or a service mesh (Istio/Linkerd) routes individual RPCs to different backends — Istio benchmarks show 50 requests evenly distributed across 5 pods. **Client-side load balancing** with headless services (`clusterIP: None`) causes DNS to return all pod IPs; the gRPC client resolves them and round-robins RPCs directly. This requires the `dns:///` scheme (triple slash) and `round_robin` policy in client config. **xDS-based look-aside load balancing** uses the `xds:///` resolver with a centralized control plane (GCP Traffic Director or custom xDS server) that provides endpoint lists, load assignments, and health status to proxyless gRPC clients.

For client-side approaches, set `GRPC_MAX_CONNECTION_AGE` on servers to force periodic re-resolution as pods scale. Implement native gRPC health checks (Kubernetes 1.24+) with `readinessProbe.grpc.port` in pod specs.

### WebSocket handling: timeout configuration is critical

GCP HTTP(S) Load Balancers support WebSocket natively with no special configuration. On upgrade, the client sends `Connection: Upgrade` and `Upgrade: websocket` headers; the LB forwards them, the backend responds with 101 Switching Protocol, and the LB proxies bidirectional traffic.

The critical difference is timeout behavior. On the **Classic Application LB**, the backend service timeout (default 30s) is a **hard cutoff** — WebSocket connections are terminated after 30 seconds regardless of activity. You must increase it to match your session length. On **Global/Regional External ALBs**, the timeout applies only to **idle** connections — active WebSocket connections with data flowing are unaffected. All WebSocket connections are automatically closed after **24 hours** (non-customizable). Configure keepalive pings at the application layer to prevent idle timeouts, enable session affinity for reconnecting clients, and set `connectionDraining.drainingTimeoutSec` in BackendConfig to allow graceful WebSocket closure during deployments.

### Connection draining and the termination race condition

When a pod is terminated, two things happen **in parallel**: the pod is removed from Service endpoints (triggering kube-proxy updates, Ingress controller updates, NEG endpoint removal) and the preStop hook executes. After preStop completes, SIGTERM is sent to PID 1 in each container. After `terminationGracePeriodSeconds` (default 30s), SIGKILL forces termination.

The race condition: the pod may receive SIGTERM before the load balancer has fully removed it from rotation, causing traffic to hit a shutting-down pod. The fix is a **preStop hook with a sleep**: GCP's official recommendation for NEG-based load balancing is `terminationGracePeriodSeconds: 210` with `preStop: sleep 120s`, leaving ~90s for graceful application shutdown after SIGTERM. The preStop delay gives the NEG controller time to remove the endpoint, the LB health check to detect the absence, and in-flight requests to complete via the configured `drainingTimeoutSec`. Always set `maxUnavailable: 0` in your rolling update strategy to ensure new pods are healthy in the LB before old ones are removed.

### Spot nodes: 15-second grace periods change everything

GKE Spot VMs offer 60–91% cost savings but have no maximum lifetime guarantee (preemptible VMs had a 24-hour cap). On preemption, the VM receives a **30-second termination notice**. GKE's graceful node shutdown grants non-system pods only **15 seconds** of grace period — this is fixed and immutable in GKE. This means `terminationGracePeriodSeconds` should be **≤25 seconds** for Spot workloads, and preStop hooks must be very brief (5 seconds max).

PodDisruptionBudgets do **not** protect against Spot preemptions (involuntary disruptions). The practical strategy is: run critical user-facing services on standard node pools with node affinity excluding Spot (`cloud.google.com/gke-spot: DoesNotExist`), use Spot for fault-tolerant batch processing or stateless workers with topology spread constraints and pod anti-affinity to prevent single-point-of-failure. Monitor for `statusDetails: backend_timeout` in LB logs to detect preemption-related traffic loss, and tune health check parameters (`checkIntervalSec: 5, unhealthyThreshold: 1`) to minimize the **10-second default detection window**.

### IPv4/IPv6 dual-stack: getting ready for the transition

GKE dual-stack clusters assign both IPv4 and IPv6 addresses to nodes, pods, and services. Each pod receives two IPs — one from the Pod CIDR, one from the IPv6 Pod range — both natively routable within the VPC. Services use `ipFamilyPolicy` (SingleStack, PreferDualStack, RequireDualStack) and `ipFamilies` ([IPv4], [IPv6], or both) to control stack behavior. The first entry in `ipFamilies` determines `.spec.clusterIP`.

IPv6 requires Premium Tier and VPC-native clusters with dual-stack subnets. Subnets can have either ULA (Unique Local Addresses, private) or GUA (Global Unicast Addresses, internet-facing) IPv6 — not both. Current limitations include no IPv6 support for Cloud DNS inbound forwarding and some managed services (e.g., Memorystore for Redis). Consider dual-stack when serving IPv6-only clients, facing regulatory requirements, or preparing for gradual IPv6 transition. Stay IPv4-only when all dependencies are IPv4-only and simpler operations are preferred.

## Related notes

- [[notes/K8s/kubernetes-autoscaling|Kubernetes Autoscaling]]
- [[notes/K8s/gke-networking-and-load-balancing|GKE Networking and Load Balancing]]
- [[notes/K8s/kubernetes-networking-internals|Kubernetes Networking Internals]]
- [[notes/K8s/gke-ingress-and-gateway-api|GKE Ingress and Gateway API]]
