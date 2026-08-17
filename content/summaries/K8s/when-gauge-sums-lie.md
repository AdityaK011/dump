---
title: "Summary: When Gauge Sums Lie"
---

> **Full notes:** [[notes/K8s/when-gauge-sums-lie|When Gauge Sums Lie: Interval Metadata, Ghost Series, and Duplicate Emitters -->]]

## Key Concepts

### The symptom
- Cluster CPU requests: API server **36,755 cores**, dashboard **26,236** — 28.6% short, on a metric feeding ~200 dashboards.
- Dead ends: container exclusions (empty), agent coverage (1,246/1,246 nodes), tag coverage (99.8%).
- **The clue:** per-container breakdown shows every container *present*, every value scaled by the **same ~0.72**.
- ⇒ Filtering *removes* series; it cannot multiply them. A uniform multiplier means a **query-time transform**, not data loss.

### Act 1 — wrong interval metadata
- A gauge point carries a value + timestamp, **no duration**. To sum a gauge across churning series, the engine must infer each series' lifetime.
- Lifetime-weighting modifiers prorate by *time a point can vouch for*; the only source for "one point = N seconds" is the metric's **declared submission interval** metadata.
- `factor = declared_interval / actual_spacing`. Declared **15**, real **20** ⇒ **×0.75** on every series, at every time range.
- **Falsifiable test:** a second metric declared **5** ⇒ predicted **×0.25** *before* measuring ⇒ measured **0.237**. Weighted read 16.5 TiB vs true 66.1 TiB (**−75%**).
- **Control:** a third metric from the same check declares **20** (correct) and *is* accurate weighted. Isolates the fault to the metadata value, not the modifier.
- Cadence must be verified from **check config** (`min_collection_interval: 20` — an override of the 15s framework default!) **and** `Total Runs ÷ uptime` (29,953 / 599,077 s = **20.0006 s**). Point spacing from the query API is *query resolution*, useless for this.
- The integration's own catalogue shipped an **empty** interval column ⇒ wrong value stored platform-side, possibly your data to fix, not a vendor bug.

### Act 2 — ghost series at coarse rollups
- Long ranges widen buckets (1-month view ⇒ ~4h buckets). A naive sum asks *"which series have data anywhere in this bucket?"* and adds each one's own average.
- Averaging a series over **its own** points never dilutes it ⇒ a pod alive 1 of 4 hours counts as if alive 4. **One deploy = one ghost.**
- **Asymmetry:** containers churn hourly (numerator inflates), nodes churn weekly (denominator honest) ⇒ `requested/allocatable` drifts past **100%** — physically impossible, hence self-announcing. Absolute counts inflate silently by 20–40%.
- Fix = the same lifetime-weighting modifier — but metrics with declared **interval 0** refuse it. Then: short ranges only, or use inside ratios.
- Trap: the *same query* is accurate at 1h and wrong at 1mo, with no visual discontinuity.

### Act 3 — duplicate emitters (leader-elected exporters)
- Leadership is exclusive; **publishing isn't**. A terminating pod keeps serving its last gauge values (gauges don't vanish when the loop stops) while the new leader publishes fresh ones ⇒ two series, both true, `sum` doubles. Observed 2×, 3×, once **5×**.
- **What one point means decides everything:**
  - **Gauge** = the whole answer ⇒ two emitters = two **copies** ⇒ `sum` doubles ❌
  - **Counter/histogram** = one event that happened once ⇒ two emitters = **disjoint halves** ⇒ `sum` correct ✅ (frozen counter emits delta 0; frozen gauge re-asserts full value)
- **Industry practice:** `max`/`avg` across replicas (not `sum`); leadership gauge + PromQL join (`metric * on(pod) group_left leader == 1`); unregister/reset collectors on shutdown or loss of leadership; still scrape all replicas for liveness. *Not* exporter-side pre-aggregation.
- **When `avg` isn't available:** a value existing only as a **sum across cells** (e.g. 20-cell 2-D band grid summed to a node count) makes `sum` do two jobs — across cells (wanted) and across emitters (not). PromQL nests (`sum by (pool) (max by (pool,cell) (...))`); many hosted languages can't.
- **Ratios cancel exactly:** `(N·x)/(N·y) = x/y`. Percentage columns stay trustworthy during duplication; the absolute count beside them doesn't.
- **Corollary trap:** patching the count's aggregator to `min` breaks the ratios if that query is *also* the denominator ⇒ use a **separate query**.

