---
title: "Summary: Elasticsearch as a Service on Kubernetes"
---

> **Full notes:** [[notes/K8s/elasticsearch-as-a-service-on-kubernetes|Elasticsearch as a Service on Kubernetes: Anatomy of a Multi-Tenant Search Platform -->]]

## Key Concepts

### Why search is the hard case
- Data is the point — a dead pod costs a 100 GB+ network transfer.
- Placement is **semantic**, not schedulable — which index lives where is a capacity + isolation decision.
- Storage is immutable where it matters: claim templates can't be edited, disk types can't change in place.
- Autoscaling has a **quantum** — you add a whole replica copy, not a node.
- Attribution ("which tenant is hot?") is genuinely non-obvious.

### Layers, and the ownership split
```
config layer → Elasticsearch CR → operator → StatefulSets/Services/Secrets/certs
                    ↓ nodeSets → 1 StatefulSet each → 1 PVC per pod
                    ↓ lifecycle Jobs + CronJobs        (NOT the operator)
                    ↓ index-aware autoscaler CRD       (NOT the operator)
```
- **Free from one CR:** StatefulSet per nodeSet, ClusterIP + per-nodeSet headless Services, superuser Secret, full internal PKI (transport + HTTP certs), config Secret, unicast-hosts ConfigMap, bootstrap, rolling upgrades, cert rotation.
- **You build:** index creation/schemas, ingestion, snapshot scheduling, retention, index-aware autoscaling, cost attribution.

### Node roles decide your storage bill
| Role | Stateful? |
|---|---|
| master | small PVC (3 for quorum) |
| **data** | **large PVC — the entire storage bill** |
| client (coordinating) | none |
| ingest | none |

⇒ Storage cost is a function of **data-node count alone**. Client/ingest scale freely.

### One nodeSet per index + attribute pinning
- Layout: one nodeSet per index ⇒ own StatefulSet, own PVCs, own scaling; noisy neighbours mostly gone.
- Kubernetes doesn't know shards, so pin inside ES: pod gets `node.attr.group=<g>`; index gets `index.routing.allocation.include.group: <g>`.
- **`include` takes a list** — set `"old,new"`, rebalance across both, narrow to `"new"`. The index setting *is* the migration control plane.
- Simpler alternative: give the new nodeSet the **same** attribute → both eligible instantly, no index change; drain by scaling down.

### Storage is the rigid layer
1. `volumeClaimTemplates` **immutable** — API rejects: *"updates to statefulset spec for fields other than 'replicas', 'ordinals', 'template', 'updateStrategy', 'revisionHistoryLimit', 'persistentVolumeClaimRetentionPolicy', 'minReadySeconds' are forbidden"*.
2. **1 PVC per data pod, 1 template per nodeSet** (verified fleet-wide) — the 1:1:1 chain is why per-volume limits are reasonable about.
3. Cloud disk **type** can't change in place (capacity usually can grow).

⇒ **Every storage change = new nodeSet + data movement.** Pattern is always *add alongside → relocate → drain → remove*.
- Operators absorb **size increases** only (patch PVCs, recreate STS with `DeletePropagationOrphan` so pods survive). Any *other* template change is attempted, rejected, and retried forever → reconcile error loop on the whole CR.

### Lifecycle: hash-the-spec Jobs
```
<verb>-index-<index>-<cluster>-<hash(jobSpec)>
```
- Spec changed → new name → runs. Unchanged → name exists → **no-op**. Exactly-once, change-triggered, **no controller** — the Job's existence is the idempotency record.
- Costs: can't re-run without perturbing config; hash covers the *whole* spec so unrelated tweaks re-trigger everything; no TTL ⇒ unbounded Jobs (but a free audit trail).

| status | Meaning |
|---|---|
| `init` | create + start ingestion |
| `active` | serving, 1 shard/node, autoscaled |
| `minimum` | parked, **all** primaries on one node |
| `deleted` | skipped |

- **`minimum` is a cost dial.** `active` and `minimum` emit the *same Job body* — status selects sizing: shards/node `1 → N`, storage `~N×`, autoscaler required → not. N nodes → **1**, still queryable. Better than delete-and-restore-later.
- `active` is **rejected without an autoscaler** — operational rule as a type constraint.
- Pipeline = `initContainers` (in order, fail-fast, shared `emptyDir`): CheckCluster → CheckNodes → (Create+UpdateSettings | RegisterRepo+Restore+Check+Reroute) → StartIngestion → VerifySettings. No DAG/retry/resumption — that's the signal to use a real engine.
- ⚠️ `backoffLimit: 0` + **BestEffort** QoS ⇒ a routine cluster-autoscaler scale-down **permanently kills** the Job. Fix: `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` + resource requests.

