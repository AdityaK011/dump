---
title: "Config Sync at Fleet Scale: RootSync, Cluster Selectors, and the Label That Wouldn't Die"
---

## Why this note exists

Config Sync is the GitOps engine behind GKE's fleet configuration management. On the surface it's simple: point a cluster at a git repo, and it keeps the cluster looking like the repo. In practice, the interesting parts are all at the seams.

One repo feeds many clusters, so something has to decide which object goes where. That's the job of `Cluster` and `ClusterSelector` objects, and the reconciler's `--cluster-name`.

The reconciler has to reach the git host, often from private nodes with no public IP, through an allowlist that only trusts a fixed egress address. That's where a forward proxy comes in.

And Config Sync applies with Server-Side Apply, which means it shares fields with every other tool that touches the same object. When one of those tools is Terraform, removing a label from git doesn't remove it from the cluster, and with Istio's injection labels that turns into pods quietly getting the wrong sidecar or none at all.

This note walks through each of those, with the failure modes that actually show up in production.

---

## The moving parts

### Components

Config Sync runs in the `config-management-system` namespace. The core pieces:

```
config-management-system
├── reconciler-manager            (Deployment, one per cluster)
│     watches RootSync / RepoSync objects
│     creates one reconciler Deployment per sync object
│
├── root-reconciler               (for RootSync "root-sync")
├── root-reconciler-<name>        (for any other RootSync)
│     containers:
│       git-sync        -> fetches the repo into a shared emptyDir
│       hydration-controller (only if kustomize/helm rendering is needed)
│       reconciler      -> parse, validate, apply, prune, remediate
│       otel-agent      -> metrics export
│
└── ResourceGroup <rootsync-name> (inventory of everything this sync applied)
```

`RepoSync` is the namespace-scoped sibling: it lives in a tenant namespace, gets a reconciler called `ns-reconciler-<namespace>`, and can only manage objects in that namespace. This note is mostly about `RootSync`, which is cluster-scoped in effect and can manage anything.

### A RootSync

```yaml
apiVersion: configsync.gke.io/v1beta1
kind: RootSync
metadata:
  name: root-sync
  namespace: config-management-system
spec:
  sourceFormat: unstructured        # or "hierarchy"
  sourceType: git
  git:
    repo: https://git.example.com/platform/fleet-config.git
    branch: main
    dir: clusters/                  # subdirectory to sync
    auth: token                     # none | ssh | cookiefile | token | gcpserviceaccount | gcenode | githubapp
    secretRef:
      name: git-creds
    period: 15s                     # polling interval, 15s is the default
    proxy: http://egress-proxy.infra.svc:3128
  override:
    reconcileTimeout: 5m
```

### The reconcile loop

Every reconciler runs the same pipeline:

```
  git host
     │  (poll every spec.git.period)
     ▼
┌──────────┐   commit   ┌────────────┐  objects  ┌──────────┐
│ git-sync │──────────▶│   parse    │─────────▶│ validate │
│          │  on disk   │ + render   │           │          │
└──────────┘            │ + select   │           └────┬─────┘
                        └────────────┘                │ declared set
                                                      ▼
                                              ┌───────────────┐
                                              │ apply (SSA)   │
                                              │ prune (diff   │
                                              │  vs inventory)│
                                              └──────┬────────┘
                                                     │
                                  ┌──────────────────┴──────────┐
                                  ▼                             ▼
                          ResourceGroup                 remediator watches
                          (inventory)                   managed objects and
                                                        reverts drift
```

The step that matters most for this note is "select". Between parsing and applying, the reconciler throws away every object that isn't meant for this cluster. What's left is the declared set. Anything that's in the inventory but not in the declared set gets pruned, meaning deleted.

That one rule, "in inventory, not declared, so delete", is the root of the scariest failure mode below.

---

## One repo, many clusters

There are three common ways to lay out a shared repo.

### Directory per cluster

Each cluster's RootSync points at its own `dir`:

