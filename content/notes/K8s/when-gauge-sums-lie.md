---
title: "When Gauge Sums Lie: Interval Metadata, Ghost Series, and Duplicate Emitters"
---

## A number that was wrong by exactly 28%

You're measuring how much CPU a Kubernetes cluster has reserved — the sum of every running pod's CPU requests. Two sources disagree:

- The API server, queried directly with `kubectl`: **36,755 cores**
- The monitoring dashboard, summing the agent's `*.cpu.requests` metric: **26,236 cores**

A 28.6% shortfall on a metric that 200 dashboards depend on. The obvious hypotheses all sound plausible and all turn out to be wrong:

- **"The agent is excluding containers."** Container-exclusion config is empty. No exclusion patterns anywhere.
- **"Some nodes aren't reporting."** The agent DaemonSet is on 1,246 of 1,246 nodes.
- **"Tag coverage is patchy, so the filter drops series."** 99.8% of the metric's value carries the expected host tags.

Then the clue that reframes everything. Break the metric down per container and compare each one to the API server:

| Container | API server | Monitoring | ratio |
|---|---|---|---|
| a | 1,470 | 1,072 | 0.73 |
| b | 716 | 508 | 0.71 |
| c | 304 | 216 | 0.71 |
| d | 220 | 158 | 0.72 |
| e | 29 | 21.2 | 0.73 |

Every container is **present**. Every value is scaled by the **same** ~0.72.

That single observation kills the entire "data is missing" family of explanations. Filtering, exclusions, coverage gaps — they all *remove series*. None of them can multiply every surviving series by a constant. A uniform multiplier means the data arrived intact and something **transformed it at query time**.

This note is about three separate ways query-time aggregation silently corrupts gauge metrics, all of which I hit in sequence: wrong **interval metadata**, **ghost series** at coarse rollups, and **duplicate emitters** from leader-elected exporters. They're independent bugs with independent fixes, and each one is invisible — no error, no warning, just plausible-looking numbers.

## Act 1: the transform that needs metadata to be honest

### Why the transform exists at all

Summing a gauge across thousands of short-lived series is harder than it looks. Consider a metric reporting each container's CPU request. Containers come and go constantly. When you ask for "total requested cores" over a window, the query engine has to decide, for each series, *how much of that window the series was actually alive*.

A gauge data point carries a value and a timestamp. It does **not** carry a duration. So the engine can't tell, from the data alone, whether a point represents one second of existence or one hour.

Monitoring platforms solve this with a "weight by actual lifetime" modifier — a function that prorates each series by the fraction of the bucket it can prove it existed. Without it, a container that lived 5 minutes inside a 4-hour bucket gets counted as though it lived the full 4 hours (more on that in Act 2). The modifier exists precisely to stop dead series from being credited forever:

```
dying series:   ●    ●    ✝  (deleted — no more points, ever)
```

If the rule were "assume alive until the next point," a deleted series would be credited indefinitely. So the modifier credits **only time a data point explicitly vouches for**. And the only thing that tells it how much time one point vouches for is the metric's **declared submission interval**, stored as metadata on the metric.

### The failure mode

The metric in question declared an interval of **15 seconds**. The agent's check actually ran every **20 seconds**.

```
actual reports:   ●                   ●                   ●        every 20s
believed alive:   |----- 15s ----|gap|----- 15s ----|gap|          metadata says 15
```

Each point vouches for 15 seconds; the next point arrives 20 seconds later. The engine concludes the container was **absent for 5 seconds of every 20** — for a container that never stopped existing — and discounts every value accordingly:

```
factor = declared_interval / actual_spacing = 15 / 20 = 0.75
```

There's the uniform 0.72. Not data loss. Arithmetic, applied faithfully to a lie in the metadata.

### Making it falsifiable

A correlation this clean still deserves a real test, so: predict before measuring.

If the model is right, the factor should equal `declared ÷ actual` for **any** metric. So look up a *different* metric from the same check, read its declared interval, and predict its distortion before querying it.

The memory-requests metric declared an interval of **5 seconds**. Prediction: `5 / 20 = 0.25` — it should read **75% low**.

Measured, against two independent sources of truth:

| Source | value |
|---|---|
| Cloud-provider-native metric (correct 60s interval) | **66.1 TiB** — truth |
| Cluster-state metric, running pods only | 66.6 TiB (+0.7%) |
| Agent metric, **unmodified** | 69.5 TiB (+4.6%) |
| Agent metric, **weighted** | **16.5 TiB** — 25% of truth |

Ratio: **0.237** against **0.25** predicted. Two metrics, two different wrong intervals, each distortion equal to `declared ÷ 20`.