### Autoscaling arithmetic
```
copies             = ceil(Count × shardsPerNode / numberOfShards)
number_of_replicas = copies − 1
desiredNodes       = ceil(copies × numberOfShards / shardsPerNode)
```
- `(a+b-1)/b` in source is just integer `ceil(a/b)`.
- **The `−1` is the primary** — `number_of_replicas` excludes it, so `1` replica = **2** copies.

| HPA asks | primaries | copies | replicas | nodes set |
|---|---|---|---|---|
| 24 | 12 | 2 | 1 | 24 |
| **14** | **12** | **2** | **1** | **24** |
| 36 | 12 | 3 | 2 | 36 |

- Can't hold 1.17 copies ⇒ rounds **up**. **Step size = primary shard count** (12 nodes at a time).
- ⇒ need a **tier cap** (one copy/reconcile) or scale-out deadlocks; a **replica floor** `max(computed, floor)` can override the math.
- **Two autoscalers can't share one index:** each derives shared state (`number_of_replicas`, an *index* property) from local state (its own pool's count); both write it ⇒ oscillation + shard thrash. **A controller writing shared state derived from local state cannot be duplicated** — span both pools with one, via the CRD's "additional pools" field.

### Recurring machinery + the IO surprise
- Snapshots per cluster (~15m); retention purge per index (**60s** vs **daily**); cron-adjusted HPA bounds (reactive scaling lags a *known* ramp — raise the floor before it).
- **Deletes don't free space — merges do.** Delete writes a tombstone; bytes return when a merge rewrites the segment.
- `index.merge.policy.deletes_pct_allowed` = max % deleted tolerated. Old default **33**, range **20–50** (floor of 20 to prevent merge-IO overload); modern default **20**.
- 60s purge ⇒ continuous tombstones ⇒ percentage pinned at threshold ⇒ **continuous segment rewrites** ⇒ **sustained** IO, not bursts. On unbundled storage a minute-purge index sits *above* the free tier while its daily-purge twin sits inside. **A cron expression decides the IO bill.**
- ⚠️ If the version default is already 20, an explicit `20` is a **no-op**. And purge is *one* factor — the hottest index measured had **no purge job**.

### Observability: node-level disk metrics can't attribute
- `system.io.*` is emitted **per node** from `/proc/diskstats` — **no pod concept**, so it can't be filtered by namespace/workload. `sdb` just means "2nd block device on this machine", frequently another workload's disk.
- ⇒ `max:system.io.r_s{...device:sdb OR sdc} by {host}` answers *"busiest 2nd disk anywhere"*, **not** *"which index is hot"*.
- Fix: the engine's own `fs.io_stats` — ES picks the device under **its own** data path, and the sample carries the **pod's** tags (cluster, index group, PVC). Same kernel counters; the difference is *device selection* + *ownership labelling*.

### Three counting traps (all measured)
- **Pods per node isn't 1:1** — 390 data pods on 353 nodes (320×1, 29×2, 4×3) ⇒ `by {host}` merges 2–3 volumes on ~9% of nodes.
- **PVC count ≠ pod count** — PVCs are retained after scale-down (18 PVCs / 14 pods). **Count PVCs for cost, pods for machine relief.**
- **Metric capacity ≠ real capacity** — the fs-total metric over-read the PVC inventory. Use it for *relative change*; use the API for absolutes.
- **Rates vs cumulative counters:** dividing bytes by ops gave 3.9 MB/read — nonsense, because one was cumulative. Plot for a monotonic ramp + restart resets before doing arithmetic.

### Safe downscale = the operator's best trick
1. Set `cluster.routing.allocation.exclude._name` for leaving nodes → 2. wait for **zero shards** → 3. delete pod.
- ⇒ "scale the old nodeSet down" is a safe drain; no hand-managed filters, no leftover cluster-level exclusion.
- Caveats: downscale invariants **stall if the cluster isn't healthy**; `total_shards_per_node` sized for the old node count can leave shards unassigned at the drain step (~2× nodes mid-migration, fewer after).