```
fleet-config/
├── clusters/
│   ├── prod-tokyo/
│   ├── prod-osaka/
│   └── dev-tokyo/
└── base/          (not synced directly, copied or kustomized in)
```

This is explicit and easy to reason about, and a cluster can't accidentally receive another cluster's config. The cost is duplication: a change meant for every prod cluster is N edits, or you add kustomize overlays and a rendering step.

### Same directory, cluster selectors

Every RootSync points at the same `dir`, and objects carry annotations saying which clusters they belong to. One commit, one place, and each reconciler filters it down to its own slice.

This is where `Cluster` and `ClusterSelector` come in, and it's what most of this note is about.

### Pre-hydrated per cluster

A CI step renders the source (CUE, Helm, kustomize, whatever) into one plain-YAML directory per cluster, commits it, and each RootSync syncs its own rendered directory. The selection logic moves into the build, and the cluster only ever sees fully resolved manifests. See [[notes/K8s/cue-kubernetes-config-at-scale|CUE-Based Kubernetes Config at Scale]] for that model in depth.

### The rollout problem all three share

With polling at 15s, a merged commit reaches every cluster that syncs that branch within about a quarter of a minute. There's no built-in progressive rollout. If you want dev before prod, you need that structure in git: separate branches, separate directories, or per-cluster `revision` pins that you advance deliberately.

---

## Cluster and ClusterSelector

### The objects

A `Cluster` object is a record in the repo that gives a cluster a name and some labels:

```yaml
apiVersion: clusterregistry.k8s.io/v1alpha1
kind: Cluster
metadata:
  name: prod-tokyo
  labels:
    env: prod
    region: asia-northeast1
    tier: edge
```

A `ClusterSelector` is a label selector over those Cluster objects:

```yaml
apiVersion: configmanagement.gke.io/v1
kind: ClusterSelector
metadata:
  name: prod-edge
spec:
  selector:
    matchLabels:
      env: prod
      tier: edge
```

And a regular object opts into a selector by name:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-external-egress
  namespace: edge-gateway
  annotations:
    configmanagement.gke.io/cluster-selector: prod-edge
spec:
  podSelector: {}
  policyTypes: [Egress]
```

In a `hierarchy` repo, Cluster and ClusterSelector objects have to live under `clusterregistry/`. In an `unstructured` repo they can live anywhere. Either way, neither kind is ever applied to the cluster. They're inputs to the parser and nothing more.

Unstructured repos also support a shortcut that skips the indirection entirely:

```yaml
metadata:
  annotations:
    configsync.gke.io/cluster-name-selector: prod-tokyo,prod-osaka
```

### How selection actually works

Each reconciler knows exactly one thing about its own identity: its cluster name. Selection goes like this:

```
reconciler --cluster-name=prod-tokyo
        │
        ▼
find Cluster object with metadata.name == "prod-tokyo"
        │
        ▼
cluster labels = {env: prod, region: asia-northeast1, tier: edge}
        │
        ▼
for each object in repo:
    no selector annotation           -> keep
    cluster-selector: <name>         -> keep if ClusterSelector <name>
                                        matches the cluster labels
    cluster-name-selector: a,b,c     -> keep if "prod-tokyo" is in the list
        │
        ▼