And the control case that proves the modifier itself isn't broken: a third metric from the same check declares **20** — matching reality — and *is* accurate when weighted (30.8 vs 30.9 pods/node against `kubectl`). Same check, same agent, same pipeline, same query path. The only variable is the metadata value.

### Establishing the real cadence (harder than it sounds)

The naive approach — measure the spacing between returned data points — **does not work**. Query APIs return points at *query resolution*, not submission cadence. Asking for a 10-minute window returned points every 2 seconds for all three metrics. That tells you nothing about how often the agent actually scraped.

Two things that do work:

**Read the check's config.** The check shipped an explicit interval in the agent image:

```
/etc/.../conf.d/<check>.d/conf.yaml.default
    min_collection_interval: 20
```

Note this was *not* the generic framework default (15s) — the check overrides it. Had I assumed the documented default, I'd have concluded 15s and "proven" there was no bug.

**Count executions over uptime.** The agent's status output reports total runs per check:

```
Total Runs: 29,953   over 599,077 s of uptime   =  20.0006 s/run
```

Exact. Two independent confirmations, one from config and one from behaviour.

### Whose bug is it?

The tempting conclusion is "the integration ships wrong metadata." Checking that assumption is worth the five minutes: the integration's published metric catalogue (`metadata.csv`) has an **empty** interval column for every one of these metrics.

```
<check>.cpu.requests,gauge,,core,,The requested cpu cores,0,...
<check>.memory.requests,gauge,,byte,,The requested memory,0,...
```

So the wrong values aren't shipped by the integration at all — they're stored on the platform side, either inferred at ingestion or written to that account at some point in the past. Which changes who owns the fix: possibly nobody's bug, just stale account data you can correct yourself in the metric's metadata settings.

Worth internalising as a heuristic:

> When a metric reads a suspiciously **round fraction** of ground truth — ×0.75, ×0.5, ×0.25 — suspect a query-time transform against wrong metadata *before* you suspect the data.

Data loss is lumpy. Arithmetic is clean.

## Act 2: ghost series at coarse rollups

Fix the modifier and a second, independent problem surfaces — this one triggered by **time range** rather than metadata.

### One point per container, buckets that widen

At a 1-hour view, chart points are ~1 minute apart. At a 1-month view, the engine can't return 43,000 points, so each point covers roughly **4 hours**.

Now: how does a naive sum build one 4-hour bucket? It asks *"which series have data anywhere in this bucket?"* and adds each one's own average.

Take one pod slot occupied by a 2-core service that redeploys hourly. Between 12:00 and 16:00, four different pods occupy it in sequence — four distinct series, because series identity includes pod name.

| Series | alive | own average | counted as |
|---|---|---|---|
| pod A | 12–13h | 2 | 2 |
| pod B | 13–14h | 2 | 2 |
| pod C | 14–15h | 2 | 2 |
| pod D | 15–16h | 2 | 2 |
| **total** | | | **8** |

One 2-core slot reported as 8 cores. Averaging a series over *its own* points can never dilute it — pod A isn't penalised for being dead three of the four hours. **One deploy = one ghost.**

### Why the denominator doesn't inflate with it

This is what makes it dangerous rather than merely noisy. Consider `requested ÷ allocatable`:

- **Containers** churn hourly — every deploy, every scale event. Numerator balloons.
- **Nodes** churn weekly. One series, alive the whole bucket, counted once. Denominator stays honest.

The ratio drifts upward as you zoom out, and eventually exceeds **100%** — a physically impossible reading for "fraction of capacity reserved," since the scheduler will never bind a pod whose requests exceed a node's remaining allocatable.

That impossibility is a gift. It's a self-announcing bug. The same mechanism silently inflates *absolute* core counts by 20–40% with no such tell.

### The fix, and when you can't have it

The lifetime-weighting modifier from Act 1 is exactly the fix: prorate each series by its alive fraction, and ghosts contribute only their real lifetime.

But it requires trustworthy interval metadata — and some metrics ship with **interval `0`**, meaning the platform has no basis for computing an alive fraction and refuses the modifier outright. For those metrics there is no query-side fix. You get two options:

1. **Stay at short ranges**, where bucket width < typical series lifetime, so no bucket ever contains two occupants of the same slot.
2. **Use them only inside ratios** (see Act 3), where the inflation partly cancels.

Note the trap this creates: the same metric is *accurate* at a 1-hour range and *wrong* at a 1-month range, with no visual discontinuity. Two engineers can pull the same query and disagree, both correct.

## Act 3: duplicate emitters and the sum-vs-avg problem

The third failure mode has nothing to do with time or metadata. It comes from **how many processes are publishing**.

### Singleton computation, multiple publishers

A common pattern: a controller with 2+ replicas for availability, leader election so only one does the work, and each replica exposing a metrics endpoint. The leader computes cluster-wide values and publishes them.

