---
title: "Exactly-Once Jobs Without a Controller: Index Lifecycle on Kubernetes"
---

## The problem with running a search index on Kubernetes

Kubernetes gives you two native ways to make something happen: a **controller** that continuously reconciles desired state against reality, or a **CronJob** that fires on a clock. Neither fits "create this index once, when someone changes its definition."

A controller is overkill and dangerous — index creation isn't idempotent in an interesting way, and you don't want a reconcile loop deciding to re-restore a 400 GB snapshot because it briefly couldn't reach the cluster. A CronJob is worse: there's no schedule at which "create the index" is the right answer.

What you actually want is: **run this pipeline exactly once, whenever its definition changes, and never again.** That's a deployment-shaped problem, not a controller-shaped one.

There's a neat trick for getting exactly that out of plain `Job` objects and a GitOps apply loop, with no operator involved. This note walks through it, plus the state machine it drives, and then the part that surprised me most — the autoscaler arithmetic that turns a node count into a whole number of index copies, and why that means two autoscalers can never share one index.

## The trick: put a hash of the spec in the name

The config layer generates a Job whose name embeds a **hash of the Job's own spec**:

```
<verb>-index-<indexName>-<clusterName>-<hash(jobSpec)>
```

where `<verb>` is `create`, `activate`, or `minimize` depending on the index's declared lifecycle status.

That's the whole mechanism, and the semantics fall out of it:

- **Spec changes** → hash changes → *new object name* → the apply creates a Job that didn't exist → **it runs.**
- **Spec unchanged** → same name → the Job already exists → the apply is a **no-op**.

You get exactly-once, change-triggered execution using nothing but `kubectl apply` and the fact that object names are unique. No controller, no state store, no "have I already done this?" bookkeeping. The Job's existence *is* the bookkeeping.

This is a genuinely useful pattern for any "run once per config revision" task — schema migrations, one-shot data backfills, cache warms.

### The tradeoffs, which are real

It's clever, not free.

**You can't re-run without changing something.** If a Job fails for an environmental reason — the cluster was briefly yellow, a node got drained — re-applying identical config does nothing, because the name still matches an existing (failed) Job. Recovery means deleting the Job object by hand or perturbing the spec. There's no `kubectl rollout restart` equivalent.

**The hash covers the *whole* spec, so unrelated changes re-trigger the pipeline.** Bump a container image, adjust a resource request, add an annotation — the hash moves, and the entire index pipeline re-runs. Most stages are written to be idempotent (check-if-exists before create), which is what makes this survivable, but it means the blast radius of a trivial edit is "the full pipeline executes again."

**Old Jobs accumulate.** With `ttlSecondsAfterFinished: null` they're never garbage collected. That's arguably a feature — you get a complete, timestamped audit trail of every lifecycle transition the index has been through, readable with `kubectl get jobs`. But it's unbounded growth you have to prune eventually.

## The state machine

One field drives everything: a `status` on the index definition, with four values.

| status | Meaning | Job emitted |
|---|---|---|
| `init` | Index doesn't exist yet — create it and start ingestion | `create-index-…` |
| `active` | Serving traffic; one shard per node; autoscaled | `activate-index-…` |
| `minimum` | Parked; **all** primaries collapsed onto one node | `minimize-index-…` |
| `deleted` | Nothing — the index is skipped entirely | none |

A second family of Jobs runs in parallel at the *node pool* level (`init-`, `activate-`, `minimize-<nodeGroup>`), setting node counts and handing off to the autoscaler. The split matters: the index-level Jobs talk to the search cluster over HTTP; the node-level Jobs talk to Kubernetes.

### `minimum` is the interesting state

`active` and `minimum` produce **the same Job body**. The minimize Job doesn't minimize anything — it runs the identical sequence of health checks and ingestion start as activate. All the actual difference lives in the *sizing* the status selects:

| | `active` | `minimum` |
|---|---|---|
| `shardsPerNode` | `1` | `N` (all primaries) |
| per-node storage | `100Gi` | `600Gi` |
| autoscaler | **required** | not required |

So `minimum` is a cost dial. An index with N primary shards normally occupies N nodes, one shard each. Minimized, it occupies **one** node holding all N shards — which is why the disk has to grow roughly N-fold on that single node. You go from N machines to one, and the index stays queryable.

This is the right shape for an index you must keep online but that serves almost no traffic: a seasonal dataset, a deprecated-but-not-deleted index, a disaster-recovery copy. Instead of deleting and re-restoring it later, you park it.

The validation is worth noting too: `active` is **rejected** unless an autoscaler is configured. The system refuses to let you run a serving index without a scaling policy — a nice example of encoding an operational rule as a type constraint rather than a runbook line.

## The pipeline is a chain of initContainers

Each lifecycle Job is a sequence of `initContainers` — which Kubernetes runs **strictly in order, fail-fast** — followed by a single regular container that reports the result.

**`init` (create):**