declared set for this cluster
```

Two details are easy to miss.

First, the cluster's labels come from the Cluster object in the repo. They don't come from the real cluster, from GKE resource labels, or from fleet membership labels. If the repo has no Cluster object for this name, the cluster has no labels as far as selection is concerned.

Second, objects without any selector annotation go to every cluster. Selection is opt-in narrowing, not opt-in inclusion.

### Why the indirection

Why not just annotate every object with a list of cluster names? Because that list would be copied onto hundreds of objects, and adding a cluster would mean editing all of them.

The indirection is the same trick Kubernetes uses with Services and Pods. You describe each cluster once, in terms of attributes (`env`, `region`, `tier`), and objects describe what attributes they want. Adding a new prod edge cluster is one new Cluster object with the right labels. Every object that targets `prod-edge` picks it up without being touched.

It also makes review easier. A diff that changes `tier: edge` to `tier: core` on one Cluster object shows clearly that this cluster's entire edge config is about to go away. The same change spread across 80 annotation lists would be unreadable.

The cost is that the indirection hides mistakes. A selector that matches nothing isn't an error, it's just an empty match. A Cluster object with a typo in its name doesn't fail validation, it just means some reconciler finds no labels. These turn into "the config silently didn't land" or, worse, "the config silently got deleted".

---

## The cluster name and how it goes wrong

### Where the name comes from

The reconciler-manager is started with a cluster name (a `--cluster-name` flag, backed by a `CLUSTER_NAME` environment variable) and passes it to every reconciler it creates. How that value gets set depends on how Config Sync was installed:

| Install method | Where the cluster name comes from |
|---|---|
| ConfigManagement operator (legacy) | `spec.clusterName` on the `ConfigManagement` object |
| Fleet-managed (GKE Enterprise / fleet feature) | the fleet membership name |
| Manual manifests | whatever you put on the reconciler-manager Deployment |

To check what a running cluster thinks its name is:

```bash
kubectl -n config-management-system get deploy reconciler-manager \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="reconciler-manager")].args}' | tr ',' '\n' | grep cluster-name

kubectl -n config-management-system get deploy root-reconciler \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="reconciler")].env}' | tr ',' '\n' | grep -A1 CLUSTER_NAME
```

### The failure mode: missing name means prune

Here's the scenario that bites.

A cluster has been syncing happily with `--cluster-name=prod-tokyo`. Its inventory includes a few dozen objects that are only there because of selectors: an edge-only Namespace, its RoleBindings, NetworkPolicies, a few ConfigMaps.

Then something changes the name. Common causes:

- migrating from the operator-based install to fleet management, where the old `spec.clusterName` isn't carried over and the membership name differs
- re-registering the cluster to a fleet under a different membership name
- a hand-edited reconciler-manager Deployment that loses the arg during an upgrade
- renaming the Cluster object in the repo without renaming the cluster

On the next sync, the reconciler runs with an empty or different name:

```
--cluster-name=""
        │
        ▼
no Cluster object matches ""  ->  cluster labels = {}
        │
        ▼
cluster-selector: prod-edge     -> selector needs env=prod,tier=edge
                                   labels are {} -> NOT selected
cluster-name-selector: prod-... -> "" not in list -> NOT selected
        │
        ▼
declared set = only the unselected, everywhere-objects
        │
        ▼
inventory - declared = every selector-targeted object
        │
        ▼
PRUNE  (DELETE each one)
```

From the reconciler's point of view nothing is wrong. The repo parsed, validation passed, and a few dozen objects are no longer declared for this cluster, so it removes them. That's exactly what GitOps is supposed to do.

It gets worse when one of the pruned objects is a Namespace. Deleting a Namespace cascades to everything in it, including workloads that Config Sync never managed. A missing string in a Deployment arg can take out a production namespace.

Config Sync does have a guard against a commit that would delete every managed object at once. That guard doesn't help here, because the everywhere-objects are still declared. This is a partial wipe, and partial wipes look like normal changes.

### The objects that survive: detach

Config Sync honours a lifecycle annotation that tells it to abandon an object rather than delete it when it drops out of the declared set:

```yaml
metadata:
  annotations:
    client.lifecycle.config.k8s.io/deletion: detach
```

When a detached object stops being declared, the reconciler removes its own management metadata and leaves the object in place. In the cluster-name scenario, detached objects survive and everything else is pruned. That's why it's worth putting this annotation on anything whose deletion would be catastrophic or slow to recover from: Namespaces holding stateful workloads, CRDs (deleting a CRD deletes every custom resource of that kind), and the RBAC that lets humans fix things.

The trade-off is that detach makes intentional removal manual. If you really do want the object gone, you delete it from git and then delete it from the cluster yourself.

### Guardrails worth having

Render each cluster's view in CI before merge. The `nomos` CLI can hydrate and vet the repo per cluster:

```bash
# render what each cluster will receive
nomos hydrate --path=. --clusters=prod-tokyo,prod-osaka --output=/tmp/hydrated

