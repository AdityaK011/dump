---
title: "Summary: Bound Is Not Attached"
---

> **Full notes:** [[notes/K8s/waitforfirstconsumer-bound-vs-attached|Bound Is Not Attached: What WaitForFirstConsumer Actually Promises -->]]

## Key Concepts

### The symptom
- Stateful pod scheduled onto an **older machine family** node; its disk is a **type only newer families can attach**; PVC reports `Bound`.
- Two wrong assumptions collide: that `WaitForFirstConsumer` prevents this, and that `Bound` implies a node accepted the disk.

### Four stages, four owners
| Stage | Owner | Failure mode |
|---|---|---|
| Provision | `external-provisioner` → `CreateVolume` | quota, unsupported type |
| **Bind** | PV controller | no matching PV / topology |
| **Attach** | attach-detach ctrl → `external-attacher` → `ControllerPublishVolume` | **machine-family incompat**, attach limits |
| Mount | kubelet → `NodeStage`/`NodePublish` | fsType, perms |
- `Bound` = end of stage 2. The bug lives in stage 3. **No PVC field can report on stage 3** — a PVC has no concept of a node.
- A PV is bound to a **PVC**, never to a node. Its only node-shaped field is `spec.nodeAffinity`.

### What WFFC actually promises
- `Immediate` — disk created before any pod exists → random zone → pod permanently confined there.
- `WFFC` — scheduler picks node → writes `volume.kubernetes.io/selected-node` on PVC → provisioner creates disk in **that node's zone** → PV created → `Bound`.
- Three ways the name misleads:
  - **"First" is literal** — once, ever. Not "current pod," not "suitable pod."
  - **Permanent** — binding is immutable; WFFC has zero influence on every pod after the first.
  - **Optimizes zone, not compatibility** — it exists only to solve "disk in the wrong zone."
- StatefulSet: PVC deliberately outlives every pod. The "first consumer" may be a pod deleted weeks ago during a migration.
- ⇒ WFFC is a *provisioning hint*, not admission control. It cannot express "stay unbound until a compatible node exists."

### Why the scheduler said yes
- PV carries `volumeAttributes.type` (decides compatibility) and `nodeAffinity` (what's enforced). **They are unconnected.**
- **Storage topology constrains zones, not machine types.** Scheduler filters on zone affinity + per-node CSI attach limits (`CSINode…allocatable.count`). Machine family is in no scheduling predicate.
- ⇒ same-zone wrong-family node passes `Filter`; failure defers to attach → `FailedAttachVolume` while PVC stays `Bound`.
- Newer CSI drivers publish **disk-type topology labels** → mismatch becomes a *scheduling* failure (`Pending`, "volume node affinity conflict") instead of an attach loop. Better, but not the fix.

### Case A vs Case B (establish this first)
- **Case A — real incompat:** pod `ContainerCreating`, `AttachVolume.Attach failed`. Will never start there.
- **Case B — no problem:** pod `Running` on the old-family node ⇒ attach **succeeded** ⇒ disk type *is* compatible (probably still a legacy SSD/balanced disk). Only the **zone** was pinned early. Nothing to fix.
- Conflating them is the actual trap: `Bound` looks like a decision about a node.

### How the mismatch is created
- Machine-family migration that also changes disk type, run side-by-side:
  - disk type fixed at `CreateVolume` → **irreversible**
  - pod's family re-decided on **every** scheduling event, constrained only by zone
- Sticky storage decision + fluid compute decision ⇒ any eviction / node upgrade / spot reclaim drops a new-type disk onto a leftover old-family node.
- `volumeClaimTemplates` is **immutable** ⇒ can't change `storageClassName` in place ⇒ teams add a parallel StatefulSet/nodeSet ⇒ two disk generations + two families in one cluster.

### The fix belongs on the pod
- `allowedTopologies` speaks **only zone-shaped keys** — no vocabulary for machine family.
- Put `nodeAffinity` on machine family in the pod template + **taint/drain the outgoing pool**.
- Ordering: **constrain compute first, then switch the StorageClass.**
- Already mis-bound? No in-place repair: (1) cordon + delete PVC, let replicas rebuild; (2) `VolumeSnapshot` → new PVC on the correct class; (3) rebuild the set.

## Quick Reference

```text
WFFC          : defer provisioning until FIRST scheduling decision. once. permanent.
Bound         : PVC <-> PV pairing exists. says NOTHING about any node.
Attached      : disk hot-plugged to a specific VM. where family compat is enforced.
PV nodeAffinity: zone only. machine family NOT expressible, NOT scheduled on.
allowedTopologies: zone-shaped keys only. cannot gate machine family.
volumeClaimTemplates: immutable -> can't change storageClassName in place.

Scheduler enforces : PV zone affinity + CSINode attach limits
Scheduler ignores  : disk type vs machine family   <-- the whole bug
```

**Diagnose**
```bash
kubectl get pod <pod> -o wide                          # which family did it land on?
kubectl describe pod <pod> | sed -n '/Events/,$p'      # FailedAttachVolume?
kubectl get volumeattachment | grep <pv>               # attached=true => Case B, fine
kubectl get pv <pv> -o jsonpath='{.spec.csi.volumeAttributes.type}'   # real disk type
kubectl get pv <pv> -o jsonpath='{.spec.nodeAffinity}' | jq           # what's enforced
kubectl get pvc <pvc> -o jsonpath='{.metadata.annotations}' | jq      # selected-node
```
`selected-node` is the archaeology — it names the node that fixed the disk's fate, even if long deleted.

**Rules of thumb**
- Never read `Bound` as "a node accepted this." Check for a `VolumeAttachment` instead.
- Any constraint about *where a pod may run* belongs on the pod, never on a StorageClass.
- During a family migration: pod-level family affinity **before** the new StorageClass.
- Prefer deleting the old node pool over trusting affinity alone — absent capacity is the strongest guardrail.

**Anti-patterns**
- Expecting WFFC to keep a PVC `Pending` until a *compatible* node exists.
- Trying to encode machine-family requirements via `allowedTopologies`.
- Diagnosing a `Running` pod as broken because its disk "belongs to" another family (Case B).
- Switching StorageClass mid-migration with no pod-level family constraint in place.
