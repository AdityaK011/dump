---
title: "Kubebuilder Controllers, Webhooks & Extension APIs"
---

## Controller Reconcile Triggers & Log Repetition

- **Q:** _What Service is `nameReference` referring to in the CRD, since our kustomization doesn't render any Service?_

  - **A:** It's referring to the **conversion webhook Service** (the Service fronting the webhook server).
  - **A:** `nameReference` doesn't render a Service; it only tells kustomize to **rewrite that CRD field** _if_ a Service in the same build has its name/namespace transformed.
  - **A:** If our build only includes CRDs and no webhook Service, the rule is effectively a **no-op** (unless the CRD already has those fields to rewrite).

- **Q:** _`kind load docker-image memcached:test --name kubebuilder-test` prints "image not yet present on node ... loading.." — explain._

  - **A:** kind nodes run containerd inside separate Docker containers; they don't automatically see our local Docker images.
  - **A:** The message means the node didn't have that image ID yet, so kind is **copying/importing** it into the node's image store.

- **Q:** _How to see if the image is present on the kind cluster?_

  - **A:** List kind nodes and check each node's containerd images via `docker exec`:
    - `kind get nodes --name kubebuilder-test`
    - `docker exec -it <node> crictl images | grep -E 'memcached|test'`
    - or `docker exec -it <node> ctr -n k8s.io images ls | grep -E 'memcached|test'`

- **Q:** _How to run a Pod with this image?_

  - **A:** Use `imagePullPolicy: IfNotPresent` so the cluster uses the loaded image:
    - `kubectl run memcached-test --image=memcached:test --restart=Never --image-pull-policy=IfNotPresent --port=11211`
  - **A:** Or apply a Pod YAML with:
    - `image: memcached:test`
    - `imagePullPolicy: IfNotPresent`

- **Q:** _Why are reconciler logs getting repeated?_

  - **A:** Reconcile is triggered by multiple sources:
    - Updating the CR **status** triggers another reconcile.
    - Creating/updating the owned **Deployment** triggers another reconcile (`Owns(&appsv1.Deployment{})`).
    - Cache/informer lag can cause a second reconcile to still "not find" the Deployment and try to create again, leading to `AlreadyExists`.
    - Any explicit `RequeueAfter` causes another reconcile later.
  - **A:** Fixes/mitigations:
    - Treat `AlreadyExists` on create as success (idempotency under cache race).
    - Avoid unnecessary `RequeueAfter` unless needed.
    - Only update status when it actually changes (already done later; do similar for the initial "Unknown" set).
    - Optional: predicates to ignore status-only updates on owned resources.

- **Q:** _Does `return ctrl.Result{RequeueAfter: time.Minute}, nil` enqueue once or every minute?_

  - **A:** It schedules **one delayed requeue** for that reconcile call.
  - **A:** It becomes "every minute" only if we return `RequeueAfter: 1m` on every reconcile.

- **Q:** _How to ensure at least 1 reconcile occurs every 3 minutes no matter what?_
  - **A:** Most direct: always return at end:
    - `return ctrl.Result{RequeueAfter: 3 * time.Minute}, nil`
  - **A:** Alternative: set manager cache `SyncPeriod` to 3 minutes (global resync behavior; less direct than `RequeueAfter`).

### Notes

- The controller watches the group and kind. So even if I replace `For(&cachev1alpha1.Memcached{})` with `For(&cachev1.Memcached{})`, it'll work as we expected.

  ```go
  func (r *MemcachedReconciler) SetupWithManager(mgr ctrl.Manager) error {
    fmt.Println("0. Setting up controller with manager")
    return ctrl.NewControllerManagedBy(mgr).
      For(&cachev1alpha1.Memcached{}).
      Owns(&appsv1.Deployment{}).
      Named("memcached").
      Complete(r)
  }
  ```

---

## Cert-Manager, Webhooks & Multi-Version CRDs

### Where do we put the certificate for a mutating/validating webhook?

* We put the **webhook server TLS cert/key** in the **webhook Pod**, typically as a **Kubernetes Secret** mounted into the Deployment.
* We put the **CA certificate** (that signed the webhook server cert) into the **WebhookConfiguration** as `webhooks[].clientConfig.caBundle` so the API server can trust the webhook endpoint.

### With cert-manager installed (self-signed Issuer), how do we create TLS for the webhook and mount it?

* We create a `Certificate` (cert-manager) that writes a TLS Secret (e.g., `webhook-server-tls`) containing `tls.crt` and `tls.key`.
* We mount that Secret into the webhook Deployment (e.g., at `/tls`) and configure the webhook server to serve HTTPS using those files.
* The certificate **must include DNS SANs** matching the Service the API server calls, e.g.:
  * `webhook-svc.<ns>.svc`
  * `webhook-svc.<ns>.svc.cluster.local`