# validate with selection applied
nomos vet --path=. --clusters=prod-tokyo
```

Diff the rendered output against the previous commit, and fail the build when a cluster's object count drops sharply. A change that removes 40 objects from one cluster should need a human to say yes.

Make the cluster name a checked invariant, not a hope. A simple check after any Config Sync install or upgrade compares the running `--cluster-name` against the expected value and against the set of Cluster objects in the repo.

Alert on prune volume. Config Sync exports apply and delete operation metrics through its otel-agent, and a burst of deletes from a root reconciler is almost never routine.

---

## Reaching git through a forward proxy

### The problem

GKE clusters with private nodes have no public IPs. Egress goes through Cloud NAT, which gives you a set of NAT IPs for a region and VPC. Meanwhile, the git host often has an IP allowlist: GitHub Enterprise Cloud with IP allow lists, a self-hosted GitLab behind a firewall, or a partner's repo with strict ingress rules.

You could allowlist the NAT IPs, but that has two problems. NAT IPs are shared by every workload on the subnet, so allowlisting them lets any pod reach the git host. And a fleet spread across projects and regions has many NAT IP sets, each of which has to be registered and kept current.

A forward proxy with a fixed egress IP fixes both. Every reconciler, in every cluster, sends its git traffic through one proxy, and the git host allowlists one address.

```
cluster A (asia-northeast1)          cluster B (us-central1)
┌──────────────────────┐             ┌──────────────────────┐
│ root-reconciler      │             │ root-reconciler      │
│  └─ git-sync         │             │  └─ git-sync         │
│     HTTPS_PROXY=...  │             │     HTTPS_PROXY=...  │
└─────────┬────────────┘             └──────────┬───────────┘
          │ CONNECT git.example.com:443         │
          └───────────────┬─────────────────────┘
                          ▼
                ┌───────────────────┐
                │  forward proxy    │  static external IP 203.0.113.10
                │  (e.g. Squid)     │  allowlist: git.example.com only
                └─────────┬─────────┘
                          │ TLS tunnel (end to end)
                          ▼
                ┌───────────────────┐
                │    git host       │  IP allowlist: 203.0.113.10
                └───────────────────┘
```

### How it's wired

`spec.git.proxy` on the RootSync sets the proxy for the git-sync container only. The reconciler's own traffic to the API server is unaffected.

```yaml
spec:
  git:
    repo: https://git.example.com/platform/fleet-config.git
    auth: token
    secretRef:
      name: git-creds
    proxy: http://egress-proxy.infra.example.internal:3128
