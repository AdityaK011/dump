---
title: "CUE-Based Kubernetes Config at Scale: How It Works and Where It Bites"
---

Large engineering orgs with hundreds of microservices tend to converge on the same shape of problem: every team needs Deployments, Services, ConfigMaps, CronJobs, HPAs, and a CD pipeline, and hand-writing that YAML per service doesn't scale. One pattern I've been working with solves it with [CUE](https://cuelang.org): teams write small CUE files describing their service, CI compiles ("hydrates") them into plain YAML committed to the same monorepo, and a GitOps controller syncs that YAML to clusters.

This is a tour of the whole pipeline, from CUE language basics down to the sharp edges I hit while debugging it. The one-sentence version: CUE in, YAML out, GitOps from there.

---

## The two-repo split

The system lives in two repositories.

The **framework repo** is a fork of cue-lang/cue plus the org's own packages. It holds the CUE schemas (what an "App" or "Cron" component is and which Kubernetes resources each expands into), custom CLI subcommands, and Go libraries that CI tools are built from.

The **config monorepo** is the data. It contains per-service CUE under something like `config/services/{service}/{env}/`, the vendored schemas under `cue.mod/pkg/`, and the generated YAML under `.hydrated/`.

The framework reaches the config repo through three separate channels, and knowing which channel a given behavior comes from saves hours of confusion:

```
 FRAMEWORK REPO (fork of cue-lang/cue + org packages)
        │
        │  (1) vendored CUE packages ─────────► cue.mod/pkg/...
        │          schemas: components, k8s types, delivery methods
        │
        │  (2) the forked `cue` CLI ──────────► installed by a tool manager
        │          custom subcommands: dump, deliverables, ...
        │
        └─ (3) the Go module ────────────────► PREBUILT BINARIES committed
                   loader + workflow runner      into the config monorepo
                                                        │
                                                        │ carries go:embed'd CUE
                                                        │ that SHADOWS cue.mod/pkg
 CONFIG MONOREPO (the data)                             │
        ├── config/services/{service}/{env}/*.cue       │
        ├── cue.mod/pkg/  ◄─────────────────────────────┘
        └── .hydrated/{service}/{env}/{cluster}/{deliverable}/
```

| Channel | What travels through it | Where you see it |
|---|---|---|
| Vendored CUE packages | schemas: platform components, k8s types, delivery methods | `cue.mod/pkg/...` -- what `import` resolves against |
| The forked `cue` CLI | custom subcommands (`dump`, `deliverables`, ...) | installed by a tool manager; hydration is literally `cue export -e "#DeliveryTransformer"` run by a shell script |
| The Go module | loader + workflow runner libraries | pinned in `go.mod`, compiled into **prebuilt binaries committed to the config repo** |

That last row matters more than it looks. Hold onto it.

---

## CUE in fifteen minutes

CUE looks like JSON with relaxed quotes, but its model is different from every templating language. There are no functions you call to build objects and there is no override. Everything is **unification**: values merge, and they must agree.

```cue
// These two declarations unify into {name: "feed", replicas: 3}
a: {name: "feed"}
a: {replicas: 3}

// Conflicting concrete values are an error, not "last one wins"
b: replicas: 3
b: replicas: 5   // conflicting values 3 and 5
```

Types and values live on one axis. `string` is a value any concrete string satisfies, `int & >0` is a value any positive integer satisfies. A schema is just a value that hasn't become concrete yet, so validation is free: unify data with schema and check for conflicts.

An identifier starting with `#` is a definition, and definitions are *closed*: you can't add fields they don't declare. This is the guardrail that stops `replcias: 3` from silently deploying. You'll see `&` everywhere in service files -- `platform.#Cron & {...}` means "this struct, constrained by the Cron schema". It reads like inheritance but it's intersection.

Structs and lists can be built with `for` comprehensions, conditional fields use `if` guards, and the idiom `x != _|_` ("x is not bottom") tests whether a field exists:

```cue
namespace: [ for r in v.resources if r.metadata.namespace != _|_ {"\(r.metadata.namespace)"}]

// String interpolation -- this exact line matters later:
label: "\(r.kind)/\(r.metadata.name)"
```

Two things worth internalizing about that interpolation: it evaluates eagerly, and referencing a field that doesn't exist is an error, not an empty string. Older CUE evaluators (the Go API around v0.4) have no reliable `?? default` escape hatch on concrete data.

---

## The authoring model: components and deliverables

A service file uses component definitions -- `#App`, `#Cron`, `#Batch`, `#ConfigMap` -- that each expand into a list of Kubernetes resources. A cron looks roughly like:

```cue
Cron: "feed-cron-backfill": _baseCron & {
    metadata: name: "feed-cron-backfill"
    spec: {
        image: #ImageSpec & {
            name: "feed-cron"
            tag:  _image_tag @imagepolicy("flux-system:...:tag")
        }
        cron: schedule: "30 7 * * *"
        resources: requests: memory: 256Mi
    }
}
```

The `@imagepolicy(...)` attribute is a Flux image-automation marker. It survives into the hydrated YAML as a comment, and Flux rewrites the tag in git when a matching image lands in the registry. That's where the endless automated "update image tag" PRs come from.

The most important concept is the top-level `Delivery` struct. **Each key in it is one deliverable**: one folder of hydrated output, one CD configuration, one independently deployed unit. The author decides resource grouping by concatenating lists:

```cue
Delivery: "feed-cron-backfill": platform.Delivery & {
    config:    CDConfig["feed-cron-backfill"]           // its CD configuration
    resources: list.Concat([
        Cron["feed-cron-backfill"].resources,            // the CronJob
        ConfigMap["feed-cron-backfill"].resources,       // rides along, deploys atomically
    ])
}
```

A CronJob and its ConfigMap share a manifest because the author put both in one Delivery key. Two crons get separate folders because they're separate keys, each with its own image policy and rollout. There is no per-resource logic downstream; grouping is 100% an authoring decision.

A transformer definition (call it `#DeliveryTransformer`) loops `for k, v in Delivery` and shapes each deliverable for output: sorts resources by install order (namespaces before configmaps before workloads), marshals them into one multi-document YAML stream, and attaches the CD configuration with standard labels.

---

## Hydration

Hydration compiles CUE to YAML and commits the result. The committed YAML is what everything downstream consumes -- review, policy checks, CD. Per changed directory, a script runs:

1. `cue export -e "#DeliveryTransformer" --out yaml ./dir/...` -- one YAML doc keyed by deliverable name.
2. For each key, write `manifest.yaml`, `deliverable.yaml`, and `cdconfiguration.yaml` into `.hydrated/{service}/{env}/{cluster}/{deliverable}/`.

```
author edits .cue
      │
      ▼
cue export -e "#DeliveryTransformer" --out yaml ./dir/...
      │   one YAML doc, keyed by deliverable name
      ▼
.hydrated/{service}/{env}/{cluster}/{deliverable}/
      ├── manifest.yaml         multi-doc k8s resources, install-order sorted
      ├── deliverable.yaml
      └── cdconfiguration.yaml
      │
      ▼   committed by CI as a "hydrate manifests" commit on the branch
   git main branch
      │
      ├── CI apply step: upsert the cdconfiguration CRD ──────────► cluster
      │
      └── CD controller (Argo CD) continuously syncs
            manifest.yaml straight from git ─────────────────────► cluster
```

Developers don't run this by hand for PRs: a CI workflow hydrates and pushes a "hydrate manifests" commit onto the branch, and a separate check verifies the committed output matches the CUE so hand-edits can't drift.

**Known gap:** hydration only *overwrites deliverables that still exist*. The script loops over the export's keys and does `rm -rf dst_dir` per key. Nothing diffs the existing `.hydrated/` folders against the current key set, so when you delete a deliverable from CUE, its hydrated folder stays behind untouched. Deleting a deliverable is a two-file-tree operation -- the CUE and the hydrated folder -- and I watched a real team need a follow-up PR two weeks later to remove 346 lines of stale YAML they didn't know was still there.

---

## CI: plan on the PR, apply on merge

The PR pipeline runs, per environment, a **deploy plan** (diff the CD configuration against the cluster) and a **prune plan** (what would be deleted), posting both as PR comments. Deletions get a loud warning table. Policy checks (OPA/Gatekeeper style) run against the hydrated manifests, and routine changes like image bumps can merge with zero human approvals.

The plan jobs compute a merge base against the main branch and compare "the world at base" with "the world at your head". That's worth remembering: **plan is a git-diff-driven simulation**, not a dry-run of your branch in isolation.

On merge, the apply pipeline runs deploy first, then prune, and stops before pruning if any deploy failed. The key subtlety: for GitOps-delivered services, **apply does not kubectl-apply your workload**. The deploy step just upserts a CD configuration CRD; the CD controller (Argo CD under the hood) then continuously syncs the resources from `manifest.yaml` in git. Deployment is a consequence of the file existing on the main branch, not of the CI job pushing anything.

---

## How deletion works (three mechanisms)

Addition and update are the happy path; deletion is where the sharp edges are.

1. **Removing a resource from a deliverable.** Delete it from CUE, rehydrate, `manifest.yaml` shrinks, and Argo CD's sync prunes the resource. CI's prune step exists to *warn* about it.
2. **The prune tool.** A Go binary answers "what existed at the merge base that no longer exists at head?" It walks `.hydrated/` at HEAD, force-checkouts the merge base in the working tree to read the old state (yes, really -- uncommitted changes do not survive), diffs the two sets keyed by `kind/metadata.name`, and runs each affected deliverable's `$prune` workflow. Anticlimax: for GitOps deliverables the `$prune` tasks are print-only. They render "Resources to be pruned: ..." into the PR comment and delete nothing, because Argo CD already handles removal.
3. **Removing a whole deliverable.** Remove the Delivery key, hand-delete the hydrated folder (see the known gap above), and a separate CI job notices the deleted `cdconfiguration.yaml` in the diff and removes the CD configuration from the cluster.

---

## War story: the generateName crash

A Job was authored with `metadata.generateName: "batch-trigger-"` instead of `name`. That's a legitimate Kubernetes pattern for run-per-deploy Jobs created via `kubectl create`, but apply-semantics sync can't create a generateName-only resource, so the Job never materialized. The fix PR renamed `generateName` to `name`. And then CI failed:

```
[FATAL] Destruction failed: Delivery.$prune.plan."print-resources".0:
        invalid interpolation: undefined field: name
```

The chain touches four layers:

```
metadata.generateName: "batch-trigger-"      (and no metadata.name)
        │
  [1]   prune identity is kind/metadata.name  ->  old key is literally "Job/"
        │   renaming generateName -> name makes that key vanish
        │   => the old resource looks DELETED => into the prune set
        ▼
  [2]   $prune print task interpolates "\(r.kind)/\(r.metadata.name)"
        │   undefined field: name  ->  CUE eval error  ->  exit 1
        ▼
  [3]   guarding with `if r.metadata.name != _|_` does NOT help
        │   field-selection error wins on concrete YAML-built data
        │   under the old evaluator
        ▼
  [4]   even a correct CUE fix never ships from the config repo:
        │   the workflow CUE is go:embed'd in the committed prune binary,
        │   registered as overlays that SHADOW cue.mod/pkg on disk
        ▼
  FIX   Go-side: backfill metadata.name <- metadata.generateName on read
```

Layer by layer:

1. Prune identity is `kind/metadata.name`. The old Job had no name, so its key was the literal `Job/`. The rename made the old resource look deleted, so into the prune set it went.
2. The prune workflow's print task interpolates `"\(r.kind)/\(r.metadata.name)"` per pruned resource. On the name-less old resource that's an undefined field, which is a CUE evaluation error, which is exit 1.
3. Guarding the CUE with `if r.metadata.name != _|_` doesn't help: on concrete YAML-built data under the old evaluator, the field-selection error wins anyway. I proved this empirically after being very confident it would work.
4. And even a working CUE fix wouldn't ship from the config repo: the workflow CUE is embedded in the committed prune binary via `go:embed`, and the loader registers the embedded files as overlays that shadow the vendored copies on disk. I patched the on-disk files twice and watched nothing change before reading the loader source.

The fix that actually works is Go-side: backfill `metadata.name` from `generateName` on every resource read from a hydrated manifest, so both the identity key and the print task always have a name to work with.

```go
func withFallbackName(v cue.Value) cue.Value {
    if v.LookupPath(cue.ParsePath("metadata.name")).Exists() {
        return v
    }
    n, err := v.LookupPath(cue.ParsePath("metadata.generateName")).String()
    if err != nil {
        return v
    }
    return v.FillPath(cue.ParsePath("metadata.name"), n)
}
```

Verified end to end against the failing PR in a scratch clone: before, the exact CI failure; after, the plan prints `Resources to be pruned: - Job/batch-trigger-` and exits 0.

---

## Key takeaways

- A **deliverable** is the unit of hydration, deployment, image automation, and pruning. Resource grouping is an authoring decision.
- **Plan is a merge-base diff**, apply is deploy-then-prune, and both mostly manage the CD configuration. Argo CD does the actual resource sync from git.
- **Deletion is three mechanisms**: sync prunes shrunk manifests, a CI job removes dead CD configurations, and you remove dead hydrated folders yourself.
- **CUE unification has no override.** Conflicting concrete values are errors, closed definitions catch typos, and interpolation on a missing field is a hard failure rather than an empty string.
- When a tool misbehaves, ask **which channel its logic came from**: vendored CUE, the installed CLI, or code embedded in a committed binary. They version independently, and the embedded copy silently shadows the vendored one.

The broader lesson generalizes past CUE: any pipeline that generates committed artifacts needs an answer for *removal*, not just generation, and any tool that embeds its config at build time needs to make that loudly visible, because the on-disk copy that looks authoritative is the one thing editing won't fix.

---

## Interview Prep

### Q: Why pick CUE over a templating language like Helm or a patch-based tool like Kustomize for a several-hundred-service config monorepo?

**A:** Because the problem at that scale is *validation*, not string generation. CUE's model is unification rather than substitution: types and values live on the same axis, so `int & >0` is just another value, and a schema is a value that hasn't become concrete yet. Validation is therefore free — unify the service's data with the component schema and look for conflicts.

Two properties matter most in practice:

- **No override, no "last one wins."** `replicas: 3` unified with `replicas: 5` is an error. In a monorepo where several layers (org defaults, environment overlays, service files) contribute to one object, silent precedence is how you ship the wrong number.
- **Closed definitions.** `#App` declares its fields, and a struct constrained by it can't add new ones. That's what turns `replcias: 3` from a no-op deploy into a compile failure.

The cost is that the mental model is unfamiliar. `&` looks like inheritance but is intersection, error messages get long, and eager evaluation means a missing field referenced in an interpolation is a hard failure rather than an empty string — which is exactly the class of bug in the war story above.

### Q: What is a "deliverable" in this pipeline, and who decides which resources deploy together?

**A:** A deliverable is one key in the top-level `Delivery` struct, and it is simultaneously the unit of hydration (one output folder), deployment (one CD configuration), image automation (its own image policy), and pruning. Its resource list is built by the *author*, typically with `list.Concat`:

```cue
Delivery: "feed-cron-backfill": platform.Delivery & {
    config:    CDConfig["feed-cron-backfill"]
    resources: list.Concat([
        Cron["feed-cron-backfill"].resources,
        ConfigMap["feed-cron-backfill"].resources,
    ])
}
```

So "does this ConfigMap deploy atomically with the CronJob?" is answered entirely by whether the author put them in the same key. There is no per-resource routing logic downstream. Two crons in two keys get two folders, two CD configurations, and two independent rollouts; the same two crons in one key roll out together. Grouping is 100% an authoring decision, which is worth knowing before you go looking for a controller that made the choice.

### Q: A change merges. Walk me through what actually deploys the workload.

**A:** Not the CI job, which is the counter-intuitive part.

```
merge to main
   │
   ├── deploy step: upsert the cdconfiguration CRD into the cluster
   │       (this is all the CI job writes; it does NOT kubectl-apply the workload)
   │
   └── prune step: runs only if every deploy succeeded
           for GitOps deliverables, print-only

then, continuously and independently:
   CD controller (Argo CD) reads manifest.yaml from git ──► applies to cluster
```

The workload deploys because `manifest.yaml` exists on the main branch and a CD configuration points the controller at it. Deployment is a consequence of the file's content in git, not of a push from the pipeline. Two practical implications: rolling back means reverting the commit, not re-running the job; and a green apply pipeline does not mean the resources are healthy — it means the CRD was upserted.

On the PR side, both plans are **merge-base diffs**: CI computes the merge base against main and compares "the world at base" with "the world at your head." It's a git-diff-driven simulation, not a dry-run of your branch in isolation, so anything that changes the merge base changes the plan.

### Q: A PR renames a Job's `metadata.generateName` to `metadata.name`. CI fails with `invalid interpolation: undefined field: name`. Debug it.

**A:** The rename is correct — apply-semantics sync can't create a `generateName`-only resource, so the Job was never materializing — and the failure is in the *prune* path, four layers deep:

1. Prune identity is `kind/metadata.name`. The old Job had no `name`, so its identity key was the literal string `Job/`. Adding a `name` makes that key disappear from the head state, so the differ classifies the old resource as **deleted** and adds it to the prune set.
2. The `$prune` workflow's print task interpolates `"\(r.kind)/\(r.metadata.name)"` for each pruned resource. On the name-less *old* resource that's a reference to a field that doesn't exist, which in CUE is an evaluation error, not an empty string. Exit 1.
3. The obvious fix — guard with `if r.metadata.name != _|_` — does not work. On concrete data built from YAML under the older evaluator, the field-selection error surfaces regardless of the guard. Worth testing rather than reasoning about; I was confident it would work and it didn't.
4. Even a correct CUE fix wouldn't ship from the config repo, because the workflow CUE is `go:embed`'d into the committed prune binary and the loader registers those embedded files as overlays that shadow the vendored copies in `cue.mod/pkg`. Editing the on-disk file changes nothing.

The fix has to be Go-side: when reading resources out of a hydrated manifest, backfill `metadata.name` from `metadata.generateName` if `name` is absent, so both the identity key and the print task always have something to interpolate. That also makes the diff stable — the old and new resource now share the key `Job/batch-trigger-` instead of one appearing deleted.

### Q: You remove a service's cron from its CUE file and the PR merges cleanly. What's left behind?

**A:** Its hydrated folder, and therefore its CD configuration, until someone deletes them by hand.

Hydration overwrites deliverables that *still exist*: the script iterates the export's keys and `rm -rf`s each destination directory before rewriting it. Nothing diffs the existing `.hydrated/` tree against the current key set, so a folder whose key no longer exists is simply never visited. Deleting a deliverable is a **two-file-tree operation** — remove the `Delivery` key *and* remove the hydrated folder. Once the folder is gone, a separate CI job spots the deleted `cdconfiguration.yaml` in the diff and removes the CD configuration from the cluster.

If you only do half of it, the resources keep syncing from the stale manifest and nothing warns you. The generic version of this: any pipeline that generates committed artifacts needs an explicit answer for *removal*, because generation-only logic can't distinguish "not regenerated" from "deleted."

### Q: You patch a CUE file under `cue.mod/pkg/`, rerun the tool, and nothing changes. Why?

**A:** Because framework logic reaches the config repo through three channels that version independently, and you patched the wrong one. Vendored CUE under `cue.mod/pkg/` is what `import` resolves against. The forked `cue` CLI supplies custom subcommands and is installed by a tool manager. And the Go module is compiled into **prebuilt binaries committed to the config repo**, with workflow CUE `go:embed`'d inside them; the loader registers those embedded files as overlays that take precedence over the on-disk copies.

So the triage question when a tool misbehaves is always "which channel is this behavior coming from?" If it's the embedded copy, the on-disk file that looks authoritative is the one thing editing won't fix, and you need a framework-repo change plus a binary refresh.

---

## Related Notes

- [[notes/Elasticsearch/from-subscription-to-shard|From Subscription to Shard]] — the same hydrate-then-GitOps pipeline seen from a platform tenant's side: a CUE index entry that hydrates into one-shot Kubernetes Jobs, and a spec-hash Job name whose revert collision is the mirror image of the `generateName` identity problem here
- [[notes/Elasticsearch/elasticsearch-as-a-service-on-kubernetes|Elasticsearch as a Service on Kubernetes]] — what a platform *renders* on top of a config layer like this, including using a Job's existence as the idempotency record
- [[notes/K8s/server-side-apply-replicas-collapse|SSA Deletes Your Replicas]] — what happens after the hydrated YAML reaches the API server: field ownership, apply semantics, and why "the manifest is correct" isn't enough
- [[notes/K8s/kubebuilder-controllers-and-webhooks|Kubebuilder Controllers, Webhooks & Extension APIs]] — the CRD and reconcile-loop machinery behind the CD configuration that this pipeline actually writes
- [[notes/K8s/config-sync-rootsync-cluster-selectors-and-field-ownership|Config Sync at Fleet Scale]] — the GitOps side of the pipeline: one repo across many clusters, cluster selectors, and the empty-cluster-name prune
