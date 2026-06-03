---
title: "Summary: Server-Side Apply Deletes Your Replicas"
---

> **Full notes:** [[notes/K8s/server-side-apply-replicas-collapse|When Server-Side Apply Deletes Your Replicas: The before-first-apply Trap -->]]

## Key Concepts

### The symptom
- HPA-managed Deployment at ~20 pods. A routine deploy briefly drops available pods to **zero** (~1–2 min), loses requests, then self-recovers to ~20.
- All the obvious culprits are dead ends: OOM (a *symptom*), PDB (doesn't apply), `maxUnavailable: 0` rollout (set correctly, irrelevant).
- Real cause: `spec.replicas` was **deleted** by an apply and defaulted to `1`. Everything else is downstream.

### Two ownership models
- **Client-side apply** — 3-way merge on the client vs `last-applied-configuration`. No per-field ownership. Omitted field = left alone. *Always safe for HPA-managed `replicas`.*
- **Server-Side Apply (SSA)** — merge in the API server; every field tagged in `metadata.managedFields` with a **manager** + **operation**.
- **SSA's defining rule:** a field a manager *owns but omits* on the next apply is **deleted**.

### Apply vs Update operations
- `Apply` = declarative SSA ownership → subject to "owned-but-omitted ⇒ delete."
- `Update` = imperative writes (`PUT`, non-apply `PATCH`, *and* legacy client-side apply). Owns only what it wrote; **omission never deletes**.
- Apply and Update owners normally **coexist** (apply over an Update owner = *conflict*, not eviction). One exception: `before-first-apply`.

### `before-first-apply` — the migration shim
- On the **first SSA** to an object previously managed **client-side**, the API server synthesizes a manager `before-first-apply` (`op=Update`) owning all pre-existing fields (from `last-applied-configuration`).
- It's **transitional** — designed to be **absorbed by the applying manager**, not to persist. That's why an `Update` owner gets removed by an `Apply` here.

### The 5-operation trap (Deployment at `replicas: 20`, HPA-managed, manifest omits `replicas`)
1. **Apply #1** (`--server-side --force-conflicts`): `replicas` owned by `before-first-apply`, CD tool owns only `[selector, strategy, template, revisionHistoryLimit]`. `replicas` untouched → **20**. ✅ Safe — CD tool doesn't own `replicas`.
2. **Patches #2–#4** (metadata merge patches): `generation` doesn't move → no spec change. Merge patches can't delete by omission. But the recompute finishes the `before-first-apply → CD tool` absorption → CD tool now owns the **entire** legacy set, **including `replicas`** (a field it never declares). Landmine armed.
3. **Apply #5** (same manifest, still omits `replicas`): CD tool now *owns* `replicas` → "owned-but-omitted ⇒ delete" → `replicas` removed → **defaults to `1`** 💥. (`progressDeadlineSeconds` reset to 600 by the same mechanism.)

### Why the fleet cratered
- `replicas: 1` → controller scales 20 → 1, deletes ~19 pods.
- Lone pod under full traffic → OOMKilled before stabilizing → ~0 Ready for 1–2 min.
- HPA scales back up via `/scale` → `kube-controller-manager` re-owns `replicas` → recovers to ~20.

### Why PDB / maxUnavailable didn't help (the key insight)
- **PDB guards only voluntary disruptions via the Eviction API** (drain, descheduler, node upgrade). A `replicas` scale-down deletes pods **directly** via the ReplicaSet controller — **never calls Eviction API → PDB never consulted.**
- **`maxUnavailable: 0` guards rolling *template* updates**, not a `spec.replicas` reduction. A scale-down is allowed to delete surplus pods.
- ⇒ **A PDB does not protect your replica count.** Anything that writes `spec.replicas` bypasses it.

## Quick Reference

```text
SSA rule           : owned + omitted  ⇒  field deleted (defaults apply)
Deployment default : replicas absent  ⇒  replicas = 1
Merge patch        : omission never deletes (only touches keys present)
generation bumps   : only on spec changes (use to find which write hit spec)

before-first-apply : synthetic Update manager on FIRST ssa apply of a
                     client-side-managed object; holds legacy fields;
                     DESIGNED to be absorbed by the applying manager
                     -> drags undeclared fields (e.g. replicas) onto it

Why apply #2 breaks, not #1:
  #1  CD tool does NOT own replicas (before-first-apply does) -> safe
  ~   migration folds before-first-apply onto CD tool -> CD now owns replicas
  #5  CD tool owns replicas, manifest omits it        -> DELETED -> 1
```

**Diagnose**
```bash
# who owns the replica count right now?
kubectl get deploy <name> --show-managed-fields -o yaml   # look for f:replicas owner

# who wrote replicas, and when? (API server audit log)
#   filter: resourceName ~ deployments/<name>, methodName ~ patch|update|apply
#   principalEmail distinguishes CD tool vs autoscaler vs human
```

**Rules of thumb**
- HPA-managed Deployment ⇒ **never** put `replicas` in the manifest — but with SSA, "omit it" is only safe if the applier never *owns* it.
- Client-side apply is immune (no ownership to migrate); SSA is where this bites.
- Fix at the tooling layer: applier must **disclaim `replicas`** / preserve live replica count; don't let the first forced apply consolidate the shim's `replicas` onto it.
- Give workloads memory headroom (`requests` < `limits`) so a replica dip degrades instead of OOM-cascading into a hard outage.

**Anti-patterns**
- Trusting a PDB to keep pods up during *any* disruption (it only covers Eviction-API ones).
- Reading deployment-level `ScalingReplicaSet` events as the root cause — they're the symptom; ownership metadata + audit log are the cause.
- Blaming OOM/memory limits for a fleet-to-zero event without first checking what `spec.replicas` was at the time.
