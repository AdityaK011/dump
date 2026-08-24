---
title: "Elasticsearch as a Service on Kubernetes: Anatomy of a Multi-Tenant Search Platform"
---

## Search is the hard case for Kubernetes

Running a stateless service on Kubernetes is a solved problem. Running *search* is not, and the reasons compound:

- **The data is the point.** A pod that dies takes a shard replica with it. Rescheduling isn't free — it's a network transfer of tens or hundreds of gigabytes.
- **Placement is semantic.** You can't let the scheduler put shards wherever. Which index lives on which nodes is a capacity *and* isolation decision.
- **Storage is immutable in the ways that matter.** A StatefulSet's volume claim template can't be edited. A disk's type can't be changed in place. Those two facts dictate the shape of every migration you'll ever do.
- **Autoscaling has a quantum.** You can't add "one more node" of an index — you add a whole replica copy, which might be twelve nodes.
- **Attribution is genuinely hard.** On a shared cluster, "which tenant is causing this disk load?" has a surprisingly non-obvious answer.

This note is an anatomy of a multi-tenant Elasticsearch platform on Kubernetes — the layers, the state machine, the autoscaling arithmetic, the recurring machinery, and the observability and cost traps. Most of it generalises to any stateful, shard-based system you're asked to run as a platform.

## The layers, and who owns what

```
config layer  (CUE / Helm / whatever renders manifests)
      ↓
Elasticsearch CR  ──→  operator (ECK)  ──→  StatefulSets, Services, Secrets, certs
      ↓
nodeSets  ──→  one StatefulSet each  ──→  one PVC per pod
      ↓
index lifecycle Jobs + CronJobs   (what the operator does NOT do)
      ↓
index-aware autoscaler CRD        (what the operator does NOT do)
```

The operator does more than you'd expect and less than you need.

**What you get free** from a single `Elasticsearch` custom resource: one StatefulSet per nodeSet, a stable ClusterIP service plus per-nodeSet headless services, a generated superuser secret, a full internal PKI (transport certs per node, HTTP certs), a rendered config secret per nodeSet, and a discovery ConfigMap of unicast hosts. Cluster bootstrap, rolling upgrades, and certificate rotation are all handled.

**What you must build yourself:** index creation and schema application, ingestion pipelines, snapshot scheduling, data retention/expiry, index-aware autoscaling, and cost attribution. That gap is the platform.

### Node roles are the first design decision

Elasticsearch nodes specialise, and the split determines your storage footprint entirely:

| Role | Stateful? | Notes |
|---|---|---|
| master | small PVC | cluster state only; 3 for quorum |
| data | **large PVC** | this is your entire storage bill |
| client (coordinating) | **none** | fan-out and merge; pure CPU |
| ingest | **none** | pipeline processing |

Only data nodes carry meaningful storage. In a real fleet the data nodes account for essentially all of the disk cost, while client and ingest nodes are stateless and can be scaled aggressively without touching storage. Worth internalising before you start modelling cost: **your storage bill is a function of data-node count alone.**

## One nodeSet per index, and how shards get pinned there

The layout that makes multi-tenancy tractable is **one nodeSet per index**. Each index gets its own StatefulSet, its own PVCs, its own scaling behaviour. Noisy-neighbour problems mostly disappear because tenants aren't sharing nodes.

But Kubernetes doesn't know about shards, so the pinning is done inside Elasticsearch, by **attribute matching**:

1. Every pod in a nodeSet gets a node attribute — say `node.attr.group=<groupName>` — injected via its config or environment.
2. Every index gets `index.routing.allocation.include.group: <groupName>`.

Elasticsearch will then only allocate that index's shards to nodes whose attribute matches.

The detail worth noticing is that the setting is `include`, not `require` — and `include` takes a **list**. That single fact is what makes live migration between node pools possible: set `include.group: "old,new"`, let the cluster rebalance across both, then narrow to `"new"`. The index setting *is* the migration control plane.