### Unbundled storage economics
- Capacity, provisioned IOPS, provisioned throughput bill separately, with a **per-volume** free performance tier.
- **Break-even peak IOPS** depends only on volume size (illustrative): 100 GiB ≈ 5,000 · 200 GiB ≈ 7,000 · 500 GiB ≈ 13,000. Bigger volumes tolerate more IOPS (saving scales with size; free tier doesn't).
- **Counterintuitive:** free tier is per *volume* and the charge scales with volume *count* ⇒ many-small-volumes is expensive even when cool; few-large-volumes can be hot and cheap. Hottest index (18 volumes) was cheaper to provision than a cooler one with 84. **Volume count, not heat, drives provisioned-IOPS cost.**
- ⇒ selection rule, not all-or-nothing: move workloads below their break-even, leave the rest.

## Quick Reference

```text
Free from the operator : StatefulSets, Services, superuser Secret, internal PKI,
                         config Secret, unicast-hosts CM, bootstrap, upgrades
You build              : index create/schema, ingestion, snapshots, retention,
                         index-aware autoscaling, cost attribution

Storage bill           = f(data-node count)   # client/ingest are stateless
Chain                  : 1 data pod = 1 ES node = 1 PVC = 1 volume
Immutable              : volumeClaimTemplates (size increase is the ONE exception)
Disk type              : cannot change in place -> new nodeSet + relocate

Shard pinning          : pod  node.attr.group=<g>
                         index index.routing.allocation.include.group: <g>
                         include takes a LIST -> "old,new" then "new" = migration

Exactly-once Job       : name = <verb>-<obj>-<hash(spec)>
                         changed -> runs ; same -> no-op
Lifecycle              : init | active | minimum | deleted
minimum                : shards/node 1->N, storage ~Nx, N nodes -> 1

Replica math           : copies = ceil(Count*sPN / primaries)
                         number_of_replicas = copies - 1     # -1 = the primary
                         step size = primaries, NOT 1 node

Deletes                : tombstone only; space returns on MERGE
deletes_pct_allowed    : old default 33 (range 20-50); modern default 20
purge @60s             : continuous merges -> SUSTAINED IO floor

Attribution            : system.io  = per NODE, no pod concept -> cannot attribute
                         fs.io_stats = engine's own device + pod tags -> can
```

**Diagnose**
```bash
# minimal bare cluster: ONE CR is enough (operator does the rest)
kubectl apply --dry-run=server -f es.yaml    # validates against the operator webhook

# what did the operator create for me?
kubectl get sts,svc,secret,cm -n <ns> | grep '^.*<cluster>-es'

# lifecycle history of an index (finished Jobs are the audit trail)
kubectl get jobs -n <ns> | grep -E '(create|activate|minimize)-index-<name>'

# did a lifecycle Job get evicted rather than fail on its own?
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'   # BestEffort = evictable

# is a cron cadence setting the IO floor?
kubectl get cronjobs -n <ns> -o custom-columns=\
'NAME:.metadata.name,SCHEDULE:.spec.schedule,LAST:.status.lastScheduleTime'

# replica count vs the formula (or is a floor overriding?)
GET /<index>/_settings?include_defaults=true
```

**Rules of thumb**
- Bare test cluster = **one `Elasticsearch` CR**. Add the autoscaler CRD only after the index exists.
- Set `node.store.allow_mmap: false` unless you've raised `vm.max_map_count` on the nodes.
- Never plan a storage change as an edit — plan it as a new nodeSet plus a relocation.
- Any Job that matters: resource requests **and** `safe-to-evict: "false"` if `backoffLimit: 0`.
- Move autoscaler *bounds* on a schedule for known diurnal patterns.
- Before a node-pool migration, look for the CRD's "additional pools" field.
- Encode operational invariants as validation errors, not runbook prose.

**Anti-patterns**
- Attributing tenant disk load from node-level `/proc/diskstats` metrics filtered by device name.
- Running two instances of a controller that derives shared state from local state.
- Assuming an explicit setting differs from the default — check the version or chase a no-op.
- Treating delete-by-query as space reclamation (it's a tombstone write).
- `backoffLimit: 0` + BestEffort on an autoscaled cluster.
- Doing arithmetic across a rate and a cumulative counter.
- Assuming one pod per node, or that PVC count equals pod count.