### Secret namespace constraints

* Kubernetes Secrets are **namespaced**, and Pods can only mount Secrets **from their own namespace**.
* The usual fixes are:
  * **Re-issue the Certificate in our webhook namespace** (best for rotation), or
  * **Copy/sync** the Secret into our namespace (one-time copy won't auto-rotate unless we use a sync mechanism).

### Can we fetch configs directly from etcd and see what `apiVersion` is stored?

* Direct etcd reads are possible but not the usual approach:
  * Objects are stored in **storage encoding** (often protobuf/binary) and may be **encrypted at rest**.
  * The "stored apiVersion" is not "whatever we applied"; it's the cluster's **storage version** decision.
* For CRDs, the persisted version is determined by the CRD's `spec.versions[].storage: true`. We can also inspect which versions have been stored historically via `status.storedVersions`.

### Why did our v2 (spoke) defaulting/validating webhook run when we applied a v1 YAML?

* Admission webhooks match **resources**, and by default `matchPolicy` behaves like **Equivalent**.
* With `Equivalent`, the API server may:
  * match a webhook registered for another served version of the same resource, and
  * **convert the object** to the version expected by the webhook before calling it.
* So applying `cache.my.domain/v1` can still trigger a webhook registered for `v2` if the match policy allows equivalence.

### In our webhook YAML, we "strongly mentioned apiVersions: [v2]" — why could v1 still trigger it?

* Because `apiVersions` in webhook rules is subject to `matchPolicy`.
* If `matchPolicy` is **Equivalent** (explicitly or by default), v1 and v2 can be treated as equivalent for webhook matching, and the API server can call the webhook after converting.

### What's the flow of `kubectl apply` with conversion + admission webhooks?

* There isn't a single, fixed "conversion step" between mutating and validating.
* Conversion can happen whenever the API server needs a different version:
  * to call a webhook registered for another version (when Equivalent matching applies),
  * to store the object in the CRD's storage version,
  * to serve responses/watches to clients/controllers.
* High-level flow:
  * `kubectl` -> API server -> (maybe convert to webhook's version) -> **Mutating** -> (maybe convert) -> **Validating** -> (convert to storage version) -> etcd -> controller watch -> reconciler

### After we set `matchPolicy: Exact`, why did our `kubectl apply` start failing with decode errors?

* We changed the behavior so the API server will call the webhook **only for the exact apiVersion in the request** (no equivalence conversion for webhook matching).
* We configured the webhook rules to include both:
  * `apiVersions: [v1alpha1, v1]`
  * but the webhook endpoint path/handler was still effectively a **v1alpha1 decoder** (kubebuilder-generated handler expects `*v1alpha1.Memcached`).
* Result:
  * We applied `cache.my.domain/v1`
  * API server called webhook with a **v1** object (because Exact)
  * Webhook tried to decode into **v1alpha1** Go type
  * Decode failed -> admission denied:
    * `unable to decode ... v1 ... into *v1alpha1.Memcached`

### How do we fix that failure?

Three valid patterns:

1. **Single-version admission (recommended)**
   * Make webhook rules list only the version our webhook code expects (e.g., only `v1alpha1`)
   * Use `matchPolicy: Equivalent` (or omit matchPolicy if default is Equivalent)
   * The API server converts v1 -> v1alpha1 before calling the webhook

2. **True version-specific admission**
   * Keep `matchPolicy: Exact`
   * Have **separate webhook handlers** (or separate paths) for `v1alpha1` and `v1`
   * Each handler decodes its own version's Go type

3. **One handler that supports multiple versions**
   * Keep `matchPolicy: Exact` + include multiple `apiVersions`
   * Update webhook code to detect request version and decode the matching Go type
   * Ensure both versions are added to the runtime scheme

### What was the "main reason" something failed in our case?

* We configured the webhook to accept **v1 requests** (`apiVersions: [v1alpha1, v1]`) while the webhook handler was still decoding only into **v1alpha1** types.
* Setting `matchPolicy: Exact` made the mismatch visible immediately because the API server stopped converting v1 payloads into the v1alpha1 form expected by the webhook.

---

## Extension API Servers vs CRDs

### What exactly is an "External API Server" in the context of Kubernetes?

An External API Server (often called an Aggregated API Server) is a separate HTTP server that you develop and deploy, which runs _alongside_ the main Kubernetes API server. It extends the Kubernetes API by adding new API groups and resources that look and feel like native Kubernetes objects (e.g., Pods, Services) but are processed by your custom code rather than the core Kubernetes logic.

### How does the main Kubernetes API server know about this external server?

It uses the **Aggregation Layer**. You register your external server using an `APIService` resource.

- **The Mechanism:** When you create an `APIService` object, you tell the main API server: _"If anyone asks for `/apis/my-extension.group/v1`, please forward (proxy) that request to this specific Service running in the cluster."_
- **The Result:** The main API server acts as a gateway. It handles authentication (usually) and then simply tunnels the HTTP request to your external server.

### What is the default storage for the main Kubernetes API Server?

The main Kubernetes API server uses **etcd**.

- **Etcd:** A strongly consistent, distributed key-value store.
- **Why:** Kubernetes relies on etcd for all cluster state data because it guarantees that once a write is confirmed, it is persisted across the cluster. It is optimized for watching changes (key to the controller pattern).

### Does an External API Server store its objects in the main cluster's etcd?

**No, it is not required to.** This is the critical distinction. Because the External API Server is just a piece of software you write, you have full control over the "backend." When the main API server proxies a request (e.g., `POST /apis/my-group/v1/my-resource`) to your server, your code receives the JSON body. What you do with that data is entirely up to you.

### Can an External API Server use the main cluster's etcd if it wants to?

Technically, yes, but it is **strongly discouraged** and often architecturally difficult.

- **Security:** Giving an external pod direct access to the core etcd (where Secrets and core cluster state live) is a massive security risk.
- **Stability:** If your extension spams etcd with bad queries, you could bring down the entire cluster.
- **Standard Pattern:** If you want etcd-like storage, you usually deploy a **separate, dedicated etcd instance** just for your API server.

### If not the main etcd, where can an External API Server store data?

It allows for "Bring Your Own Storage" (BYOS). Common options include:

1. **A Separate Etcd Cluster:** If you want the same behavior as standard K8s resources (watches, consistency) but want isolation.
2. **Relational Databases (SQL):** If your objects have complex relationships, require joins, or need strict referential integrity (e.g., PostgreSQL, MySQL). Etcd is a key-value store and is poor at complex queries; an External API server allows you to bypass this limitation.
3. **In-Memory (RAM):** For data that is calculated on the fly or doesn't need to persist if the pod restarts.
   - _Example:_ The **Metrics Server**. It scrapes node CPU/Memory usage and stores it in RAM. If the metrics server restarts, it just scrapes them again. It doesn't need a database.
4. **No Storage (Proxy/Adapter):** The server might simply translate the K8s API request into a call to a third-party API (like AWS, Google Cloud, or a legacy corporate API) and return the result.

### CRDs vs External API Servers

| Feature                   | **CRD**                                  | **External API Server**                       |
| ------------------------- | ---------------------------------------- | --------------------------------------------- |
| **Primary Storage**       | Main Cluster Etcd (Mandatory)            | Developer Choice (SQL, Memory, Separate Etcd) |
| **Who writes the bytes?** | Kube-APIServer                           | Your Custom Binary                            |
| **Max Object Size**       | ~1.5MB (Etcd limit)                      | Unlimited (Depends on your backend)           |
| **Complexity**            | Very Low                                 | Very High                                     |
| **Ideal Use Case**        | config, operators, standard k8s patterns | Metrics, heavy data, proxying legacy systems  |

### When to choose an External API Server over a CRD

1. **Storage constraints:** You need to store data in a legacy SQL database, or the data is too large for etcd (etcd has a limit of ~1.5MB per object).
2. **Ephemeral Data:** You have data (like metrics) that changes constantly and shouldn't be written to disk (etcd) to avoid burning out the disk I/O.
3. **Complex Validation/Behavior:** You need very specific API behavior (like special verbs or non-standard patching strategies) that declarative CRDs cannot support.

### Using `k8s.io/apiserver` library and the RESTStorage interface

Most developers build External API Servers using the official Kubernetes library (`k8s.io/apiserver`). This library provides a framework that "looks" like the standard K8s API server. Out of the box, this library includes an **etcd adapter**. If you use the default setup, it will ask for an etcd connection string. **However**, you can swap out the `RESTStorage` interface in that library to point to anything else (memory, SQL, etc.).

### Downsides of using non-etcd storage

If you switch to SQL or In-Memory, you lose some "magic" features that Kubernetes provides for free when using etcd:

- **Watch Events:** `kubectl get pods -w` works because etcd supports "watching" keys. If you use PostgreSQL, you have to implement a mechanism to notify the API server when rows change so it can push updates to the user. This is hard to implement manually.
- **Resource Versions:** Kubernetes relies on optimistic locking (resource versions) to prevent write conflicts. You must implement this logic yourself if you use a custom backend.

## See also

- [[notes/K8s/gke-internals-and-networking|GKE Internals & Networking]]
- [[notes/K8s/interactive-containers-piping-ttys|Interactive Containers, Piping & TTYs]]
- [[notes/AuthNZ/oauth-oidc-and-workload-identity|OAuth, OIDC & Workload Identity Federation]] — for GitHub Actions OIDC used with Kubebuilder CI/CD
