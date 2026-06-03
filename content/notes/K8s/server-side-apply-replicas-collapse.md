---
title: "When Server-Side Apply Deletes Your Replicas: The before-first-apply Trap"
---

## The symptom that shouldn't be possible

A stable web service is running ~20 pods behind an HPA. A routine deploy goes out — same kind of image bump that's shipped a hundred times before. For a window of about one to two minutes, **the available pod count drops to zero**. Requests in that window are lost. Then it recovers on its own and settles back to ~20-something pods, perfectly healthy, as if nothing happened.

The first instinct is to blame the obvious things, and every one of them is a dead end:

- **"It OOMed."** Pods *did* get OOMKilled during the window — but that's a *symptom*. Steady-state memory is a fraction of the limit; the kills only happened during the incident.
- **"The PodDisruptionBudget failed."** The service has a PDB with `minAvailable: 66%`. It was configured correctly and was never violated in the way people assume — it simply doesn't apply to this failure mode.
- **"The rollout was misconfigured."** The Deployment uses `maxSurge: 30%, maxUnavailable: 0` — the *safe* setting that guarantees no pod is removed before a replacement is Ready. It was set correctly. It didn't help either.

The real cause is none of these. It's a **field-ownership migration inside Kubernetes' Server-Side Apply machinery**, triggered the first time a new CD tool applied the Deployment. The fleet didn't get *rolled* down to zero — its `spec.replicas` got silently **deleted**, and a deleted `replicas` defaults to `1`. Everything else is downstream of that one fact.

This note is about how that happens, why it's invisible until you read the audit log's `managedFields`, and why it only fires on the *second* apply, not the first.

## Two different ways to "own" a field

Kubernetes has two apply models, and the bug lives in the seam between them.

**Client-side apply** (`kubectl apply`, the classic) computes a three-way merge *on the client*. It diffs your manifest against a snapshot stored in the `kubectl.kubernetes.io/last-applied-configuration` annotation, and against the live object, then PATCHes the difference. The server doesn't track *who* owns *what* — it just sees writes. Crucially: **a field you omit is left alone** as long as it wasn't in your previous last-applied snapshot. Many CD tools (and the older generation of GitOps tooling) work this way.

**Server-Side Apply (SSA)** moves the merge *into the API server*. Every field of the object is tagged in `metadata.managedFields` with the **manager** that owns it and the **operation** that claimed it. This is the model `kubectl apply --server-side` uses. Its defining rule is the one that bites here:

> On an apply, for every field a manager **owns but did not include** this time, that field is **removed** from the object.

That's a feature — it's how SSA lets you delete a field by deleting it from your manifest. It's also a loaded gun if you ever end up *owning a field you never meant to manage*.

### Apply vs Update operations

`managedFields` entries carry an `operation` of either `Apply` or `Update`:

- **`Apply`** — ownership claimed declaratively via SSA. Subject to the "owned-but-omitted ⇒ delete" rule.
- **`Update`** — ownership from an imperative write: a `PUT`, a non-apply `PATCH` (JSON/merge/strategic-merge), *and* legacy client-side applies. Update owners only own what they actually wrote, and **omission never deletes**.

A field can be co-owned by an `Apply` manager and one or more `Update` managers. Normally these coexist peacefully; an apply that collides with an `Update` owner raises a *conflict* (which you resolve, or override with `--force-conflicts`) rather than silently evicting it.

Hold onto that last point — "Apply and Update normally coexist" — because the whole incident is about the **one exception** to it.

## The HPA setup that makes `replicas` a landmine

For any Deployment scaled by an HPA, the golden rule is: **do not put `replicas` in your manifest.** The HPA owns the replica count at runtime; if your manifest also declared it, every apply would fight the autoscaler and snap the count back to a static number.

So a correct HPA-managed Deployment manifest looks like this — note the conspicuous absence of `replicas`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  # no replicas: — intentionally. The HPA manages it.
  revisionHistoryLimit: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 30%
      maxUnavailable: 0
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: web
          image: registry.example/web:v2