An alternative that's often simpler: give the new nodeSet the **same** attribute value. Elasticsearch immediately treats both pools as eligible and rebalances with no index-setting change at all, and you drain the old one by scaling it down. Fewer moving parts, and it sidesteps any tooling that assumes one group per index.

## Storage is the rigid layer

Three constraints, and everything about migrations follows from them:

1. **A StatefulSet's `volumeClaimTemplates` are immutable.** You cannot change the storage class, size (downward), or any other claim field on an existing StatefulSet. The API server rejects it outright:
   ```
   spec: Forbidden: updates to statefulset spec for fields other than 'replicas',
   'ordinals', 'template', 'updateStrategy', 'revisionHistoryLimit',
   'persistentVolumeClaimRetentionPolicy' and 'minReadySeconds' are forbidden
   ```
2. **One PVC per data pod, one claim template per nodeSet.** Verified across a real fleet: every data pod had exactly one PVC and every data StatefulSet exactly one template. That 1:1:1 chain is what lets you reason about per-volume limits at all.
3. **A cloud disk's *type* can't be changed in place.** Capacity can usually grow; type cannot.

Put together: **any storage change means a new nodeSet plus a data movement.** Not an edit. That's why the migration pattern for stateful systems on Kubernetes is always *add alongside, relocate, drain, remove* — never *modify in place*.

Operators paper over this partially. A good one will detect a *size increase*, patch the PVCs directly, then delete and recreate the StatefulSet with `DeletePropagationOrphan` so the pods survive the swap. But that escape hatch usually fires **only** for size increases — any other claim-template change will be attempted, rejected by the API server, and retried forever. A reconcile error loop on the whole custom resource is a nastier failure than a clean rejection, so it's worth knowing exactly which changes your operator can absorb.

## The index lifecycle as a state machine

Index management doesn't fit either native Kubernetes primitive. A controller is too eager — you don't want a reconcile loop deciding to re-restore a 400 GB snapshot because it briefly couldn't reach the cluster. A CronJob is nonsense — there's no schedule at which "create the index" is correct.

What you want is: **run this pipeline exactly once, when its definition changes.**

### The trick: hash the spec into the object name

```
<verb>-index-<indexName>-<clusterName>-<hash(jobSpec)>
```

- Spec changes → hash changes → new object name → the apply creates a Job → **it runs.**
- Spec unchanged → same name → the Job exists → the apply is a **no-op.**

Exactly-once, change-triggered execution from a plain GitOps apply loop. No controller, no state store — **the Job's existence is the idempotency record.** This generalises nicely to schema migrations, one-shot backfills, and cache warms.

The costs are real, though. You **can't re-run without changing config** — a Job that failed for environmental reasons won't retry on re-apply, so recovery means deleting the object by hand. The hash covers the *whole* spec, so an unrelated image or resource tweak **re-triggers the entire pipeline** (survivable only because each stage is check-then-act idempotent). And with no TTL, finished Jobs accumulate — unbounded growth, but also a free timestamped audit trail of every transition an index has been through.

### The four states

| status | Meaning | Job |
|---|---|---|
| `init` | Index absent — create it and start ingestion | `create-index-…` |
| `active` | Serving; one shard per node; autoscaled | `activate-index-…` |
| `minimum` | Parked; **all** primaries on one node | `minimize-index-…` |
| `deleted` | Skipped entirely | none |

**`minimum` is the interesting one, and it's a cost dial.** `active` and `minimum` emit the *same Job body* — the minimize Job minimizes nothing. The status selects *sizing*:

| | `active` | `minimum` |
|---|---|---|
| shards per node | 1 | N (all primaries) |
| per-node storage | small | ~N× larger |
| autoscaler | **required** | not required |

An index with N primaries normally occupies N nodes. Minimized, it occupies **one** node holding all N shards — hence the much larger disk on that single node. You go from N machines to one and the index stays queryable. That's the right answer for something you must keep online but that serves almost no traffic: far better than delete-and-restore-later.

Note also that `active` is **rejected** unless an autoscaler is configured. That's an operational rule encoded as a type constraint rather than a runbook line — a pattern worth copying.

### The pipeline is a chain of initContainers

