---
title: "Ingress vs Gateway API"
---

## How the GKE Ingress controller creates GCP resources

The built-in `ingress-gce` controller runs on GKE master nodes — not in the user's project — and watches for Ingress resources. For each Ingress, it creates the full GCP resource chain: Global Forwarding Rule, Target HTTP(S) Proxy, URL Map, Backend Service(s) (one per unique service:port combination), Health Checks, NEGs or instance groups, and Compute Engine firewall rules. **Initial provisioning takes several minutes.** At scale, reconciliation can be slow: with 20 Ingress objects each containing 20 NEG backends, a single change can take over **30 minutes** to propagate. With 100+ Ingress objects, batch changes can take 6+ hours.

Service-level annotations drive critical behavior: `cloud.google.com/neg: '{"ingress": true}'` enables container-native load balancing, `cloud.google.com/backend-config` references a BackendConfig CRD, and `cloud.google.com/app-protocols` specifies the backend protocol (HTTP, HTTPS, HTTP/2).

## BackendConfig and FrontendConfig: the configuration CRDs you cannot ignore

**BackendConfig** (attached to Service resources) configures per-backend-service settings on the GCP side. The critical fields include `timeoutSec` (backend response timeout, default 30s), `connectionDraining.drainingTimeoutSec` (time to drain connections during endpoint removal — set to 1.5–2x your longest request time), `healthCheck` (override default health check parameters), `securityPolicy.name` (reference a Cloud Armor policy), `sessionAffinity` (CLIENT_IP, GENERATED_COOKIE, HEADER_FIELD), and `logging` (HTTP access log sampling). **Always use API version `cloud.google.com/v1`** — v1beta1 has a known bug that removes Cloud Armor policies.

**FrontendConfig** (attached to Ingress resources) configures frontend-level settings: `sslPolicy` (references a GCP SSL policy controlling TLS versions and ciphers) and `redirectToHttps` (enables automatic HTTP->HTTPS redirects with configurable response codes). FrontendConfig only works with external Ingress, not internal.

## Gateway API vs Ingress: role separation changes everything

The Ingress model combines infrastructure definition, routing rules, and TLS config into a single resource, controlled by one persona, with advanced features bolted on through controller-specific annotations. The Gateway API model separates these concerns into three resources: **GatewayClass** (owned by the infrastructure provider, defines LB capabilities), **Gateway** (owned by the cluster operator, defines listeners and TLS), and **HTTPRoute** (owned by application developers, defines routing rules). Multiple HTTPRoutes from different namespaces can attach to a single shared Gateway, enabling genuine multi-tenant L7 load balancing.

GKE provides pre-installed GatewayClasses: `gke-l7-global-external-managed` (global external ALB), `gke-l7-regional-external-managed` (regional external ALB), `gke-l7-rilb` (regional internal ALB), and multi-cluster variants. The GKE Gateway Controller is Google-hosted (not on control plane nodes), making it more scalable than the Ingress controller. It creates the same GCP resources as Ingress but merges all HTTPRoutes for a Gateway into a single URL map. Notably, **Gateway API supports Certificate Manager** (for scalable certificate management with thousands of entries per map), while GKE Ingress does not. Google now labels Gateway API as "recommended" over Ingress in GKE documentation.

## TLS certificate management at scale

Three certificate approaches exist on GKE.

### GCP Managed Certificates (ManagedCertificate CRD)

The simplest option: Google provisions Domain Validation certificates automatically, renews them approximately one month before expiry, and initial provisioning takes **30–60 minutes**. Limitations include no wildcard domain support, no modification after creation, and GKE Ingress only (not Gateway API).

### cert-manager

Provides more flexibility through the ACME protocol. When a Certificate resource is created, cert-manager creates a CertificateRequest -> Order -> Challenge. For HTTP01 challenges, it temporarily creates a Pod, Service, and Ingress to serve the ACME validation token. For DNS01 challenges (required for wildcards), it creates TXT records via the DNS provider's API. On GKE with GCLB, use the `acme.cert-manager.io/http01-edit-in-place: "true"` annotation to prevent the GKE Ingress controller from assigning a second IP during the challenge.

### Certificate Manager (GCP native)

The most scalable option. It supports certificate maps with thousands of entries, each mapping a hostname to certificates. A single map can be shared across multiple load balancers. DNS authorization (via CNAME records) supports wildcards and allows provisioning before the LB exists. For SNI-based selection, Certificate Manager uses exact hostname match -> wildcard match -> primary entry fallback, with ECDSA certificates prioritized over RSA.

## Related notes

- [[notes/K8s/gke-request-path-and-load-balancing|GKE Request Path and Load Balancing]]
- [[notes/K8s/kubernetes-services-dns-and-network-policies|Kubernetes Services, DNS, and Network Policies]]
