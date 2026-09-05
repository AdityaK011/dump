---
title: "Summary: CUE-Based Kubernetes Config at Scale"
---

> **Full notes:** [[notes/K8s/cue-kubernetes-config-at-scale|CUE-Based Kubernetes Config at Scale: How It Works and Where It Bites -->]]

## Key Concepts

### The pattern in one line
- **CUE in, YAML out, GitOps from there.** Teams write small CUE files; CI "hydrates" them into plain YAML committed to the same monorepo; a CD controller (Argo CD) syncs that YAML to clusters.
- Solves the "hundreds of services each needing Deployment + Service + ConfigMap + CronJob + HPA + a pipeline" problem without hand-written YAML per service.

### Two repos, three delivery channels
- **Framework repo** — fork of cue-lang/cue + org packages: schemas, custom CLI subcommands, Go libraries.
- **Config monorepo** — the data: `config/services/{service}/{env}/*.cue`, vendored schemas in `cue.mod/pkg/`, generated YAML in `.hydrated/`.
- Framework logic arrives through **three independently versioned channels**:
  1. **Vendored CUE** in `cue.mod/pkg/` — what `import` resolves against.
  2. **The forked `cue` CLI** — custom subcommands; hydration is literally `cue export -e "#DeliveryTransformer"`.
  3. **The Go module** — compiled into **prebuilt binaries committed to the config repo**, with workflow CUE `go:embed`'d inside.
- ⇒ The embedded copy **shadows** the vendored one via loader overlays. Editing the on-disk file does nothing.

### CUE mental model (not templating)
- **Unification, not override.** Declarations merge and must agree. `replicas: 3` & `replicas: 5` = **error**, not last-one-wins.
- **Types and values on one axis.** `string`, `int & >0` are values. A schema is a non-concrete value ⇒ validation is free.
- **`#Name` = definition = closed.** Can't add undeclared fields — this is what catches `replcias: 3`.
- **`&` is intersection, not inheritance.** `platform.#Cron & {...}` = "this struct, constrained by the Cron schema."
- `x != _|_` ("not bottom") tests field existence; `for` builds structs/lists; `if` guards conditional fields.
- **Interpolation evaluates eagerly** and a missing field is an **error, not an empty string**. Old evaluators (Go API ~v0.4) have no reliable `?? default` on concrete data.

### Deliverable = the unit of everything
- Each key in the top-level `Delivery` struct is **one deliverable**: one hydrated folder, one CD configuration, one independently deployed unit, one image policy, one prune scope.
- Resources are grouped by the author with `list.Concat([...])`. **Grouping is 100% an authoring decision** — no per-resource logic downstream.
- Same key ⇒ deploys atomically. Separate keys ⇒ separate folders and independent rollouts.
- `#DeliveryTransformer` loops `for k, v in Delivery`, sorts by install order (namespaces → configmaps → workloads), marshals to multi-doc YAML, attaches the CD configuration.
- `@imagepolicy(...)` = Flux image-automation marker; survives into hydrated YAML as a comment and Flux rewrites tags in git. Source of the endless automated "update image tag" PRs.

### Hydration
- Per changed dir: `cue export -e "#DeliveryTransformer" --out yaml ./dir/...` ⇒ one YAML doc keyed by deliverable name; then write `manifest.yaml`, `deliverable.yaml`, `cdconfiguration.yaml` per key.
- CI hydrates and pushes a "hydrate manifests" commit onto the branch; a separate check verifies committed output matches the CUE so **hand-edits can't drift**.
- **Known gap:** hydration only overwrites deliverables that **still exist** (`rm -rf dst_dir` per exported key). Nothing diffs `.hydrated/` against the current key set ⇒ **deleted deliverables leave their folder behind**. One real case: a follow-up PR two weeks later to remove 346 lines of stale YAML.

### CI: plan on the PR, apply on merge
- PR: **deploy plan** (diff CD configuration vs cluster) + **prune plan** (what would be deleted), both posted as comments; deletions get a loud warning table. OPA/Gatekeeper-style policy checks run on the hydrated manifests. Image bumps can merge with **zero approvals**.
- **Plan is a merge-base git diff** — "world at base" vs "world at head" — not a dry-run of your branch in isolation.
- Merge: deploy, then prune; prune is skipped if any deploy failed.
- **Apply does not kubectl-apply your workload.** It upserts a CD configuration CRD. Argo CD then syncs `manifest.yaml` from git. **Deployment is a consequence of the file existing on main.**

