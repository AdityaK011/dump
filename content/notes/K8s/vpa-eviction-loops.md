---
title: "The VPA Eviction Loop: When Right-Sizing Never Converges"
---

## The symptom that looks impossible

A single-replica worker — say a queue consumer that processes large files and takes a few minutes per message — starts getting killed roughly **every two minutes**, around the clock, for a day and a half. Each kill aborts whatever it was doing. Because it's a queue worker, the in-flight message is negative-acknowledged and redelivered, the replacement pod picks up the same message, and it too is killed before finishing. The queue makes no progress for ~36 hours.

Nothing was deployed. The container image is unchanged. CPU usage is *lower* than it has ever been. Nobody touched any autoscaling setting. And yet the pod is being evicted on a metronome by something labelled *"evicted to apply resource recommendation."*

This is a **Vertical Pod Autoscaler (VPA) eviction loop**, and it's one of those failures that's completely logical once you see the pieces — but baffling if you reach for the usual suspects (a bad release, a CPU spike, an OOM). This note is about how the loop forms, why it can run forever instead of converging after a single eviction, and how a second, "dormant" VPA on the same workload is what turns a one-time resize into an infinite loop.

For the VPA building blocks (Recommender / Updater / Admission Controller, and the four update modes), see [[notes/K8s/kubernetes-autoscaling|Kubernetes Autoscaling]]. This note assumes them and focuses on the failure mode.

## VPA Auto is two cooperating halves

The single most important thing to internalize: **VPA in `Auto`/`Recreate` mode is two independent components that must agree for the system to settle.**

- **The Updater** watches running pods. If a pod's resource *request* is far enough from the Recommender's target (beyond an eviction tolerance), it **evicts** the pod.
- **The Admission Controller** is a mutating webhook on pod creation. When the replacement pod is created, it **rewrites the request** to the recommended value.

The happy path is a single step:

```
Updater evicts oversized pod
   -> ReplicaSet recreates it
   -> Admission Controller stamps the new pod with the recommended request
   -> new request == recommendation
   -> Updater is satisfied -> done. (one eviction, total)
```

Convergence depends entirely on step three actually happening. **If the Updater evicts but the Admission Controller never applies the recommendation, there is no exit condition** — the replacement is born "wrong" again, the Updater re-flags it, and you have a loop.

## How the loop ignites without anyone touching anything

The trigger is almost always **chronic over-provisioning plus the passage of time**, not an event.

Imagine a pod whose request is `500m` CPU but which only ever uses `~40m`. The VPA Recommender, fed by live usage, settles on a target around `35m`. The pod's request is now ~14× larger than the recommendation. As the Recommender accumulates data, its confidence bounds narrow; at some point the pod's `500m` request falls outside the acceptable band and crosses the Updater's **eviction threshold**.

The Updater fires. **No deploy, no config change, no load change is required** — this happens on the autoscaler's own schedule. That's why these incidents feel like "the autoscaler suddenly started behaving differently": from the operator's chair nothing changed, but internally a slow-moving recommendation finally crossed a line.

> **Load-bearing fact:** A VPA eviction is *not* evidence that a workload got bigger. The most common real trigger is the opposite — a pod that has been *over*-provisioned for a long time, where the recommendation has drifted far *below* the request and the Updater finally decides to shrink it. Check usage-vs-request before assuming a spike.

## Why it never converges: the resize never lands

In a healthy single-VPA setup the loop above self-terminates after one eviction. When it *doesn't*, there is exactly one question worth asking:

> Did the replacement pod actually come up at the recommended request?

The fastest way to answer it is to chart the pod's **CPU/memory request over time** (not usage — the request). In a runaway loop you'll see the request **pinned dead-flat at the original template value** for the entire incident, never stepping toward the recommendation. That flat line is the whole diagnosis: the Updater is evicting, but the Admission Controller is **not** applying the smaller value, so every new pod is oversized on arrival and immediately re-eligible for eviction.

So the real question becomes: *why didn't admission apply the recommendation?*

## The usual culprit: more than one VPA on the same Deployment

VPA's contract is **one VPA object per pod**. Violate it and behavior becomes undefined — and this is surprisingly easy to do by accident. You can end up with several VPA objects all selecting the same Deployment when, for example:

- a workload has its own VPA, **and**
- a higher-level autoscaling operator (the kind that wraps HPA/VPA and manages resources for you) creates *its own* VPA objects for the same workload — often a "monitor" VPA (recommendation-only) and an "updater" VPA (the one it applies through).

Now three VPAs target one Deployment. To understand what happens at pod creation, you have to know exactly how the Admission Controller picks a winner when several match. From the upstream VPA admission code:

**1. `Off`-mode VPAs are skipped entirely.**

```go
if vpa_api_util.GetUpdateMode(vpaConfig) == vpa_types.UpdateModeOff && vpaConfig.Spec.StartupBoost == nil {
    continue
}
```

A recommendation-only ("monitor") VPA in `Off` mode never participates in admission. It's a pure data source — innocent.

**2. Among the remaining (non-`Off`) VPAs, the *oldest* one wins.** Selection is by earliest `creationTimestamp`, with name as a tiebreak. **`updateMode` is not part of the comparison** — an `Initial`-mode VPA is just as eligible to "win" admission as an `Auto`-mode one.

