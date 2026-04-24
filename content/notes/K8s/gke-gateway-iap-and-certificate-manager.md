---
title: "GKE Gateway API with IAP and Certificate Manager: A Debugging Deep Dive"
date: 2026-04-23
tags:
  - gke
  - gateway-api
  - iap
  - oauth
  - tls
  - certificate-manager
  - debugging
draft: false
---

**You deploy a Gateway, attach an IAP policy, wire up a Certificate Manager cert map, and hit the endpoint — only to receive a cryptic "Error 52" from Identity-Aware Proxy.** The load balancer is provisioned, the cert serves fine, OAuth is configured, and yet the request never reaches your backend. Welcome to one of the more frustrating corners of GKE networking, where three independent Google Cloud products — Gateway API, IAP, and Certificate Manager — have to agree on a shared model of "what is a route" for your traffic to flow.

This post walks the full request path for a GKE Gateway protected by IAP, explains where each product hooks in, and unpacks the subtle matcher incompatibility that causes Error 52.

## The request flow, layer by layer

When a browser hits a hostname fronted by a GKE Gateway with IAP enabled, the request traverses roughly this path:

1. **DNS and anycast** — The client resolves to a Google global anycast VIP, routed via Maglev to the nearest Google Front End (GFE). See [[gke-request-path-and-load-balancing]] for the edge path.
2. **TLS termination at the GFE** — The GFE selects a cert from the **Certificate Manager cert map** attached to the forwarding rule (via the `networking.gke.io/certmap` annotation on the Gateway). The SNI hostname selects the matching entry in the cert map.
3. **URL map evaluation** — The Gateway controller compiles your `HTTPRoute` resources into a Compute URL map. Hostname and path matchers select a backend service.
4. **IAP enforcement at the backend service** — If IAP is enabled on the backend service (via `GCPBackendPolicy` in Gateway API, or `BackendConfig` under the classic Ingress), the GFE pauses the request and runs the OAuth dance.
5. **Backend forwarding** — Only after IAP validates the signed `X-Goog-IAP-JWT-Assertion` does the GFE forward the request to the container-native NEG backing your Service.

The critical point: **IAP runs at the backend service, not at the Gateway.** You can have a single Gateway hosting ten routes where five are public and five are IAP-protected, and the enforcement decision is made per-backend-service after URL map routing.

## Gateway API vs classic Ingress: same LB, different control plane

The Gateway produced by GKE's Gateway controller ultimately reconciles the same underlying Compute Engine primitives as classic Ingress — forwarding rule, target proxy, URL map, backend services, NEGs. What changes is the Kubernetes API surface:

| Concern              | Classic Ingress                    | Gateway API                                     |
|----------------------|------------------------------------|-------------------------------------------------|
| Routing              | `Ingress`                          | `Gateway` + `HTTPRoute`                         |
| Backend tuning       | `BackendConfig`                    | `GCPBackendPolicy`                              |
| Frontend (TLS, etc.) | `FrontendConfig`                   | `GCPGatewayPolicy`                              |
| Certs                | `ManagedCertificate` CRD           | Certificate Manager cert map (via annotation)   |
| IAP                  | `BackendConfig.iap`                | `GCPBackendPolicy.default.iap`                  |

See [[ingress-vs-gateway-api]] for the broader comparison. The trap when migrating: these are not drop-in replacements, and certs in particular have fundamentally different lifecycle models.

## Certificate Manager cert map vs ManagedCertificate CRD

This is where much of the confusion lives.

**`ManagedCertificate`** is a GKE CRD. You create it in your cluster, the GKE controller provisions a `compute.sslCertificates` resource, attaches it to the target HTTPS proxy, and handles ACME-style domain validation via the load balancer itself. Lifecycle is Kubernetes-native.

**Certificate Manager** is a standalone GCP product with its own resource model:

- A **Certificate** resource holds the cert material (managed by Google, imported, or self-managed).
- A **CertificateMap** groups many certificates behind a single attachment point.
- **CertificateMapEntry** resources map SNI hostnames (including wildcards and `PRIMARY`) to certificates.
- The map is attached to an HTTPS target proxy via a single reference, replacing the old list of `sslCertificates`.

On a Gateway, you attach a cert map by annotating the Gateway with `networking.gke.io/certmap: <cert-map-name>`. The Gateway controller sets the map on the target proxy; individual `ManagedCertificate` or TLS listener configs on the Gateway are ignored for that listener.

Why would you use Certificate Manager over `ManagedCertificate`?

- **Scale** — Cert maps comfortably serve thousands of SNI hostnames per proxy, far beyond the `sslCertificates` list limit.
- **Wildcards with DNS-01** — Google-managed wildcards (`*.example.com`) require DNS-01 validation, which `ManagedCertificate` does not support but Certificate Manager does via DNS authorization resources.
- **Shared cert inventory** — A single cert can be bound to many maps; rotation happens once.
- **Decoupled lifecycle** — Certs outlive clusters, making blue/green cluster swaps safer.

