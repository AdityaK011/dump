---
title: "Summary: Exactly-Once Jobs Without a Controller"
---

> **Full notes:** [[notes/K8s/index-lifecycle-jobs-on-kubernetes|Exactly-Once Jobs Without a Controller: Index Lifecycle on Kubernetes -->]]

## Key Concepts

### The problem
- Kubernetes offers **controllers** (continuous reconcile) and **CronJobs** (clock-triggered). Neither fits *"run once, when the definition changes."*
- A controller might re-restore a 400 GB snapshot on a transient error; no cron schedule makes "create the index" correct.

### The trick: hash the spec into the name
```
<verb>-index-<indexName>-<clusterName>-<hash(jobSpec)>
```
- Spec changes → hash changes → **new object name** → Job created → **runs once**.
- Spec unchanged → same name → Job exists → apply is a **no-op**.
- Exactly-once change-triggered execution from `kubectl apply` alone. **The Job's existence is the idempotency record.**

### Its tradeoffs
- **No re-run without changing config.** A Job that failed environmentally (yellow cluster, drained node) won't retry on re-apply — delete the object or perturb the spec.
- **Hash covers the whole spec** → an image bump or resource tweak re-triggers the entire pipeline. Survivable only because stages are check-then-act idempotent.
- **`ttlSecondsAfterFinished: null`** → finished Jobs never GC'd. Unbounded growth, but a free timestamped audit trail.

### The state machine
| status | Meaning | Job |
|---|---|---|
| `init` | Index absent — create + start ingestion | `create-index-…` |
| `active` | Serving; 1 shard/node; autoscaled | `activate-index-…` |
| `minimum` | Parked; **all** primaries on one node | `minimize-index-…` |
| `deleted` | Skipped entirely | none |

A parallel family of **node-pool-level** Jobs (`init-`/`activate-`/`minimize-<nodeGroup>`) sets node counts. Index Jobs talk HTTP to the search cluster; node Jobs talk to Kubernetes.

### `minimum` is a cost dial
- `active` and `minimum` emit the **same Job body** — the minimize Job minimizes nothing. The status selects *sizing*.
- `shardsPerNode`: `1` → `N` (all primaries) · storage `100Gi` → `600Gi` · autoscaler required → not required.
- N primaries normally occupy N nodes; minimized they occupy **one** (hence the ~N× disk). Right for keep-online-but-idle indices — better than delete-and-restore-later.
- `active` is **rejected without an autoscaler** — an operational rule encoded as a type constraint.

### Pipeline = initContainers chain
- `initContainers` run **strictly in order, fail-fast**; a final container reports the result.
- `init`: CheckCluster → CheckNodes → (**CreateIndices → UpdateSettings** | **RegisterRepo → RestoreSnapshot → CheckIndices → RerouteShards**) → RunFeeder → CheckSettings
- `active`/`minimum`: same **minus step 3**.
- Pattern: sequential + fail-fast + shared `emptyDir`, zero infra. **No DAG, no per-step retry, no resumption** — those are the signals to move to a real workflow engine.
- Stages are `bash` + `curl` in a generic cloud-CLI image. No compiled code anywhere: auditable, but untestable in isolation.

### The fragility footgun
- Jobs run `backoffLimit: 0` (single failure is terminal, no retry pod) **and** with no resource requests (**BestEffort** → first evicted on node consolidation).
- ⇒ a routine cluster-autoscaler scale-down **permanently kills** a lifecycle Job.
- Mitigation: `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"`.

### Three CronJob families
- **Snapshots** — per cluster, every 15m.
- **Purge-by-query** — per index; two regimes: **every 60s** vs **once daily**. Hugely consequential (below).
- **Scheduled autoscaler bounds** — cron-moved HPA min/max (`0 9 * * *`, `0 23 * * *`). Reactive autoscaling always lags a known ramp; raise the floor before it.

### Deletes don't free space — merges do
- Deleting a doc writes a **tombstone**. Bytes return only when a **merge** rewrites the segment.
- `index.merge.policy.deletes_pct_allowed` = max % deleted docs tolerated before merges reclaim. Historically default **33**, range **20–50** (floor of 20 existed to prevent merge-IO overload). Modern versions default to **20**.
- Purge every 60s ⇒ continuous tombstones ⇒ percentage pressed against threshold ⇒ **continuous segment rewriting** ⇒ **sustained** read+write IO, not bursts.
- On storage billing IOPS separately from capacity, a minute-purge index may sit permanently *above* the free tier while a daily-purge twin sits inside it. **A cron expression can dominate the IO bill.**
- ⚠️ Two cautions: (1) if your version already defaults to 20, an explicit `20` is a **no-op**, not a tightening; (2) purge is *one* contributor — the hottest index measured had **no purge job at all**.

