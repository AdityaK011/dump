---
title: "Summary: Kubebuilder Controllers, Webhooks & Extension APIs"
---

> **Full notes:** [[notes/K8s/kubebuilder-controllers-and-webhooks|Kubebuilder Controllers, Webhooks & Extension APIs -->]]

## Key Concepts

### Controller Reconcile Triggers & Idempotency
- Reconcile triggered by: CR status update, owned resource changes (`Owns()`), cache/informer lag, explicit `RequeueAfter`
- Treat `AlreadyExists` on create as success (idempotency under cache race)
- `RequeueAfter: time.Minute` schedules ONE delayed requeue per call; becomes periodic only if returned every time
- Periodic reconcile: always return `RequeueAfter: 3m` at end, or use manager cache `SyncPeriod`
- Controller watches group+kind, not specific version -- `For(&v1alpha1.X{})` works even with `v1`

### kind (Kubernetes in Docker)
- kind nodes run containerd inside Docker containers -- local Docker images not visible
- `kind load docker-image <image> --name <cluster>` copies image into node's containerd store
- Verify: `docker exec -it <node> crictl images | grep <image>`
- Use `imagePullPolicy: IfNotPresent` to use loaded images

### Cert-Manager & Webhook TLS
- Webhook TLS cert/key: mounted as K8s Secret into webhook Pod
- CA certificate: placed in `webhooks[].clientConfig.caBundle` in WebhookConfiguration
- Certificate DNS SANs must match Service: `webhook-svc.<ns>.svc.cluster.local`
- Secrets are namespaced -- pods can only mount secrets from their own namespace

### Multi-Version CRDs & Admission Webhooks
- `matchPolicy: Equivalent` (default): API server may convert objects between versions before calling webhook
- `matchPolicy: Exact`: webhook called only for exact apiVersion in request (no conversion)
- Gotcha: webhook handler expecting v1alpha1 fails when receiving v1 with Exact matching
- Fix patterns:
  1. Single-version admission: webhook rules list only one version, use Equivalent matching
  2. Separate handlers per version
  3. One handler detecting request version and decoding the matching Go type

### Conversion Webhooks
- Conversion happens whenever API server needs a different version: for webhooks, for storage, for watches
- `nameReference` in kustomization: tells kustomize to rewrite CRD fields if Service names transformed (no-op if no Service in build)
- Stored version: determined by `spec.versions[].storage: true`; check `status.storedVersions`

### Flow: kubectl apply with webhooks
- kubectl -> API server -> (convert to webhook's version) -> Mutating -> (convert) -> Validating -> (convert to storage version) -> etcd -> controller watch -> reconciler

### Extension API Servers vs CRDs

| Aspect | CRD | Extension API Server |
|---|---|---|
| Storage | Main cluster etcd (mandatory) | Developer choice (SQL, memory, separate etcd) |
| Who writes bytes | kube-apiserver | Your custom binary |
| Max object size | ~1.5 MB (etcd limit) | Unlimited |
| Complexity | Very low | Very high |

- Extension API Server registered via `APIService` resource (aggregation layer)
- Never give extension direct access to main etcd (security + stability risk)
- Metrics Server example: stores in RAM, scrapes fresh data on restart
- Non-etcd storage loses: watch events, resource versions (optimistic locking) -- must implement manually

## Quick Reference

```
Reconcile Trigger Sources:
  1. CR spec/status update
  2. Owned resource change (Owns(&Deployment{}))
  3. Cache lag / informer resync
  4. Explicit RequeueAfter

Webhook TLS Setup:
  cert-manager Certificate -> Secret (tls.crt, tls.key)
  Mount Secret into webhook Deployment at /tls
  Set caBundle in WebhookConfiguration
  DNS SAN: webhook-svc.<ns>.svc.cluster.local

matchPolicy Behavior:
  Equivalent: v1 request -> converted to v1alpha1 -> webhook called
  Exact: v1 request -> webhook called with v1 -> must decode v1

When to use Extension API Server:
  - Data too large for etcd (>1.5 MB)
  - Ephemeral data (metrics, calculated values)
  - Need SQL/relational storage
  - Proxy to external APIs
```
