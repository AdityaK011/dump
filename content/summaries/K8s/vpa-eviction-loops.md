---
title: "Summary: The VPA Eviction Loop"
---

> **Full notes:** [[notes/K8s/vpa-eviction-loops|The VPA Eviction Loop: When Right-Sizing Never Converges -->]]

## Key Concepts

### The symptom
- Single-replica worker evicted ~every 2 min for hours/days. No deploy, no load change, CPU *lower* than usual.
- Reason on the events: `EvictedByVPA` / *"evicted to apply resource recommendation"* (reporting controller `vpa-updater`).
- Long-running queue worker → each eviction aborts in-flight work → message Nack'd + redelivered → next pod killed too → no progress.

### VPA Auto = two halves that must agree
- **Updater** — evicts pods whose *request* is far from the recommendation.
- **Admission Controller** — mutating webhook that rewrites the *new* pod's request to the recommendation.
- Happy path = one eviction: evict → recreate → admission stamps recommended request → request == recommendation → done.
- **No convergence if admission never applies the recommendation** → infinite loop.

### Why it ignites with "nothing changed"
- Trigger is usually **chronic over-provisioning + time**, not an event.
- e.g. request `500m`, usage `~40m`, recommendation `~35m`. As Recommender bounds narrow, the oversized request crosses the eviction threshold — on the autoscaler's own schedule, no human action.
- An eviction often means the pod is too **big**, not too small.

### Why it loops forever
- Diagnosis = chart **request over time**: pinned flat at the template value, never stepping toward the recommendation → admission isn't applying it.
- Cause = **multiple VPAs on one Deployment** (unsupported; contract is one VPA per pod).

### How admission picks among multiple VPAs
1. **`Off`-mode VPAs are skipped** (`continue`) — a recommendation-only "monitor" VPA never participates. Innocent.
2. Among the rest, **oldest `creationTimestamp` wins** (name as tiebreak). **`updateMode` is NOT considered.**
- Trap: a *newer* `Auto` VPA evicts, but an *older* `Initial` VPA with **no recommendation** wins admission → applies nothing → pod stays at `500m` → `Auto` VPA's `35m` never lands → evict forever.
- A higher-level autoscaler that's "off" can still leave an empty stray VPA attached that wins admission by age.

### Amplifiers
- **Single replica** (`minReplicas: 1`) — every eviction = full outage; no pod to cover. (Vanilla VPA won't evict the last pod by default, but operators often relax this.)
- **Work longer than the ~2-min eviction interval** — can never complete; for queues, a Nack/redelivery loop on top of the eviction loop.

### Fix / prevent
- Set the evicting VPA `updateMode: Off` (or static requests) — removing the only evictor stops it immediately.
- **One VPA per Deployment** — delete duplicates; let the operator own it exclusively.
- Don't run VPA `Auto` on single-replica, long-running queue/batch workers.
- Guardrail: refuse `Auto`/`Recreate` on `minReplicas: 1`, or flag multiple VPAs per Deployment.

## Quick Reference

```
VPA Auto = Updater (evicts)  +  Admission Controller (resizes new pod)
           converges ONLY if both act.

Loop fingerprint:
  ~2-min evictions, no deploy, usage << request,
  request line FLAT (never moves toward recommendation)
  => admission isn't applying the recommendation.

Eviction often means TOO BIG, not too small (over-provisioned).

Multiple VPAs on one Deployment (unsupported) -> admission:
  1) skips Off-mode VPAs
  2) picks OLDEST creationTimestamp (mode ignored!)
  => old empty Initial VPA wins admission, applies nothing,
     new Auto VPA's rec never lands -> infinite evictions.

Amplifiers: replicas==1 (full outage) + work > eviction interval
            (queue Nack/redelivery loop).

Fix: evicting VPA -> Off | one VPA per Deployment |
     no Auto on single-replica long-running workers |
     guardrail on minReplicas:1
```