```go
// "Stronger" VPA = earlier creation timestamp (then name)
if !aTime.Equal(&bTime) { return aTime.Before(&bTime) }
```

Put those two rules together and you get the trap:

> **The eviction loop's root cause:** The *newer* `Auto` VPA does the evicting (its Updater), but an *older* VPA — for instance an operator-managed "updater" VPA sitting in `Initial` mode with **no recommendation populated** — wins the admission selection because it was created first. It applies *nothing*, so the new pod keeps the template's `500m`. The `Auto` VPA's `35m` never lands. The Updater evicts again. Forever.

The operator can even be "switched off" at the top level and still be the cause: if it's idle it won't *write* a recommendation into its updater VPA (leaving it empty), but the stray, empty VPA object is still attached to the Deployment and still wins admission by age. A dormant controller's leftover objects are enough to jam convergence.

## The amplifier: single replica + long-running work

The eviction loop is bad on any workload, but two properties turn it from "annoying churn" into an outage:

- **Single replica** (`minReplicas == 1`). There is no second pod to absorb the disruption — every eviction is a full outage of the workload. (Vanilla VPA Updater refuses to evict the last pod by default via `--min-replicas`, but operators and custom configs frequently relax this for `replicas: 1` workloads, re-enabling exactly the dangerous behavior.)
- **Work that outlasts the eviction interval.** If a unit of work takes longer than the ~2-minute eviction cadence, it can *never complete*. For a queue consumer this is catastrophic: every eviction `SIGTERM`s mid-processing, the message is Nack'd and redelivered, the next pod starts over and is killed again — a redelivery loop layered on top of the eviction loop.

This is why long-running batch/queue workers are the classic victims. They're the workloads least able to tolerate eviction and most likely to be over-provisioned (sized for peak file size, idle most of the time).

## Diagnosing it quickly

A tight runbook when you see periodic, unexplained pod churn:

1. **Read the eviction reason.** `kubectl get events` (or your events backend) — `EvictedByVPA` / *"evicted to apply resource recommendation"* with a `vpa-updater` reporting controller points straight here. A flat ~2-minute cadence with no deploys is the fingerprint.
2. **Chart request vs. usage vs. recommendation.** Over-provisioned (usage << request) plus a request line that never moves toward the recommendation = a non-converging resize. This single chart usually closes the case.
3. **List every VPA targeting the Deployment**, with mode and age:
   ```
   kubectl get vpa -o custom-columns=\
   'NAME:.metadata.name,KIND:.spec.targetRef.kind,TARGET:.spec.targetRef.name,MODE:.spec.updatePolicy.updateMode'
   ```
   More than one VPA pointing at the same Deployment is the smoking gun. Note their creation timestamps — the *oldest* non-`Off` one is your admission winner.
4. **Confirm which VPA holds a recommendation.** If the admission-winning VPA has an empty recommendation while a different VPA is the one evicting, you've found the split-brain.

## Fixing and preventing it

- **Stop the evictions now:** set the evicting VPA to `updateMode: Off` (or pin static requests for the workload). Removing the *only* component that evicts breaks the loop immediately — you don't need to untangle admission to stop the bleeding.
- **One VPA per Deployment.** Delete or disable the duplicates. If a higher-level operator manages the workload, let it own the VPA *exclusively*; don't also hand-roll a second VPA on the same target.
- **Don't run VPA `Auto` on single-replica, long-running workers.** Eviction interrupts in-flight work and there's no replica to cover for it. Prefer `Off`/`Initial` (right-size at creation only) or static requests for queue/batch consumers, especially when a unit of work can exceed the eviction interval.
- **Add a guardrail.** A policy/admission check that refuses VPA `updateMode: Auto`/`Recreate` on `minReplicas: 1` workloads (or that flags multiple VPAs on one Deployment) prevents the whole class.
- **Right-size deliberately, not via eviction.** If the goal is to reclaim the `500m → 35m` over-provisioning, do it as a one-time static request change, not by letting an `Auto` VPA evict a workload that can't tolerate it.

## Key takeaways

- **VPA `Auto` = Updater (evicts) + Admission Controller (resizes).** It only converges if *both* act. An Updater that evicts while admission fails to apply the new size loops forever.
- **An eviction often means a pod is too *big*, not too small.** Chronic over-provisioning + a slowly-narrowing recommendation crosses the eviction threshold with no human action — which is why it feels like "nothing changed."
- **The flat request line is the diagnosis.** If the pod's request never moves toward the recommendation across the incident, admission isn't applying it.
- **One VPA per Deployment, period.** When several match, admission **skips `Off`-mode VPAs** and otherwise picks the **oldest by creation timestamp** — *not* by mode. An old, empty `Initial` VPA can win admission and silently neutralize a newer `Auto` VPA's recommendation.
- **Single replica + work longer than the eviction interval = outage,** and for queue workers, a Nack/redelivery loop on top. Keep `Auto` eviction away from these.

## Related notes

- [[notes/K8s/kubernetes-autoscaling|Kubernetes Autoscaling]] — VPA's Recommender/Updater/Admission Controller, the four update modes, and the HPA/VPA feedback-loop hazard
- [[notes/K8s/kubebuilder-controllers-and-webhooks|Kubebuilder Controllers & Admission Webhooks]] — how mutating admission webhooks (like VPA's) intercept and rewrite pods