```

A few things to know:

- The proxy uses HTTP `CONNECT` to tunnel TLS. The proxy sees the destination hostname but not the request contents, and the git host sees the proxy's IP as the source. TLS still terminates at the git host, so certificate validation works as normal.
- The proxy setting only applies to HTTPS-based auth types (`token`, `cookiefile`, `none`, and the like). SSH doesn't go through an HTTP proxy. If you're on `auth: ssh`, either switch to HTTPS or route SSH another way.
- Lock the proxy down to the git host's hostnames. An open forward proxy with a trusted IP is a gift to anyone who gets a pod on your network.
- Run the proxy with more than one replica behind a stable address, because every cluster's sync depends on it.

### What happens when the proxy is down

This failure mode is gentler than you might fear. If git-sync can't fetch, the reconciler doesn't get a new commit, so it keeps the last synced commit as its source of truth. Nothing is pruned, and the remediator keeps correcting drift against the last good state. The RootSync reports a source error:

```bash
kubectl -n config-management-system get rootsync root-sync -o yaml | yq '.status.source.errors'
nomos status --contexts=prod-tokyo
```

The cluster is stuck, not broken. New changes don't land until the proxy is back, which matters if you're trying to push a fix during an incident. Keep a documented break-glass path, such as temporarily pointing `proxy` at a second proxy or applying the fix directly.

---

## The label that wouldn't die

### Server-Side Apply in one paragraph

Config Sync applies objects with Server-Side Apply (GA in Kubernetes 1.22), using its own field manager. Under SSA, every field of every object is tracked in `metadata.managedFields` with the manager that set it. When a manager applies again and leaves out a field it used to own, the API server drops that manager's claim. The field itself is only removed if no other manager still owns it. If another manager co-owns it, the value stays and nothing tells you.

That last sentence is the whole bug. For a deeper look at SSA ownership and another way it misbehaves, see [[notes/K8s/server-side-apply-replicas-collapse|SSA Deletes Your Replicas]].

### Client-side apply vs Server-Side Apply

The two are easy to mix up because both are spelled `kubectl apply`. They answer the same question differently: when you apply config to an object that other people and tools also write to, which fields are yours, and what happens to a field you stop declaring?

#### Client-side apply

This is plain `kubectl apply -f`, the long-standing default. The merge logic runs in kubectl, on your machine.

Each apply stores your full config on the object in the `kubectl.kubernetes.io/last-applied-configuration` annotation. The next apply does a three-way diff between that annotation (what you declared last time), your new file (what you declare now), and the live object (what's actually there). It turns the result into a strategic merge patch and sends it.

The removal rule follows from that diff. A field that's in the annotation but missing from your new file is one you declared before and dropped, so kubectl deletes it. A field that exists only on the live object was set by someone else, never appeared in your annotation, and is left alone.

The weakness is that the annotation only remembers one writer. The server has no idea who else cares about which field. If a controller or someone running `kubectl edit` changes a field you also declare, your next apply overwrites it without a word. And because every object carries a copy of its own config, large objects such as CRDs can blow past the 256KB annotation limit.

#### Server-Side Apply

With SSA you send your config to the API server (`kubectl apply --server-side`, or a PATCH with content type `application/apply-patch+yaml`) along with a field manager name. The server does the merge and records ownership per field, per manager, in `metadata.managedFields`.

Every entry there has an operation. `Apply` means the manager sent a declarative "this is everything I want" config. `Update` means an ordinary imperative write: `kubectl edit`, `kubectl label`, a controller's PUT or PATCH, or many Terraform provider resources.

The rules all follow from ownership:

1. An Apply is your complete intent. Fields you include become yours. Fields you used to own and now leave out are released.
2. A released field is deleted only if no other manager owns it. Otherwise it stays, now owned only by the others. This is exactly the bug in this note.
3. Applying a field that another manager owns, with a different value, returns a conflict (HTTP 409) and writes nothing. `--force-conflicts` takes the field over. GitOps tools usually force, since git is meant to be the source of truth. With an identical value there's no conflict and ownership is shared, which is how two tools end up co-owning a label without anyone noticing.
4. Update operations never conflict. They write and take ownership. That's why an imperative `kubectl label ns payments istio-injection-` removes a co-owned label when re-applying from git can't.

#### Telling them apart

Client-side apply asks one question: was this field in what I applied last time? It only knows about you, so it removes eagerly and overwrites other writers silently.

Server-Side Apply asks: who owns this field? It knows about every writer, so it holds back on removal (a field survives as long as anyone owns it) and refuses to overwrite without a conflict.

That gives a quick way to triage. "I removed it and it's still there" almost always points at SSA co-ownership, and the first command is `kubectl get <obj> --show-managed-fields -o yaml`. "My change keeps getting reverted" points at another writer, often a client-side apply or a controller doing Updates.

#### Where the two meet

Tools mix them. When an object managed with client-side apply moves to SSA, kubectl moves the old ownership under a manager called `kubectl-client-side-apply`. The `last-applied-configuration` annotation can be left behind too. Either can keep a field alive the same way a second tool does.

Controllers vary as well. Config Sync, Flux, and Argo CD with `ServerSideApply=true` use SSA, while older Helm releases and many Terraform resources write with plain Updates. Once more than one of them has touched an object, `managedFields` is the only reliable record of who controls what.

### The setup

A platform team creates namespaces with Terraform, because namespace creation is tied to cloud IAM, quotas, and other infrastructure. Terraform sets a baseline label, including Istio's classic injection label:

```hcl
resource "kubernetes_labels" "payments_ns" {
  api_version = "v1"
  kind        = "Namespace"
  metadata {
    name = "payments"
  }
  labels = {
    "istio-injection" = "enabled"
    "team"            = "payments"
  }
  field_manager = "Terraform"
}
```

Later, namespace config moves into Config Sync, and the same Namespace is declared in git with the same label:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    istio-injection: enabled
    team: payments
```