1. **CheckCluster** — block until the search cluster is healthy
2. **CheckNodes** — block until this index's own data nodes are present
3. Then one of two paths:
   - **from schema:** `CreateIndices` → `UpdateSettings` (replica count, shard-allocation routing, refresh interval, merge policy)
   - **from snapshot:** `RegisterRepository` → `RestoreSnapshot` → `CheckIndices` → `RerouteShards`
4. **RunFeeder** — start ingestion, if the index has a subscription
5. **CheckSettings** — verify the settings actually landed

**`active` / `minimum`:** the same, minus step 3. CheckCluster → CheckNodes → RunFeeder → CheckSettings.

Using `initContainers` as a workflow engine is a pattern worth recognising. You get sequential execution, fail-fast, and a shared `emptyDir` workspace between stages, for zero extra infrastructure. What you *don't* get is a DAG, per-step retries, resumption from a failed step, or any visualisation. If you find yourself wanting those, that's the signal to move to a real workflow engine — but a linear check-then-act pipeline genuinely doesn't need one.

Each stage is a small `bash` + `curl` script against the search cluster's HTTP API, running in a generic cloud-CLI image. There's no compiled application code anywhere in this system — the "code" is shell embedded in the config layer, and the config layer's job is to render it into Job manifests. That's a deliberate tradeoff: trivially auditable and modifiable, but untestable in isolation and easy to get subtly wrong in quoting.

### One sharp edge: these Jobs are deliberately fragile

The Jobs run with `backoffLimit: 0` and **no resource requests** (so, BestEffort QoS). Both choices compound:

- `backoffLimit: 0` means a single pod failure is terminal — the Job fails outright with `BackoffLimitExceeded` and no retry pod is created.
- BestEffort pods are the **first** thing the cluster autoscaler evicts when it consolidates nodes.

Put together: a routine scale-down can kill a lifecycle Job mid-flight, permanently. The mitigation is an annotation:

```yaml
cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
```

which pins the node until the Job finishes. Worth internalising as a general rule: **`backoffLimit: 0` and BestEffort QoS is a hazardous combination** on any cluster with an active autoscaler. If a Job matters and can't be retried, either give it resource requests or mark it un-evictable — ideally both.

## The recurring work: three CronJob families

Alongside the change-triggered Jobs, three things run on a clock.

**Snapshots — one CronJob per cluster, every 15 minutes.** Straightforward incremental snapshots to object storage.

**Purge-by-query — one CronJob per index.** Each runs a delete-by-query to expire documents. The cadences split into two very different regimes: some indices run **every 60 seconds**, others **once a day**. That difference matters enormously, for reasons in the next section.

**Scheduled autoscaler bounds — cron-adjusted HPA min/max.** Rather than letting the autoscaler discover a diurnal traffic pattern reactively every day, the bounds are moved on a schedule (e.g. `0 9 * * *`, `0 23 * * *`). This is an underrated technique for predictable daily cycles: reactive autoscaling always lags the ramp, so if you *know* traffic rises at 09:00, raise the floor at 08:45 and let the autoscaler handle only the unpredictable part.

## Why a minute-interval purge shows up as sustained disk IO

This is the part that connects the job schedule to a storage bill, and it's not obvious.

In Lucene-based engines, **deleting a document doesn't free space.** It writes a tombstone marking the doc as deleted. The bytes come back only when a **merge** rewrites the containing segment and drops deleted docs on the way through.

The threshold that governs this is:

```
index.merge.policy.deletes_pct_allowed
```

the maximum percentage of deleted documents tolerated in an index before merges kick in to reclaim. Historically it defaulted to **33** with a valid range of 20–50; the floor of 20 existed specifically to stop people from drowning nodes in merge IO. Modern versions have **lowered the default to 20**, following a Lucene change that found the write-amplification cost of going from 33% to 20% was small.

Now combine that with a delete-by-query every 60 seconds. You get a continuous trickle of tombstones, which keeps the deleted-document percentage pressed against the threshold, which keeps the merge policy continuously rewriting segments. The result is **sustained read-and-write IO rather than bursts** — a permanently elevated floor on IOPS and throughput for that index.

That's a big deal if you're moving onto storage that bills provisioned IOPS separately from capacity, where a free performance tier covers the baseline and you pay for anything above it. An index purging every minute may sit permanently above the free tier, while an otherwise identical index purging daily sits comfortably inside it. Same data, same query load, ~100× difference in the IO bill — decided entirely by a cron expression.

**Two cautions if you go looking for this.** First, check your version's default before assuming an explicit `deletes_pct_allowed: 20` is a tightening — if the default is already 20, that line is a no-op and not your culprit. Second, and more important: purge cadence is *one* contributor, not *the* explanation. When I actually measured this, the hottest index in the fleet had **no purge job at all** — so something else drove its load. Merge pressure is a hypothesis to test per index, not a conclusion to assume.

## The autoscaler arithmetic: node counts become whole index copies