### Autoscaler arithmetic: node count → whole index copies
```
copies             = ceil(Count × shardsPerNode / numberOfShards)
number_of_replicas = copies − 1
desiredShards      = copies × numberOfShards
desiredNodes       = ceil(desiredShards / shardsPerNode)
```
- `(a + b - 1) / b` in source is **just integer `ceil(a/b)`** — not meaningful arithmetic.
- **The `−1` is the primary.** `number_of_replicas` counts copies *excluding* the primary, so `1` replica = **2** copies. `copies` includes it; the setting doesn't.

### Granularity is a whole copy, not a node
| HPA asks | primaries | copies | replicas | nodes set |
|---|---|---|---|---|
| 24 | 12 | 2 | 1 | 24 |
| **14** | **12** | **2** | **1** | **24** |
| 36 | 12 | 3 | 2 | 36 |

- Can't hold 1.17 copies ⇒ rounds **up**. **Step size = primary shard count** (12 nodes at a time on a 12-primary index).
- ⇒ needs a **tier cap** (one copy per reconcile) or scale-out deadlocks trying to create a whole tier at once.
- ⇒ a **replica floor** (`max(computed, floor)`) can override the computed value — check it before concluding your model is broken.

### Why two autoscalers can't share one index
- The autoscaler derives **shared state** (`number_of_replicas`, a property of the *index*) from **local state** (its own node pool's count).
- Two of them ⇒ each computes a different replica count from its own count, both write it ⇒ **replica count oscillates, shards thrash**.
- **General rule: a controller writing shared state derived from local state cannot be duplicated.** Use one controller spanning both pools — well-designed CRDs expose an "additional node pools" field for exactly this.
- Migration corollary: "turn the old autoscaler off, the new one on" either creates a fight or leaves a window with **no** replica management.

## Quick Reference

```text
Exactly-once Job    : name = <verb>-<obj>-<hash(spec)>
                      spec changed -> new name -> runs
                      spec same    -> name exists -> no-op
                      failure recovery = delete Job or perturb spec

Lifecycle states    : init | active | minimum | deleted
minimum             : shardsPerNode 1->N, storage 100Gi->600Gi,
                      N nodes -> 1 node, still queryable
active              : REJECTED without an autoscaler (type constraint)

Job body            : activate == minimize (status drives SIZING, not steps)
Pipeline            : initContainers, in order, fail-fast, shared emptyDir

Replica math        : copies = ceil(Count * shardsPerNode / numberOfShards)
                      number_of_replicas = copies - 1      # -1 = the primary
                      1 replica == 2 copies
                      step size = numberOfShards, NOT 1 node

Deletes             : tombstone only; space returns on MERGE
deletes_pct_allowed : max % deleted tolerated. old default 33, range 20-50
                      modern default 20. lower = less space, more CPU/IO
purge @every 1m     : continuous tombstones -> continuous merges
                      -> SUSTAINED IO floor (not bursts)
```

**Diagnose**
```bash
# what lifecycle transitions has this index been through? (audit trail)
kubectl get jobs -n <ns> | grep -E '(create|activate|minimize)-index-<name>'

# did a lifecycle Job get evicted rather than fail on its own?
kubectl describe job <job>        # BackoffLimitExceeded + no retry pod
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'   # BestEffort = evictable

# is a cron cadence driving the IO floor?
kubectl get cronjobs -n <ns> -o custom-columns=\
'NAME:.metadata.name,SCHEDULE:.spec.schedule,LAST:.status.lastScheduleTime'

# does the replica count match the formula, or is a floor overriding it?
GET /<index>/_settings   # compare number_of_replicas vs ceil(nodes*sPN/primaries)-1
```

**Rules of thumb**
- Need run-once-per-config-revision? Hash the spec into the name before reaching for a controller.
- Any Job that matters: give it resource requests **and** `safe-to-evict: "false"` if `backoffLimit: 0`.
- Design a "parked" state for stateful workloads — consolidating onto fewer, larger nodes beats tearing down and rebuilding.
- Move autoscaler *bounds* on a schedule for known diurnal patterns; let the autoscaler handle only the unpredictable part.
- Before planning a migration around an index-aware autoscaler, look for an "additional node pools" field.
- Encode operational invariants as validation errors, not runbook prose.

**Anti-patterns**
- Running two instances of a controller that derives shared state from local state.
- Assuming an explicit setting differs from the default — check the version first, or you'll chase a no-op.
- Attributing sustained IO to one mechanism without measuring per-index (the hottest index may have none of it).
- `backoffLimit: 0` + BestEffort on an autoscaled cluster.
- Treating delete-by-query as a space-reclamation operation — it's a tombstone write; merges do the reclaiming.