Now both managers own `metadata.labels.istio-injection`. Both set the same value, so SSA records shared ownership and there's no conflict.

```bash
kubectl get ns payments --show-managed-fields -o yaml
```

```yaml
managedFields:
- manager: Terraform
  operation: Apply
  fieldsV1:
    f:metadata:
      f:labels:
        f:istio-injection: {}
        f:team: {}
- manager: configsync.gke.io
  operation: Apply
  fieldsV1:
    f:metadata:
      f:labels:
        f:istio-injection: {}
        f:team: {}
```

(The exact manager names depend on the Terraform provider resource and the Config Sync version. The shape is what matters.)

### The migration

The mesh team moves to revision-based Istio upgrades. Instead of `istio-injection: enabled`, namespaces should carry `istio.io/rev: stable`, where `stable` is a revision tag pointing at whichever control plane revision is current. The change in git is two lines:

```diff
 metadata:
   name: payments
   labels:
-    istio-injection: enabled
+    istio.io/rev: stable
     team: payments
```

Config Sync applies it. The new label shows up. The old one doesn't go away:

```bash
$ kubectl get ns payments --show-labels
NAME       STATUS   AGE    LABELS
payments   Active   412d   istio-injection=enabled,istio.io/rev=stable,team=payments,...
```

What happened, field by field:

```
before:   istio-injection   owners = {Terraform, configsync}
          istio.io/rev      owners = {}

Config Sync applies without istio-injection, with istio.io/rev
          istio-injection   owners = {Terraform}           <- configsync dropped its claim
                            still owned -> value kept
          istio.io/rev      owners = {configsync}          <- new label set

after:    both labels present
```

Config Sync's remediator won't fix it either. Drift correction only covers the fields Config Sync manages, and Config Sync no longer manages `istio-injection`. From its point of view the namespace matches git exactly.

### Why this matters: injection label precedence

On its own, a stale label is just clutter. With Istio it changes which sidecar your pods get, and whether they get one at all.

Istio's sidecar injector registers mutating webhooks with namespace selectors. Roughly:

```
default injector webhook:
    namespaceSelector: istio-injection In [enabled]

revisioned injector webhook (rev / tag "stable"):
    namespaceSelector: istio.io/rev In [stable]
                       AND istio-injection DoesNotExist
```

The `DoesNotExist` clause is how Istio enforces its documented rule: when both labels are present, `istio-injection` wins. The revisioned webhook deliberately ignores any namespace that still has the old label.

So with both labels on `payments`:

```
pod create in ns "payments"
      │
      ├─ revisioned webhook: istio.io/rev=stable ✓, istio-injection DoesNotExist ✗
      │       -> does NOT fire
      │
      └─ default webhook: istio-injection=enabled ✓
              -> fires, injects the DEFAULT revision's sidecar
```

That leads to two different failures depending on the state of the control plane.

If the old default revision is still installed, pods keep getting the old proxy and keep talking to the old istiod. The canary upgrade looks finished on every dashboard that reads namespace labels, but this namespace never moved. You find out when you uninstall the old revision.

If the default webhook is gone, because the old revision was removed or the default tag points somewhere unexpected, nothing injects at all. New pods start with no sidecar. In a mesh with `PeerAuthentication` set to `STRICT`, every other workload rejects their plaintext connections, and the service falls over on its next rollout. The pods themselves are `Running` and `Ready`, which makes it confusing to diagnose.

One more variant: `istio-injection: disabled` combined with `istio.io/rev` also loses to the old label, and the namespace gets no injection.