## OAuth flow: Google-managed vs custom brand

IAP sits in front of your backend and requires users to authenticate via an OAuth 2.0 client. There are two flavors of configuration:

**Google-managed OAuth** — You toggle IAP in the Cloud Console, Google mints an internal OAuth client tied to your project, and consent is automatic. No client ID or secret to manage. The catch: it only works for **Internal** OAuth brands, meaning only users inside your Google Workspace organization can authenticate. It's the fastest path for internal tools.

**Custom OAuth client** — You create the OAuth client yourself (client ID + secret), configure an `iap.OAuthSettings` with those credentials on the backend policy, and register the IAP redirect URI (`https://iap.googleapis.com/v1/oauth/clientIds/<CLIENT_ID>:handleRedirect`) in the client. This is required if you want:

- An **External** OAuth brand (any Google account, or federated identities).
- Multiple audience scopes or custom consent screens.
- Different clients per environment sharing one brand.

**OAuth brand requirements** are the part most people miss. The brand's support email must be a group that the project's owner belongs to. Flipping a brand from Internal to External requires app verification if you use sensitive scopes, and once External is locked in, you cannot easily revert. For staging environments, keep Internal; for public-facing IAP, plan the verification work upfront.

At runtime, a successful IAP flow results in the GFE injecting an `X-Goog-IAP-JWT-Assertion` header into the upstream request, signed by Google. Your backend should verify this JWT's signature, issuer (`https://cloud.google.com/iap`), and audience (the numeric backend service ID, or the Gateway signed-header audience). See [[oauth-oidc-and-workload-identity]] for the JWT verification mechanics.

## Debugging IAP Error 52: the PRIMARY matcher incompatibility

The specific failure that sent me down this rabbit hole:

> **Error 52**: `You don't have access to this application. The administrator has not granted you access.`

Except… the user did have `roles/iap.httpsResourceAccessor`. And the error only appeared on *one* hostname served by the Gateway; other hostnames on the same Gateway worked fine.

The root cause turned out to be a **URL map matcher incompatibility between the Gateway controller and the cert map's PRIMARY entry.**

Here's what was happening:

1. The cert map had a `CertificateMapEntry` with matcher type `PRIMARY` — the catch-all for SNI values that don't match any explicit hostname entry. Google uses this to serve a default cert for unmatched SNI.
2. The Gateway's `HTTPRoute` declared an explicit hostname, say `app.example.com`, which *did* have its own cert map entry.
3. But a second hostname, `legacy.example.com`, was only declared as a `HTTPRoute` hostname — it had **no explicit cert map entry**, so TLS fell through to the PRIMARY entry.
4. When the URL map evaluated the request, the Gateway controller had emitted a host rule for `legacy.example.com` pointing at a backend service with IAP enabled. The backend service's IAP config had been created before Gateway API support, and its `GCPBackendPolicy` targeted the backend by name.
5. IAP looked up the OAuth client configured for that backend, but the client's authorized redirect URIs did **not** include the `legacy.example.com` origin — they had been set for `app.example.com` only.

The IAP token exchange therefore failed the origin check, and IAP returned Error 52 as a generic "access denied" rather than the more accurate "redirect URI mismatch" you would see directly from the OAuth endpoint.

The fix had three parts:

1. Add an explicit `CertificateMapEntry` for `legacy.example.com` (do not rely on PRIMARY for hostnames that have IAP policies — PRIMARY is an OAuth-opaque catch-all).
2. Register the `legacy.example.com` origin as an authorized JavaScript origin on the custom OAuth client.
3. Confirm the IAP redirect handler URI was registered on the client.

After those three, the flow worked end-to-end.

## Key takeaways

- **IAP enforces at the backend service, not the Gateway.** A single Gateway can mix public and IAP-protected routes.
- **Certificate Manager cert maps replace the `sslCertificates` list on the target proxy.** Attach via `networking.gke.io/certmap` annotation on the Gateway; do not mix with `ManagedCertificate`.
- **PRIMARY cert map entries are a TLS fallback, not a routing construct.** If a hostname has an IAP policy, give it an explicit cert map entry.
- **Google-managed OAuth works only for Internal brands.** Public-facing IAP needs a custom client and brand verification.
- **Error 52 is a generic refusal.** Always check: IAM grant, OAuth client redirect URIs, OAuth client authorized origins, and brand type — in that order.
- **Every host served by IAP must be a registered origin on the OAuth client**, even if multiple hosts share a backend service.

The Gateway API makes the Kubernetes side cleaner, but it does not abstract away the GCP identity and cert plumbing underneath. When something breaks, the debugging surface is the full seven-layer path: DNS, TLS, URL map, IAP, OAuth, backend service, pod. Knowing which layer owns which error message saves hours.
