---
title: "Summary: Config Sync at Fleet Scale: RootSync, Cluster Selectors, and the Label That Wouldn't Die"
---

> **Full notes:** [[notes/K8s/config-sync-rootsync-cluster-selectors-and-field-ownership|Config Sync at Fleet Scale -->]]

## Key Concepts

### RootSync and the reconciler
- `RootSync` (`configsync.gke.io/v1beta1`) lives in `config-management-system`. The reconciler-manager creates one reconciler Deployment per sync object (`root-reconciler`, `root-reconciler-<name>`).
- Containers: `git-sync` (fetch), `hydration-controller` (kustomize/helm only), `reconciler` (parse, select, apply, prune, remediate), `otel-agent`.
- `RepoSync` is the namespace-scoped sibling (`ns-reconciler-<ns>`).
- Polls every `spec.git.period`, default `15s`. A merged commit reaches every cluster on that branch in about 15s, with no built-in progressive rollout.
- Inventory is a `ResourceGroup`. Prune rule: in inventory but not declared means delete.

### Shared repo layouts
- Directory per cluster: explicit, but duplicated.
- Same directory plus selectors: one copy, each reconciler filters.
- Pre-hydrated per cluster in CI: selection moves into a build you can diff.

### Cluster and ClusterSelector
- `Cluster` (`clusterregistry.k8s.io/v1alpha1`) gives a cluster name plus labels. `ClusterSelector` (`configmanagement.gke.io/v1`) is a label selector over them. Neither is applied to the cluster; both are parser inputs only.
- Object annotation `configmanagement.gke.io/cluster-selector: <selector>`. Unstructured repos also allow `configsync.gke.io/cluster-name-selector: a,b`.
- Hierarchy repos keep them under `clusterregistry/`.
- Cluster labels come from the repo's Cluster object, not from GKE or fleet labels.
- No annotation means the object goes to every cluster.
- The indirection works like Service-to-Pod labels: describe clusters once, objects ask for attributes, and adding a cluster is one new object. The cost is that empty matches are silent.

### Cluster name failure mode
- The reconciler-manager's `--cluster-name` (`CLUSTER_NAME`) is passed to every reconciler. It comes from ConfigManagement `spec.clusterName` (operator install), the fleet membership name (fleet install), or the Deployment args (manual install).
- Empty or wrong name: no Cluster match, so labels are `{}`, so every selector-targeted object leaves the declared set, so it gets pruned.
- A pruned Namespace cascades to everything inside it, managed or not.
- The mass-deletion guard doesn't fire because the declared set isn't empty.
- Survivors carry `client.lifecycle.config.k8s.io/deletion: detach`, which abandons instead of deleting. Use it on stateful Namespaces, CRDs, and break-glass RBAC.
- Triggers: operator-to-fleet migration, fleet re-registration, a lost Deployment arg, a renamed Cluster object.
- Guardrails: `nomos hydrate --clusters=...` and `nomos vet` in CI, fail on object-count drops, verify the name after install, alert on delete bursts.

### Forward proxy for git-sync
- Private nodes egress via Cloud NAT. NAT IPs are shared by the whole subnet and differ per region, which makes them a poor allowlist.
- A forward proxy with a static IP lets the git host allowlist one address for the whole fleet.
- `spec.git.proxy` affects the git-sync container only. HTTP `CONNECT` keeps TLS end to end.
- HTTPS auth types only (`token`, `cookiefile`, `none`, and similar). SSH isn't proxied.
- Restrict the proxy to git hostnames, and replicate it.
- Proxy down: no new commit, last synced state is kept, no prune, remediation continues. RootSync shows `status.source.errors`. Frozen, not broken.

### Client-side apply vs SSA
- Client-side apply: kubectl diffs the `last-applied-configuration` annotation, the new file, and the live object. A field that was in the annotation but isn't in the new file gets deleted. It only tracks one writer, so other writers' changes get overwritten silently.
- SSA: the API server tracks owners per field in `managedFields` (`Apply` = declarative, `Update` = imperative). A field you drop is deleted only if nobody else owns it.
- SSA conflict: a different value on a field someone else owns returns 409 unless `--force-conflicts` is set. The same value means shared ownership. Updates never conflict.
- Triage: "removed but still there" points to SSA co-ownership. "my change keeps reverting" points to another writer.
- CSA→SSA migration leaves a `kubectl-client-side-apply` manager behind, which can keep fields alive.

### SSA co-ownership gotcha
- Config Sync applies with SSA (GA in Kubernetes 1.22).
- When a manager omits a field it owned, its claim is dropped. The field is deleted only if no other manager owns it.
- Terraform and Config Sync both declaring `istio-injection`, then removing it from git: Terraform still owns it, so the label stays.
- The remediator ignores it because Config Sync no longer manages the field.
- Diagnose with `kubectl get ns X --show-managed-fields -o yaml`.

### istio-injection vs istio.io/rev
- Documented rule: `istio-injection` takes precedence over `istio.io/rev`.
- The revisioned webhook selector requires `istio-injection DoesNotExist`, so it skips the namespace. The default webhook (`istio-injection=enabled`) injects the default revision.
- Old revision still installed: pods stay on the old proxy and the canary never covers the namespace.
- Default webhook gone: no sidecar, so `STRICT` mTLS peers reject the pods, which still look `Running` and `Ready`.
- `istio-injection: disabled` plus `istio.io/rev` also means no injection.
- Fix: remove the label from Terraform (or `kubectl label ns X istio-injection-` once nothing declares it), then restart workloads.
- Ownership moves take two steps: add to the new owner, verify `managedFields`, then remove from the old owner.

## Quick Reference

```
SELECTION
--cluster-name=N -> Cluster{name:N}.labels -> ClusterSelector match -> declared set
                                                                       │
                              inventory - declared  ─────────────▶  PRUNE
                              (detach-annotated objects are abandoned instead)

EGRESS
git-sync --CONNECT--> forward proxy (static IP) --TLS--> git host (allowlist 1 IP)

SSA CO-OWNERSHIP
istio-injection owners {Terraform, configsync}
   git removes it ->  owners {Terraform}  -> value KEPT
```

| Labels on namespace | Webhook that fires | Result |
|---|---|---|
| `istio-injection=enabled` | default | default revision |
| `istio.io/rev=stable` | revisioned | `stable` revision |
| both | default only | old or default revision, or none if default webhook is gone |
| `istio-injection=disabled` + rev | none | no sidecar |

| Symptom | Likely cause |
|---|---|
| Selector-targeted objects deleted on one cluster | cluster name empty or changed |
| Config never lands on a new cluster | missing Cluster object or labels |
| RootSync source error, cluster unchanged | git or proxy unreachable |
| Label removed in git still present | co-owned by another field manager |
| Namespace pods on old Istio revision | stale `istio-injection` label |