### Act 4 — pre-aggregate at the source
- All three bugs are artifacts of **reconstructing cluster-level facts from per-object series at query time**.
- Some joins are simply inexpressible: container metrics have **no node dimension**, so "requests per node pool" cannot be queried in any language.
- Do the join in a controller (informer caches already hold nodes + pods) and emit the *result*: ~**500 stable series** instead of ~33,000 churning ones.
- ⇒ no churn (Act 2 impossible), no lifetime to prorate (Act 1 irrelevant), semantics fixed once: running pods only, **scheduler-effective requests** = `max(main + sidecars, init peak) + pod overhead` (plain init containers release their reservation — naive all-container sums overstate).
- Emit the cluster total too as a **continuous self-test** against a provider-native metric (~1% agreement = live correctness signal).
- Still doesn't fix Act 3 — it's still a leader-elected gauge exporter.

## Quick Reference

```text
Lifetime weighting : factor = declared_interval / actual_point_spacing
                     declared 15, real 20 -> x0.75
                     declared  5, real 20 -> x0.25
                     declared 20, real 20 -> x1.00  (correct)

Ghost counting     : coarse bucket + series shorter than bucket
                     -> each series counted at FULL value
                     -> numerator (pods, hourly churn) inflates
                        denominator (nodes, weekly churn) doesn't
                     -> ratio can exceed 100% = impossible = your tell

Duplicate emitters : gauge publishing the whole answer, N publishers
                     sum  -> N x truth        avg/max -> truth
                     ratio of two sums -> N cancels exactly

Error sizing       : overlap ~1 scrape (20-60s), handover ~1/15h
                     1 week -> ~0.1%    30 min -> ~2%
                     5 min  -> ~10%     1 min  -> up to 2x
```

**Diagnose**
```bash
# 1. is it a transform or missing data? break down per object and look for a
#    CONSTANT ratio vs ground truth. constant => transform, lumpy => data.

# 2. what does the platform think the interval is?
GET /api/v1/metrics/<metric>        # look for the stored submission interval

# 3. what is the real cadence? (two independent ways)
grep min_collection_interval /etc/.../conf.d/<check>.d/conf.yaml.default
<agent> status                       # Total Runs / uptime = seconds per run
#    NOT from returned point spacing -- that's query resolution

# 4. is the integration even shipping that metadata?
#    check the integration's metadata.csv interval column (often empty)

# 5. how many processes are publishing?
sum:<metric>{...} by {host,endpoint,pod}   # >1 series with identical values?
```

**Rules of thumb**
- Ground-truth every aggregate against the source of record (`kubectl` / API server) **before** building on it.
- Round-fraction error (×0.75, ×0.5, ×0.25) ⇒ suspect metadata-driven arithmetic, not the data.
- Make one **falsifiable prediction** on a second metric; it converts a plausible story into a proven mechanism — and catches you when it's wrong.
- Never `sum` a gauge that publishes a whole-system value across replicas.
- Prefer **percentages over counts** wherever duplication is possible.
- State the **valid time range** next to any number whose accuracy depends on bucket width.
- Size a transient before engineering around it — the workaround often costs more than the bug.
- If a fact needs a join the metric model can't express, **emit it from a controller** instead of reconstructing it.
- Put the caveat **on the widget** (which columns are exact, which aggregator must not change), not in a doc nobody opens.

**Anti-patterns**
- Concluding "the agent drops containers / exclusions are set" from a uniform shortfall.
- Trusting a documented framework default (15s) over the check's actual shipped config (20s).
- Measuring submission cadence from query-API point spacing.
- Applying a lifetime-weighting modifier because it's "best practice for gauges" without verifying the interval metadata — and note some tooling **auto-appends it**, silently.
- Combining a lifetime-weighting modifier with an explicit coarse rollup (order-of-operations can divide by the rollup/interval ratio — verify magnitudes).
- Fixing an inflated count by changing an aggregator that is also serving as a ratio denominator.