Kubernetes runs `initContainers` strictly in order, fail-fast, with a shared `emptyDir`. That's a serviceable linear workflow engine for zero extra infrastructure:

**`init`:** CheckCluster → CheckNodes → (`CreateIndices` → `UpdateSettings`) *or* (`RegisterRepository` → `RestoreSnapshot` → `CheckIndices` → `RerouteShards`) → StartIngestion → VerifySettings

**`active` / `minimum`:** the same, minus the creation/restore step.

What you don't get is a DAG, per-step retries, or resumption from a failed step. Wanting those is the signal to move to a real workflow engine — but a linear check-then-act sequence genuinely doesn't need one.

### One sharp edge worth generalising

These Jobs typically run `backoffLimit: 0` (a single pod failure is terminal — no retry pod) **and** with no resource requests (BestEffort QoS → first to be evicted when the cluster autoscaler consolidates nodes). Those two combine badly: a routine scale-down can permanently kill a lifecycle Job mid-flight.

The fix is an annotation:
```yaml
cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
```

**Rule of thumb: `backoffLimit: 0` + BestEffort QoS is a footgun on any autoscaled cluster.** If a Job matters and can't be retried, give it resource requests, mark it un-evictable, or both.

## Index-aware autoscaling, and its quantum

A generic HPA can't scale a search index correctly, because there's a structural constraint it doesn't know about: **you cannot have a fractional replica.**

So the pattern is a small CRD that sits between the HPA and the cluster. The HPA scales the CRD's node count; the CRD's controller translates that into a legal replica count and writes it to the index:

```
copies             = ceil(Count × shardsPerNode / numberOfShards)
number_of_replicas = copies − 1
desiredShards      = copies × numberOfShards
desiredNodes       = ceil(desiredShards / shardsPerNode)
```

Two things to decode. In source, `ceil(a/b)` is usually written `(a + b - 1) / b` — an integer-division idiom, not meaningful arithmetic. And **the `−1` is the primary**: `number_of_replicas` counts copies *excluding* the primary, so `1` replica means **two** copies exist.

Concretely, on a 2-primary index with one shard per node:

```
number_of_replicas = 6   →   copies = 7

shard 0:   P   R R R R R R      ← 7 copies
shard 1:   P   R R R R R R      ← 7 copies
                                  14 shard instances → 14 nodes
```

### Granularity is a whole copy, not a node

| HPA asks for | primaries | copies | replicas | nodes actually set |
|---|---|---|---|---|
| 24 | 12 | 2 | 1 | 24 |
| **14** | **12** | **2** | **1** | **24** |
| 36 | 12 | 3 | 2 | 36 |

The middle row is the lesson: the HPA asked for 14, you can't hold 1.17 copies, so it rounds **up** to two full copies and 24 nodes. **The scaling step size is the primary shard count** — twelve nodes at a time on a twelve-primary index.

Two consequences follow immediately. You need a **tier cap** (advance at most one copy per reconcile), or a scale-out tries to stand up an entire tier of pods at once and deadlocks under resource pressure. And a configured **replica floor** (`max(computed, floor)`) can override the computation entirely — when I checked real numbers, seven of eight indices matched the formula and the eighth was explained by its floor. Look for that before concluding your model is wrong.

### Why two autoscalers can never share one index

This is the most transferable insight here. The controller derives **shared state** (`number_of_replicas`, a property of the *index*) from **local state** (the node count of the one pool it manages).

With two controllers on one index — the obvious thing to try when migrating between node pools — each computes a *different* replica count from its own node count, and both write it. The replica count oscillates between tiers and the shards thrash.

> **A controller that writes shared state derived from local state cannot be duplicated.**

If you need two pools serving one index, you need *one* controller spanning both. Well-designed CRDs of this shape expose exactly that: a field listing *additional* pools to treat as one logical unit. Check for it before planning a migration around "turn the old one off, the new one on" — that approach either creates a fight or leaves a window with no replica management at all.

## The recurring machinery

Three things run on a clock, and one of them has a surprising cost consequence.

**Snapshots** — one CronJob per cluster, typically every 15 minutes. Incremental, to object storage.