The last piece is the most conceptually interesting, and the source of a real design constraint.

An index-aware autoscaler can't just set a node count, because a search index has a structural quantum: you can't have a fractional replica. Its job is to translate a *desired node count* into a legal *replica count*.

The inputs are the desired node count (from the HPA), the number of primary shards, and how many shards each node should hold. The computation:

```
copies             = ceil(Count × shardsPerNode / numberOfShards)
number_of_replicas = copies − 1
desiredShards      = copies × numberOfShards
desiredNodes       = ceil(desiredShards / shardsPerNode)
```

In source you'll see the ceiling written as the integer idiom `(a + b - 1) / b`, which is easy to misread as meaningful arithmetic. It isn't — it's just `ceil(a/b)`.

### The `−1` is the primary

The step that confuses everyone is `number_of_replicas = copies − 1`. It's pure terminology: **`number_of_replicas` counts copies *excluding* the primary.** So `number_of_replicas: 1` means **two** copies exist. `copies` includes the primary; the setting doesn't; hence the subtraction.

Worked through, on an index with **2 primary shards** and one shard per node:

```
number_of_replicas = 6   →   copies = 7

shard 0:   P    R R R R R R      ← 7 copies
shard 1:   P    R R R R R R      ← 7 copies
                                   14 shard instances → 14 nodes
```

And backwards: `copies = ceil(14 × 1 / 2) = 7`, so `number_of_replicas = 6`. Consistent.

### Scaling granularity is a whole index copy

Here's the consequence that actually shapes operations:

| HPA asks for | primaries | copies | replicas | nodes actually set |
|---|---|---|---|---|
| 24 | 12 | 2 | 1 | 24 |
| **14** | **12** | **2** | **1** | **24** |
| 36 | 12 | 3 | 2 | 36 |

The middle row is the lesson. The HPA asked for 14 nodes; you cannot hold 1.17 copies of an index; so it rounds **up** to two full copies and 24 nodes. **The autoscaling step size is a whole index copy — on a 12-primary index, that's 12 nodes at a time.**

Two things follow directly. First, you need a **tier cap** — advance at most one copy per reconcile — or a scale-out event tries to stand up an entire tier of pods simultaneously, which deadlocks under cluster resource pressure. Second, a configured **replica floor** can override the computed value entirely, so the node count isn't always what determines replicas. When I checked real numbers, seven of eight indices matched the formula exactly and the eighth didn't — the floor was the likely explanation. Always look for the `max(computed, floor)` before concluding your model is wrong.

### Why two autoscalers can never share one index

This is the generalisable insight, and it bites during migrations.

The autoscaler derives a **cluster-wide setting** (`number_of_replicas`, a property of the index) from its **own local state** (the node count of the one node pool it manages). That's fine with one autoscaler. With two — say you've added a second node pool on new hardware and want to drain the old one — each computes a *different* replica count from its *own* node count, and both write it. The index's replica count oscillates between tiers, and the shards thrash.

So: **any controller that writes shared state derived from local state cannot be duplicated.** If you need two node pools serving one index, you need *one* controller that spans both — not two controllers each managing one. Well-designed CRDs of this shape expose exactly that escape hatch: a field listing *additional* node pools to treat as one logical pool.

Two lessons here. The narrow one: check for that field before you plan a migration around "turn the old autoscaler off and the new one on" — that approach either creates a fight or leaves a window with no replica management at all. The broader one: when you see a controller computing shared state from local state, "just run two of them" is never the answer.

## Key takeaways

- **Hash the spec into the object name** to get exactly-once, change-triggered Jobs from a plain GitOps apply loop — no controller, no state store. The Job's existence is the idempotency record.
- Accept its costs knowingly: no re-run without perturbing config, unrelated spec changes re-trigger the pipeline, and finished Jobs accumulate (which doubles as an audit trail).
- **`initContainers` are a serviceable linear workflow engine.** Sequential, fail-fast, shared workspace, zero infrastructure. Reach for a real engine when you need a DAG, per-step retry, or resumption.
- **`backoffLimit: 0` + BestEffort QoS is a footgun** on an autoscaled cluster. A routine node consolidation permanently kills the Job. Pin it with `safe-to-evict: "false"`, give it requests, or both.
- **A "minimize" state is worth designing in.** Collapsing every primary shard onto one larger node keeps an index queryable at a fraction of the footprint — far better than delete-and-restore-later.
- **Encode operational rules as type constraints.** "An active index must have an autoscaler" is more reliable as a validation error than as a line in a runbook.
- **Deletes don't free space; merges do.** A delete-by-query cadence plus a merge threshold together set an index's steady-state IO floor. On storage that bills IOPS separately, a cron expression can dominate the bill.
- **Index autoscaling quantises to whole replica copies**, so the step size is the primary shard count, not one node. Budget for a tier cap and look for a replica floor.
- **A controller deriving shared state from local state can't be duplicated.** Span both pools with one controller instead.