### Deletion = three separate mechanisms
1. **Resource out of a deliverable** — delete from CUE, rehydrate, `manifest.yaml` shrinks, Argo CD sync prunes it. CI's prune step only *warns*.
2. **The prune tool** — Go binary: walk `.hydrated/` at HEAD, **force-checkout the merge base** in the working tree to read old state (uncommitted changes do not survive), diff keyed by `kind/metadata.name`, run each deliverable's `$prune` workflow. For GitOps deliverables those tasks are **print-only** — Argo CD already handles removal.
3. **Whole deliverable** — remove the `Delivery` key, **hand-delete the hydrated folder**, and a separate CI job spots the deleted `cdconfiguration.yaml` and removes the CD configuration from the cluster.

### War story: the generateName crash
- A Job authored with `metadata.generateName: "batch-trigger-"` and no `name`. Legit for `kubectl create`, but **apply-semantics sync cannot create a generateName-only resource** ⇒ the Job never materialized.
- The fix PR renamed `generateName` → `name`, and CI died: `invalid interpolation: undefined field: name`.
- Four layers: prune identity is `kind/metadata.name` ⇒ old key was literally `Job/` ⇒ rename made it look deleted ⇒ print task interpolated a missing field ⇒ CUE eval error ⇒ exit 1.
- `if r.metadata.name != _|_` **does not save you** on concrete YAML-built data under the old evaluator — the field-selection error wins anyway.
- And a CUE fix can't ship from the config repo at all: the workflow CUE is `go:embed`'d in the committed binary, overlays shadow the on-disk copies.
- **Fix is Go-side:** backfill `metadata.name` from `generateName` on every resource read from a hydrated manifest.

## Quick Reference

```text
CUE unification    : values merge and must AGREE (3 & 5 = error, not override)
#Definition        : closed -> undeclared field = compile error (typo guard)
&                  : intersection (not inheritance)
x != _|_           : "field exists" test (bottom check)
"\(a)/\(b)"        : eager; missing field = ERROR, not empty string

Delivery key       = deliverable = 1 folder + 1 CD config + 1 image policy
                                 + 1 rollout + 1 prune scope
Grouping           = authoring decision (list.Concat), zero downstream logic
Prune identity     = kind/metadata.name
Plan               = merge-base diff (base world vs head world)
Apply              = upsert CD config CRD; Argo CD syncs manifest.yaml from git
```

**The pipeline**
```text
.cue  ──cue export -e "#DeliveryTransformer"──►  .hydrated/{svc}/{env}/{cluster}/{deliverable}/
                                                    manifest.yaml
                                                    deliverable.yaml
                                                    cdconfiguration.yaml
                                                        │ committed to git
              CI apply: upsert cdconfiguration ─────────┤──► cluster
              Argo CD: sync manifest.yaml from git ─────┘──► cluster
```

**Three channels — ask which one a behavior came from**
| Channel | Lives in | Shadowing risk |
|---|---|---|
| Vendored CUE | `cue.mod/pkg/` | **shadowed** by embedded copy |
| Forked `cue` CLI | tool manager / `$PATH` | independent version |
| Go module | **committed prebuilt binaries** | `go:embed` overlays **win** |

**Rules of thumb**
- Deleting a deliverable is a **two-file-tree operation**: the CUE *and* the hydrated folder.
- A green apply pipeline means "CRD upserted," not "resources healthy." Rollback = revert the commit.
- Never author a workload with `generateName` only if delivery uses apply semantics.
- Test CUE existence guards empirically; don't reason about evaluator behavior on concrete data.
- Generation-only pipelines can't tell "not regenerated" from "deleted" — build removal in explicitly.

**Anti-patterns**
- Patching `cue.mod/pkg/` to change behavior that actually lives in a `go:embed`'d binary.
- Assuming plan reflects your branch alone (it's a merge-base diff — a moving base moves the plan).
- Expecting a controller to decide resource grouping; the author already did with `list.Concat`.
- Treating a tool that embeds its config at build time as debuggable from the on-disk copy.