Here's the seam. Leadership is exclusive, but **publishing isn't**. When the leader is replaced:

- The terminating pod keeps serving its **last computed values** — gauges don't vanish when the compute loop stops.
- The new leader starts publishing fresh values.
- Both endpoints get scraped. Each carries its own pod-identity tags, so the platform sees **two distinct series**.

Both series say "this pool totals 7,500 cores." Both are *true statements about the same thing*. A `sum:` across them returns **15,000**.

I measured 2×, 3×, and once **5×** (36,090 against a true ~7,200) during overlapping pod generations.

Note this is not the ghost problem from Act 2. These series genuinely coexist; no rollup width is involved. Same symptom, different cause — which is exactly why I misdiagnosed it the first time.

### The critical asymmetry: what one data point means

Whether duplicate emission hurts depends entirely on what a single point *represents*:

| | Gauge of the whole answer | Event/histogram observation |
|---|---|---|
| One point means | "the total is 7,500" | "one node measured 0.87" |
| Two emitters produce | two **copies** of the answer | **disjoint halves** of an event stream |
| `sum` across them | doubles ❌ | correct — assembles the timeline ✅ |
| Frozen emitter during shutdown | keeps re-asserting the full value ❌ | cumulative counter frozen → delta 0 ✅ |

Two guards at shift handover: one writes *"500 people are in the building"* on a whiteboard every minute; the other writes *one logbook line per person entering*. Sum everything written during the handover hour and the whiteboard guard reports 1,000 people. The logbook guard's count is simply correct — no line exists twice.

So: **counters and histograms are naturally duplicate-safe. Gauges are not.**

### What the industry actually does

Surveying the practice, the dominant patterns are:

1. **Aggregate with `max`/`avg` across replicas, not `sum`.** Standby replicas report zeros or duplicates; `max`/`avg` is robust to both. This is the single most-recommended fix and it costs nothing.
2. **Export a leadership gauge and filter in queries.** Controller frameworks already expose a `leader_election_master_status`-style gauge (1 for leader, 0 otherwise), and PromQL can join on it: `metric * on(pod) group_left leader == 1`. Elegant — but only if your query language supports vector joins.
3. **Unregister or reset the collectors on shutdown / loss of leadership**, so a terminating process publishes nothing. This is the pattern for the specific case of "process stays up and keeps being scraped while shutting down."
4. **Scrape every replica anyway** — you still want standby liveness and lease-acquisition failures visible.

Notably, the standard answer is *not* to pre-aggregate in the exporter to dodge the problem. Make duplicates **harmless at read time**.

### The case where `avg` isn't available

Pattern 1 works when a logical value is **one series**. It breaks when the value only exists as a **sum across several series**.

Concrete example: classify every node into a 2-D grid — CPU-fill band × pod-slot-fill band — and emit a node count per cell. A pool's total node count is then the sum of its 20 cells. Now `sum:` is doing two jobs at once:

- summing across **cells** — wanted
- summing across **emitters** — not wanted

And the query engine cannot tell them apart. PromQL can express the fix with nested aggregation:

```promql
sum by (pool) ( max by (pool, cpu_band, slot_band) (nodes_by_band) )
#              └─ max across emitters first ─┘
# └─ then sum across cells ─┘
```

Many hosted query languages have no nested space aggregation, so this exact fix is unavailable. Which leaves three imperfect options:

- **`min` over the window.** Duplicate emission only ever inflates, so a window minimum is never inflated. Cost: over a long window it reports the *lowest* value rather than the average — a semantic change, not an error.
- **Show only ratios** (below).
- **Fix the emission** (pattern 3) and let `sum` be honest.

### Ratios cancel exactly — the free lunch

The genuinely delightful property. If numerator and denominator both inflate by the same factor `N`:

```
(N × slot_bound) / (N × total)  =  slot_bound / total
```

`N` divides out **exactly**. So `% of nodes that are X` is trustworthy even during duplicate emission, while the absolute count beside it is not. In a dashboard table, that means percentage columns are safe and raw-count columns aren't — a distinction worth writing on the widget, because it is deeply non-obvious to the next reader.

One corollary that bit me: if you "fix" the raw count by switching its aggregator to `min`, and that query is *also* the ratio denominator, you get `avg ÷ min` and break the cancellation — trading one wrong column for three. Use a **separate query** for the displayed count.

### Sizing the error instead of fearing it

Duplicate emission is a *transient*, so its impact is a function of how often it happens and how long each overlap lasts, versus your viewing window:

```
overlap ≈ one scrape interval (~20–60 s)
handover rate ≈ 1 per 15 h  (measured: 13 pods / 7 days, 2 replicas)

1-week window   → ~0.1 %   error   (negligible)
30-min window   → ~2 %             (fine)
5-min window    → ~10 %            (visible)
1-min window    → up to 2×         (wrong)
```

Which flips the intuition: for long-range dashboards `avg` is essentially exact, and the `min` workaround costs more (biasing toward the window minimum) than the problem it solves. Measure the transient before engineering around it.

Also worth measuring separately per environment: the same setup churned **once per 2.8 h** in a development cluster (spot-node evictions) versus **once per 15 h** in production. Same code, 5× different exposure.

## Act 4: stop reconstructing, start pre-aggregating

Step back from the three bugs and notice what they share: all of them are artifacts of **reconstructing a cluster-level fact from per-object series at query time**.

Some facts can't be reconstructed at all. A useful one: *"how much CPU is requested on the nodes of a given node pool?"* Container-level metrics carry namespace, pod, container — but **no node dimension**, because the metric's resource model doesn't include one. So there is no query, in any language, that joins pod requests onto node pools. The join simply isn't expressible.

The alternative is to do the join **where the data lives** — inside a small controller with informer caches that already hold both nodes and pods — and emit the *result*:

```
node_pool_cpu_requested_cores{pool, machine_type, max_pods}
node_pool_cpu_allocatable_cores{pool, ...}
node_pool_pods_running{pool, ...}
node_pool_pods_capacity{pool, ...}
node_pool_nodes_by_band{pool, cpu_band, slot_band}   # 2-D distribution
```

The properties that follow are the whole point:

- **~500 stable series instead of ~33,000 churning ones.** No per-object churn ⇒ Act 2's ghost problem cannot occur at any time range.
- **No lifetime to prorate** ⇒ Act 1's metadata bug is irrelevant.
- **Correct semantics fixed at the source**: running pods only, scheduler-effective requests (`max(main + sidecars, init peak) + pod overhead`, mirroring what the scheduler actually reserves — plain init containers release their reservation, so naively summing all containers overstates it).
- **A built-in self-test**: emit the cluster total too, and compare it continuously against an independent provider-native metric. Agreement within ~1% is a live correctness signal; divergence is an alarm rather than a mystery.

Verified against API-server snapshots, the emitted values matched to four significant figures (178.012 vs 178.0 cores; 774 vs 774 pods), with the 2-D cells correct node-by-node.

What pre-aggregation does **not** solve is Act 3 — it's still a leader-elected gauge exporter, so duplicate emission during handovers still applies. Different layer, different fix.

## Takeaways

1. **A uniform multiplier is never data loss.** Filtering removes series; only a transform can scale them all equally. Round fractions (×0.75, ×0.5, ×0.25) point at metadata-driven arithmetic.
2. **Gauge points carry no duration.** Any "sum a gauge across many series" operation must infer lifetime, and that inference is only as good as the metric's declared interval.
3. **Verify the declared interval against reality** — from the check's shipped config *and* from runs ÷ uptime. Don't trust documented framework defaults, and don't try to measure cadence from returned point spacing (that's query resolution).
4. **Make one falsifiable prediction.** Reading a second metric's interval and predicting its distortion *before* measuring converts a plausible story into a proven mechanism. It also catches you when the story is wrong.
5. **Check whether the metadata is even shipped by the integration** before blaming the vendor. Empty upstream + wrong stored value = your data to fix.
6. **The same query can be right at 1 hour and wrong at 1 month.** Bucket width crossing typical series lifetime is the threshold. State the valid range next to the number.
7. **Never `sum` a gauge that publishes the whole answer.** Use `max`/`avg` across replicas. Counters and histograms are duplicate-safe; gauges are not.
8. **Ratios cancel duplication exactly.** Percentages survive what absolute counts don't — and if you patch a count's aggregator, make sure it isn't also a ratio denominator.
9. **Size transients before engineering around them.** A 60-second overlap once per 15 hours is 0.1% error on a weekly view. The workaround can easily cost more than the bug.
10. **If a fact needs a join that the metric model can't express, stop querying and start emitting.** Pre-aggregating at the source collapses cardinality, kills the churn artifacts, and lets you fix semantics once instead of in every query.
11. **Write the caveat on the artifact, not in a doc nobody opens.** Which columns are exact, which are approximate, which aggregator must not be changed — on the widget itself.

---

## Related notes

- [[notes/K8s/node-log-pipeline-silent-failures|Alive But Not Shipping]] — the logging counterpart: a node's log agent stalls with zero errors, and the alert that would have caught it is silenced when its metrics exporter dies and the series goes *absent* rather than zero
- [[notes/K8s/kubernetes-autoscaling|Kubernetes Autoscaling]] — where the pod and node churn that produces ghost series comes from