**Retention purges** — one CronJob per index running a delete-by-query. Cadences in practice split sharply: some indices every **60 seconds**, others **once daily**.

**Scheduled autoscaler bounds** — cron-adjusted HPA min/max. Underrated: reactive autoscaling always *lags* a ramp, so for a known diurnal pattern, raise the floor at 08:45 and let the autoscaler handle only the unpredictable part.

### Deletes don't free space — merges do

In Lucene-based engines, deleting a document writes a **tombstone**. The bytes return only when a **merge** rewrites the containing segment and drops deleted docs on the way through.

The governing knob is `index.merge.policy.deletes_pct_allowed`: the maximum percentage of deleted documents tolerated before merges reclaim. Historically it defaulted to **33** with a valid range of **20–50** — the floor of 20 existing specifically to stop merge IO from overwhelming nodes. Modern versions **lowered the default to 20**, following a Lucene change that found the write-amplification cost of doing so was small.

Now combine that with a delete-by-query every 60 seconds: a continuous trickle of tombstones keeps the deleted-document percentage pressed against the threshold, which keeps the merge policy continuously rewriting segments. The result is **sustained read-and-write IO rather than bursts** — a permanently elevated IO floor for that index.

That matters enormously on storage that bills provisioned IOPS separately from capacity, with a free performance tier covering the baseline. A minute-purge index may sit permanently *above* the free tier while an otherwise identical daily-purge index sits comfortably inside it. **Same data, same query load, and a cron expression decides the IO bill.**

Two cautions if you go hunting for this. If your version's default is already 20, an explicit `20` is a **no-op**, not a tightening — check before chasing it. And purge cadence is *one* contributor, not *the* explanation: when I measured a real fleet, the single hottest index had **no purge job at all**.

## Observability: attribution is the actual problem

This is where I got the most wrong before measuring, so it's worth spelling out.

The intuitive way to find hot disks is a node-level metric — the agent's `system.io.*` family, read from `/proc/diskstats`, filtered to the device names your data volumes use:

```
max:system.io.r_s{cluster:X AND (device:sdb OR device:sdc)} by {host}
```

**This cannot attribute load to a tenant, and here's why.** `system.io` is emitted once per *node* and has no concept of pods — so it cannot be filtered by namespace or workload. The only scoping available is the node and a literal device name. And `sdb` means nothing more than "the second block device the kernel enumerated on this machine". On a shared cluster, that's frequently a completely different workload's disk. The query answers *"what is the busiest second disk anywhere in the cluster?"* — not *"which index is hot?"*

The fix is to use the engine's **own** metric. Elasticsearch reports `fs.io_stats` for the devices backing *its own configured data paths* — so the device selection is done by the process that knows which device is its own. And because the monitoring integration polls a specific pod, each sample inherits that pod's tags: cluster, index group, and even the PVC name. Attribution becomes measured rather than inferred.

Both metrics ultimately read the same kernel counters. The difference is entirely in **which device gets selected, and whether the sample knows who owns it.**

### Three counting traps

Measuring a real fleet turned up three places where the obvious count is wrong:

- **Pods per node isn't 1:1.** I'd assumed one data pod per machine (a dashboard note said so). Measured: 390 data pods across 353 nodes — 320 nodes with one, 29 with two, 4 with three. So a `by {host}` panel silently merges two or three distinct volumes on ~9% of nodes.
- **PVC count ≠ pod count.** PVCs are retained after scale-down, so a nodeSet can have 18 PVCs backing 14 pods. Retained volumes still cost capacity — so **count PVCs for cost and pods for machine relief.** They are different numbers answering different questions.
- **Metric-derived capacity ≠ actual capacity.** The engine's filesystem-total metric over-read the real PVC inventory substantially. Use it for *relative change over time*; use the Kubernetes API for absolute counts.

And a semantics trap: within one metric family, some fields are **rates** and some are **cumulative counters**. I tried to derive average IO size by dividing bytes by operations and got 3.9 MB per read — obvious nonsense, because one was cumulative and the other a rate. Always verify whether a counter is monotonic before doing arithmetic on it. A quick check: plot it and look for a monotonic ramp with resets on pod restart.