### Diagnosing it

```bash
# which labels are actually on the namespace
kubectl get ns payments --show-labels

# who owns the stale label
kubectl get ns payments --show-managed-fields -o json \
  | jq '.metadata.managedFields[] | select(.fieldsV1["f:metadata"]["f:labels"]["f:istio-injection"]) | .manager'

# what revision a running pod's proxy came from
kubectl -n payments get pod <pod> -o jsonpath='{.metadata.annotations.sidecar\.istio\.io/status}' | jq .revision

# fleet-wide: namespaces with both labels
kubectl get ns -l 'istio-injection,istio.io/rev'

# Istio's own checks flag conflicting injection labels
istioctl analyze -n payments
```

### Fixing it

The label survives because Terraform still claims it, so that claim has to go.

1. Remove the label from Terraform. With an SSA-based Terraform resource like `kubernetes_labels`, dropping the key and applying makes Terraform release ownership. Since Config Sync no longer owns it either, the API server removes the label.
2. If Terraform isn't the owner you expect, or the owner is gone (a decommissioned tool, a one-off `kubectl apply`), remove the label directly with `kubectl label ns payments istio-injection-`. An imperative removal deletes the field regardless of who owned it. Make sure no live tool still declares it, or it comes back on that tool's next run.
3. Restart workloads in the namespace so new pods go through the revisioned webhook. Existing pods keep their old sidecar until they're recreated.

The broader fix is ownership hygiene: decide which system owns which fields of shared objects, and don't let two systems declare the same field. Terraform creating the namespace and Config Sync owning its labels is a fine split. Both declaring the labels is the trap. If you're moving a field from one manager to another, do it as two steps: add it in the new owner, then remove it from the old one, and check `managedFields` between steps.

---

## Production checklist

- Every cluster's running `--cluster-name` matches a Cluster object in the repo, and that check runs after every Config Sync install or upgrade.
- CI renders each cluster's declared set and flags large drops in object count.
- Namespaces with state, CRDs, and break-glass RBAC carry `client.lifecycle.config.k8s.io/deletion: detach`.
- Alerts exist for reconciler delete bursts and for RootSync source errors that last more than a few minutes.
- The git egress proxy is replicated, allowlisted to the git host only, and has a documented bypass.
- Shared objects have one declared owner per field, and migrations check `managedFields` before and after.
- No namespace carries both `istio-injection` and `istio.io/rev`.

---

## Interview Prep

### Q: How does a single Config Sync repo serve many clusters with different config?

**A:** Each cluster runs its own reconciler, and every reconciler parses the whole repo, then filters it down to the objects meant for that cluster. The filter uses the reconciler's cluster name. That name is looked up against `Cluster` objects in the repo to get a set of labels, and objects carry either a `configmanagement.gke.io/cluster-selector` annotation naming a `ClusterSelector` (a label selector over those labels) or a `configsync.gke.io/cluster-name-selector` annotation listing cluster names directly. Objects with no annotation go everywhere. The alternatives are a directory per cluster, where each RootSync syncs its own `dir`, or pre-rendering per-cluster output in CI. Selectors keep one copy of shared config, directories make each cluster's view explicit, and pre-rendering moves selection into a build step you can diff.

### Q: Why have Cluster and ClusterSelector objects instead of listing cluster names on each object?

**A:** It's the same reason Services select Pods by label instead of by name. You describe each cluster once, by attributes like `env`, `region`, and `tier`, and objects describe the attributes they want. Adding a new prod edge cluster is one new Cluster object. Every object that targets `prod-edge` picks it up without being edited. Listing names on objects means touching every object on every fleet change. The trade-off is that the indirection hides mistakes. A selector that matches nothing is just an empty match, not an error, and a cluster with no matching Cluster object simply has no labels.

### Q: What happens if a Config Sync reconciler starts with no cluster name?

**A:** It can't find its Cluster object, so its label set is empty. Every object that relies on a ClusterSelector or a cluster-name-selector stops matching, so the declared set shrinks to the objects with no selectors. Config Sync then diffs the declared set against its inventory (the ResourceGroup) and prunes everything that dropped out.