```

With **client-side** apply this is bulletproof: `replicas` is never in the manifest and never in the last-applied snapshot, so it's left untouched on every deploy, forever. The HPA's value survives.

With **server-side** apply, "omit `replicas`" is only safe *as long as the applier doesn't own `replicas`*. And that's exactly the invariant the migration quietly breaks.

## The migration shim: `before-first-apply`

Here's the piece almost nobody knows about. When you run the **first ever** server-side apply against an object that was previously managed **client-side**, the API server has a problem: the live object is full of fields (set by the old client-side tool) that have no SSA owner. If the first apply just took over and the rest were unowned, anything the new manifest didn't mention could get wiped.

To prevent that, the API server **synthesizes a manager named `before-first-apply`** with `operation: Update`, and assigns it ownership of all the pre-existing fields (reconstructed largely from the `last-applied-configuration` annotation). It's a *transitional shim* whose entire purpose is to make the first apply safe and then be **absorbed by the applying manager**. It is not meant to persist like a real controller's Update entry.

This is the subtlety that answers "why did an `Update` owner get removed by an `Apply`?": **`before-first-apply` is special-cased to be consumed by the first apply.** Normal Update owners coexist; this one is designed to be dissolved.

## The five operations, step by step

A modern server-side CD tool rarely does one clean `kubectl apply`. A real deploy step here fired **five API calls against the Deployment in ~350 ms**, all from the same service account:

1. a full `apply --server-side` (with `--force-conflicts`, required to adopt a client-side-managed object),
2. three metadata-only merge patches (annotations/labels housekeeping),
3. a second full `apply --server-side`.

Watching `spec.replicas` and `managedFields` across those five calls (reconstructed from the API server audit log) tells the whole story. Take a Deployment sitting at `replicas: 20`, HPA-managed, previously client-side-managed.

### Op 1 — first server-side apply: `replicas` stays at 20 ✅

The applier sends its manifest, which **omits `replicas`** and declares the fields it actually manages: `selector`, `strategy`, `template`, `revisionHistoryLimit`. Because the object was client-side-managed, the server spins up the `before-first-apply` shim. Snapshot right after:

```
manager=web-cd          op=Apply    f:spec -> [revisionHistoryLimit, selector, strategy, template]
manager=before-first-apply op=Update f:spec -> [progressDeadlineSeconds, replicas, revisionHistoryLimit, selector, strategy, template]
```

`spec.replicas` is owned by `before-first-apply`, **not** by the CD manager. The CD tool didn't send `replicas`, so it can't and doesn't own it. Result: `replicas` is untouched → still **20**. Generation bumps once (the image/template changed). All good — *this is why the first apply is harmless.*

### Ops 2–4 — metadata merge patches: no spec change, but the shim dissolves

These three are **merge patches**, not applies (the audit log shows no full object body, and `metadata.generation` does **not** move — proof they changed no spec field). A merge patch only touches the keys physically present in its body, and **absence in a merge patch never deletes anything**. So these *cannot* affect `replicas` directly, regardless of ownership.

But underneath, something important completes. Any write triggers the API server to recompute `managedFields`, and that's when the `before-first-apply → web-cd` absorption finishes. By the time we can next observe ownership:

```
manager=web-cd          op=Apply    f:spec -> [progressDeadlineSeconds, replicas, revisionHistoryLimit, selector, strategy, template]
# before-first-apply is GONE
```

The **entire** `before-first-apply` field set — including `replicas` and `progressDeadlineSeconds`, the two fields the CD manifest **never declared** — has been folded onto the CD tool's `Apply` entry. Generation is unchanged; `replicas` is still **20**. Nothing looks broken.

This is the landmine being armed. The CD tool is now the `Apply`-owner of `replicas`, a field it doesn't manage and will never send.

> **Why the whole set transferred, not just the declared fields.** The `--force-conflicts` apply seizes the fields the manifest *does* declare (they conflict with the shim's copies). The shim is then left holding only the residual fields the manifest doesn't declare (`replicas`, `progressDeadlineSeconds`) — and because `before-first-apply` is a one-shot migration entity designed to be consumed, the upgrade path dissolves it entirely and consolidates *its remaining fields* onto the applying manager rather than leaving an orphaned shim behind.

### Op 5 — second server-side apply: `replicas` deleted → defaults to 1 💥

The CD tool applies **the same manifest again** — still omitting `replicas`. But now it *owns* `replicas`. The SSA rule fires without mercy:

> field owned but not included this time ⇒ remove it.

`replicas` is removed from the object. A `Deployment` with no `replicas` **defaults to `1`** (the schema default). Snapshot:

```
manager=web-cd          op=Apply    f:spec -> [revisionHistoryLimit, selector, strategy, template]
# replicas owner: NOBODY ;  spec.replicas: 1
```

Generation bumps again. The CD manager's ownership shrinks back to exactly the four fields it declares. `progressDeadlineSeconds` got silently reset to its default of `600` by the identical mechanism — harmless, but a tell that this is a *general* field-deletion, not something replicas-specific.

The two visible symptoms — replicas collapsing **and** any residual legacy labels on the pod template disappearing — are the *same* mechanism: both were fields the CD tool inherited via the shim migration but never declares, so the second apply deleted both. One of them happened to be load-bearing.

### Summary of the trap

| field | in CD manifest? | owner after Op 1 | owner by Op 4 | Op 5 (omitted) |
|---|---|---|---|---|
| `spec.replicas` | no (HPA-managed) | `before-first-apply` | **CD tool** | **deleted → default 1** |
| `progressDeadlineSeconds` | no | `before-first-apply` | **CD tool** | deleted → default 600 |
| `selector/strategy/template` | yes | CD tool (+shim) | CD tool | kept |

The first apply is safe because the CD tool doesn't own `replicas` yet. The second apply is fatal because, between them, the `before-first-apply` migration handed it ownership of `replicas` — *a field it never declares.*

## Why the fleet then cratered — and why PDB/maxUnavailable didn't save it

The five API calls only changed *desired state*. The cluster then reconciled to it, and that reconciliation is the outage:

1. **Collapse.** `spec.replicas` is now `1`. The Deployment controller wants exactly one pod. With `maxSurge: 30%` of 1 (= ceil → 1), the max is two pods total. It tears the fleet down from 20 toward 1, deleting ~19 pods.
2. **No protection from the usual guards — and this is the important, counter-intuitive part:**
   - **`maxUnavailable: 0`** governs a *rolling template update*. It bounds how many pods the controller removes *while rolling out a new pod template at the current replica count*. It does **not** constrain a reduction of `spec.replicas` itself — that's not a rollout step, it's a scale-down. Lowering `replicas` from 20 to 1 is *allowed* to delete 19 pods; there's nothing "unavailable" about pods the spec no longer asks for.
   - **PodDisruptionBudget** only governs **voluntary disruptions** routed through the **Eviction API** (`kubectl drain`, the descheduler, node upgrades). Scaling a Deployment down deletes pods **directly via the ReplicaSet controller**, which never calls the Eviction API — so the PDB is *never consulted*. `minAvailable: 66%` is structurally incapable of stopping a `replicas` reduction. (It also can't stop OOMKills or readiness failures — both involuntary.)

   This is the single most useful takeaway: **a PDB protects against eviction, not against someone changing your replica count.** Anything that writes `spec.replicas` — a bad apply, a fat-fingered `kubectl scale`, a CD tool that inherited ownership — sails straight past it.
3. **The brief total outage.** With ~1 pod up against full production traffic, the lone pod (and the fresh ones starting under load) get overwhelmed and OOMKilled before they can stabilize — so for ~1–2 minutes there are effectively **zero Ready pods**. *This* is the OOM everyone chases first; it's a consequence of the collapse, not the cause.
4. **Recovery.** The HPA observes the deficit (its `minReplicas` floor plus live load) and scales `spec.replicas` back up via the `/scale` subresource. That write re-establishes `kube-controller-manager` as the owner of `replicas`. The fleet climbs back to ~20+ and holds. From the outside it "fixed itself."

## How to actually diagnose this

The deployment-level events (`Scaled down replica set ... from 20 to 2`) look like a berserk rollout and send people down the rolling-update rabbit hole. The truth is only visible one layer down:

- **Read `metadata.managedFields`** on the live Deployment: `kubectl get deploy web --show-managed-fields -o yaml`. Look at who owns `f:replicas`. If your CD tool's `Apply` entry owns it, you have the bug latent right now.
- **Read the API server audit log** for `patch`/`update`/`apply` on the Deployment, filtered by `protoPayload.resourceName` and the deployment name. The principal (`authenticationInfo.principalEmail`) tells you *which* controller wrote `replicas` and when. This is what distinguishes "the CD tool did it" from "the autoscaler did it" from "a human did it" — and it's how you rule out the tempting-but-wrong theory that two CD systems raced. (In the real incident, the audit log showed a single CD service account doing all five writes; the "other" CD tool people suspected had merely left *residual labels* on the object and never actually wrote to it.)
- **Watch `metadata.generation`** to separate spec changes from metadata churn: it only increments on spec writes. It's the cheapest way to prove which of N rapid patches actually touched the spec.

## The fix

The bug is in the *apply behavior*, not the manifest — the manifest is correct to omit `replicas`. Options, roughly in order of correctness:

1. **Make the applier never own `replicas`.** The applier should explicitly *disclaim* the field rather than inheriting it through the migration. The clean fix at the tooling layer is to keep `replicas` out of the apply set *and* avoid the force-migration that consolidates the shim's residual fields — e.g. by pre-seeding SSA ownership so the HPA/autoscaler manager owns `replicas` before the first CD apply, so the shim never hands it to the CD tool.
2. **Preserve the live replica count on each apply.** Some CD tooling has an explicit "don't touch replicas" / "retain existing replicas for autoscaled workloads" mode. This is what well-behaved CD systems do, and it's why the *previous* client-side tool never hit this — client-side apply has no field-ownership concept to migrate, so an omitted `replicas` is simply always left alone.
3. **As a last resort, pin `replicas` in the manifest.** This makes the two applies consistent (no owned-but-omitted field), but it *defeats the HPA* and is wrong for an autoscaled workload. Avoid unless the workload isn't actually autoscaled.

And independently, give the workload enough memory headroom (split `requests` from `limits`) so that *if* the replica count ever dips, the survivors degrade instead of OOM-cascading into a hard outage. That doesn't fix the root cause, but it converts "total outage" into "brief brownout."

## Key takeaways

- **SSA's defining rule is "owned-but-omitted ⇒ delete."** Owning a field you don't intend to manage is dangerous, and migrations can hand you ownership you never asked for.
- **`before-first-apply` is a transitional shim, not a normal owner.** It exists to make the first client-side→server-side apply safe, and is *designed* to be absorbed by the applying manager — dragging along *any* legacy field the new manifest doesn't declare, including `replicas`.
- **That's why the second apply breaks, not the first.** Apply 1 doesn't own `replicas` (safe). The migration completes. Apply 2 owns `replicas` but omits it → deletes it → defaults to `1`.
- **Merge patches can't delete by omission; only applies can.** Generation bumps tell you which writes touched the spec at all.
- **A PDB does not protect your replica count.** It guards voluntary, Eviction-API disruptions only. `maxUnavailable: 0` guards rollouts, not scale-downs. Neither stops a `spec.replicas` write.
- **Read `managedFields` and the audit log.** Deployment-level events describe the *symptom* (frantic scaling); ownership metadata and the audit principal reveal the *cause* (who deleted/wrote `replicas`, and when).

For the autoscaler side of ownership and how the HPA writes `replicas` through the `/scale` subresource, see [[notes/K8s/kubernetes-autoscaling|Kubernetes Autoscaling]]. For another "the autoscaler did something nobody triggered" failure that's really a controller-ownership problem, see [[notes/K8s/vpa-eviction-loops|The VPA Eviction Loop]].