## Scaling down safely is the operator's best trick

The reason *add-alongside-then-drain* works at all is that a good operator makes downscale shard-aware. Reducing a nodeSet's count triggers, in order:

1. Set `cluster.routing.allocation.exclude._name` for the leaving nodes.
2. Wait until each of those nodes holds **zero shards**.
3. *Then* delete the pod.

That means "scale the old nodeSet down" is a safe drain — you never hand-manage allocation filters, and there's no leftover cluster-level exclusion to forget about (a classic footgun). Combined with the same-attribute trick from earlier, a full node-pool migration is: add the new nodeSet → let the cluster rebalance → scale the old one down → remove it. Reversible at every step until the last.

Two caveats. Downscale invariants usually **stall if the cluster isn't healthy**, so a perpetually-yellow cluster blocks the whole migration. And the shard-count-per-node cap (`total_shards_per_node`) is sized for your *current* node count — during a migration you briefly have ~2× the nodes, then fewer, so a cap that was fine before can leave shards unassigned at the drain step.

## When storage bills IOPS separately

Newer cloud disk types unbundle what used to be one price: capacity, provisioned IOPS, and provisioned throughput each bill separately, with a free performance tier per volume. That changes the economics in a way that's worth doing the algebra on.

Moving a volume saves a fixed amount per GiB-month on capacity, and adds a per-IOPS cost above the free tier. Set those equal and you get a **break-even peak IOPS** that depends only on volume size:

| Volume size | Break-even peak IOPS (illustrative) |
|---|---|
| 100 GiB | ~5,000 |
| 200 GiB | ~7,000 |
| 500 GiB | ~13,000 |

Below its break-even, a workload is cheaper on the new storage; above it, the IOPS bill outweighs the capacity saving. **Bigger volumes tolerate more IOPS**, because the capacity saving scales with size while the free tier doesn't.

The genuinely counterintuitive consequence: since the free tier is **per volume** and the IOPS charge scales with **volume count**, a nodeSet with many small volumes is expensive to provision even at modest IOPS, while a nodeSet with few large volumes can be hot and still cheap. In one real comparison, the *hottest* index in the fleet was among the cheapest to provision standing — because it had only 18 volumes — while a cooler index with 84 volumes cost nearly 3× more. **Volume count, not heat, drives provisioned-IOPS cost.**

This gives you a clean selection rule instead of an all-or-nothing migration: compute each workload's break-even, move the ones below it, and leave the rest.

## Key takeaways

- **The operator handles the cluster; you build the platform.** Bootstrap, certs, rolling upgrades, and services come free. Index creation, schemas, ingestion, retention, index-aware autoscaling, and cost attribution do not.
- **Storage is the rigid layer.** Claim templates are immutable and disk types can't change in place, so every storage change is *add alongside, relocate, drain, remove* — never an edit.
- **Pin shards with node attributes, and prefer `include` (a list) over `require`** — that's what makes live migration between node pools possible.
- **Hash a spec into an object name** for exactly-once, change-triggered Jobs from a plain apply loop. The object's existence is the idempotency record.
- **Design a "parked" state.** Collapsing all primaries onto one larger node keeps an index queryable at a fraction of the footprint.
- **Encode operational invariants as type constraints**, not runbook prose ("an active index must have an autoscaler").
- **Index autoscaling quantises to whole replica copies**, so the step size is the primary shard count. Budget for a tier cap and look for a replica floor.
- **A controller writing shared state derived from local state cannot be duplicated.** Span both pools with one controller.
- **Deletes don't free space; merges do.** Purge cadence plus merge threshold set the steady-state IO floor — and on unbundled storage, a cron expression can dominate the bill.
- **Node-level disk metrics can't attribute to a tenant.** Use the engine's own IO stats, which know which device is theirs and carry the pod's identity.
- **Verify your counts and your counters.** Pods per node isn't 1:1, PVCs outlive pods, and mixing a cumulative counter with a rate produces confident nonsense.