```
inventory:  [everywhere objects] + [selector-targeted objects]
declared:   [everywhere objects]
prune:      [selector-targeted objects]   -> DELETE
```

If one of those is a Namespace, the delete cascades to everything inside it. The mass-deletion guard doesn't fire because the declared set isn't empty. The only objects that survive are those annotated `client.lifecycle.config.k8s.io/deletion: detach`, which Config Sync abandons instead of deleting. Typical triggers are migrating from the operator install to fleet management, re-registering under a new membership name, or renaming the Cluster object in git. The defences are a post-install check on the cluster name, per-cluster rendering in CI with a threshold on object count drops, and detach on anything catastrophic to lose.

### Q: Your git host only accepts connections from allowlisted IPs and your GKE nodes are private. How does Config Sync fetch the repo?

**A:** Route git-sync through a forward proxy with a static egress IP and allowlist that one IP on the git host. `spec.git.proxy` on the RootSync sets the proxy for the git-sync container only. The proxy uses HTTP CONNECT, so TLS stays end to end between git-sync and the git host, and the git host sees the proxy's address. This is better than allowlisting Cloud NAT IPs because NAT IPs are shared by every workload on the subnet, and a multi-region fleet has many of them. The proxy only works with HTTPS auth types, not SSH. It should be replicated and restricted to the git host's hostnames. If it goes down, the reconciler keeps applying its last synced commit and reports a source error, so the cluster is frozen rather than broken.

### Q: You removed a label from a Namespace in git, Config Sync synced, and the label is still there. Why?

**A:** Config Sync uses Server-Side Apply. When a manager stops including a field it owned, the API server drops that manager's claim, but only deletes the field if nobody else owns it. If Terraform (or anything else) also set that label, it's still an owner, so the value stays. Config Sync's remediator doesn't help because it only corrects fields Config Sync manages, and it no longer manages that label. `kubectl get ns <name> --show-managed-fields -o yaml` shows who still owns it. The fix is to remove it from the other owner's config, or remove it imperatively with `kubectl label ns <name> <key>-` once nothing still declares it.

### Q: Why does a leftover istio-injection label matter if istio.io/rev is set correctly?

**A:** Istio documents that `istio-injection` takes precedence over `istio.io/rev`, and the webhooks enforce it. The revisioned injector's namespace selector requires `istio-injection DoesNotExist`, so with both labels present it never fires. The default injector, which matches `istio-injection=enabled`, fires instead and injects the default revision. Pods silently stay on the old control plane, and the canary upgrade never covers that namespace. If the default webhook has been removed, nothing injects, pods start without sidecars, and in a `STRICT` mTLS mesh their peers reject them. Pods look `Running` and `Ready` throughout. `istioctl analyze` and `kubectl get ns -l 'istio-injection,istio.io/rev'` find these.

### Q: How would you move ownership of a field from Terraform to a GitOps tool safely?

**A:** Treat it as two changes. First, add the field to the GitOps source and let it sync. Now both own it with the same value, which is harmless. Check `managedFields` to confirm the GitOps manager shows up. Second, remove the field from Terraform and apply. Terraform drops its claim, and the GitOps tool is the sole owner. Doing it in the opposite order or in one step risks either a window where nobody owns the field (it gets deleted) or a long-term co-ownership where removals stop working. The general rule is one declared owner per field, verified with `managedFields` rather than assumed.

---

## Related Notes

- [[notes/K8s/server-side-apply-replicas-collapse|SSA Deletes Your Replicas]]
- [[notes/K8s/cue-kubernetes-config-at-scale|CUE-Based Kubernetes Config at Scale]]
- [[notes/K8s/istio-and-envoy-internals|Istio & Envoy Internals]]
- [[notes/K8s/istio-traffic-management-and-security|Istio Traffic Management & Security]]
- [[notes/K8s/kubebuilder-controllers-and-webhooks|Kubebuilder Controllers, Webhooks & Extension APIs]]
