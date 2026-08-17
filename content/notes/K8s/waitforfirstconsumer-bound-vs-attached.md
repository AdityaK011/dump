---
title: "Bound Is Not Attached: What WaitForFirstConsumer Actually Promises"
---

## The symptom that shouldn't be possible

A stateful workload — say a data node of a managed search cluster — comes up on a node from an **older machine family**. Its persistent disk, however, was provisioned as a **disk type only newer machine families can attach**. And the `PersistentVolumeClaim` reports:

```
NAME              STATUS   VOLUME      CAPACITY   STORAGECLASS
data-es-data-0    Bound    pvc-8f3c…   1Ti        premium-rwo
```

Two things feel wrong at once:

1. Why did the scheduler put the pod on a node that can't use its disk?
2. The StorageClass says `volumeBindingMode: WaitForFirstConsumer`. Isn't the entire point of that setting to stop exactly this? Shouldn't the PVC have stayed `Pending`?

Both questions have the same root: **`WaitForFirstConsumer` promises far less than its name suggests, and `Bound` means far less than people assume.** This note pulls the two apart.

## Four stages, four owners

The most useful mental model is that a PVC passes through four distinct stages, each driven by a *different* component, each with its own failure mode:

| Stage | Who does it | What it means | Where it fails |
|---|---|---|---|
| **Provision** | `external-provisioner` sidecar → CSI `CreateVolume` | A real disk is created in the cloud | quota, unsupported type, bad params |
| **Bind** | PV controller (kube-controller-manager) | PVC ↔ PV paired 1:1, permanently | no matching PV, topology mismatch |
| **Attach** | attach/detach controller → `external-attacher` → CSI `ControllerPublishVolume` | Disk hot-plugged onto a **specific VM** | **machine-type incompatibility**, per-VM attach limits |
| **Mount** | kubelet → CSI node plugin `NodeStage`/`NodePublish` | Filesystem staged and bind-mounted into the pod | fs corruption, wrong fsType, permissions |

`Bound` is the end of stage 2. The incompatibility in our story lives in stage 3. There is nothing in the PVC's `status.phase` that can ever tell you about stage 3, because `PersistentVolumeClaim` has no concept of a node.

> **Load-bearing fact:** a PV is never "bound to a node." It is bound to a **PVC**. The only node-shaped thing it carries is `spec.nodeAffinity`, and as we'll see, that affinity is zone-shaped, not machine-shaped.

## What WaitForFirstConsumer actually does

There are two binding modes, and the difference is *when* stages 1 and 2 happen relative to scheduling.

**`Immediate`** — the provisioner creates a disk the moment the PVC appears, before any pod exists. In a multi-zone cluster this is a coin flip: the disk lands in some zone, and from then on the pod is permanently confined to that zone. Classic cause of "my pod is Pending forever and I don't know why."

**`WaitForFirstConsumer` (WFFC)** — inverts the order. Provisioning is deferred until the scheduler has picked a node, so the disk gets created in the *right* zone:

```
PVC created                      -> Pending, no disk exists anywhere
Pod created referencing it       -> scheduler's VolumeBinding plugin runs
Scheduler picks node N           -> writes annotation on the PVC:
                                      volume.kubernetes.io/selected-node: N
external-provisioner sees it     -> CreateVolume in N's topology (its zone)
PV object created                -> PV controller binds PVC <-> PV
                                 -> PVC: Bound
```

Read that carefully, because the name misleads on three separate counts:

- **"First"** is literal. It waits for the *first* pod to reach the scheduling decision, once, ever. Not "the current pod." Not "a suitable pod."
- **The outcome is permanent.** Once bound, the PVC-to-PV pairing is immutable. WFFC has no further effect on that PVC for the rest of its life. It is not a re-evaluated constraint; it is a one-time deferral.
- **What it optimizes is zone placement**, not compatibility. It exists to solve "the disk is in the wrong zone," and that is *all* it solves.

For a StatefulSet this matters enormously: the PVC deliberately **outlives every pod incarnation**. That's the point of `volumeClaimTemplates`. So the "first consumer" that fixed your disk's fate may have been a pod that was deleted weeks ago, during a node-pool migration nobody remembers. Every restart, eviction, node upgrade, and rollout since then has had to live with that decision.

> WFFC is a *provisioning hint*, not an admission control. It cannot express "stay unbound until a compatible node exists," because after the very first bind it has no voice at all.

## Why the scheduler said yes

Once a PV exists, the scheduler's `VolumeBinding` plugin enforces the PV's own constraints during `Filter`. Here's what a provisioned block PV actually looks like:

```yaml
apiVersion: v1
kind: PersistentVolume
spec:
  csi:
    driver: pd.csi.storage.gke.io
    volumeHandle: projects/…/zones/us-central1-a/disks/pvc-8f3c…
    volumeAttributes:
      type: hyperdisk-balanced          # <- the compatibility-relevant bit
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["us-central1-a"]     # <- the ONLY enforced constraint
```

The `type` is the field that decides which machine families can attach the disk. The `nodeAffinity` is the field the scheduler enforces. **They are not connected.** Nothing in the PV says "newer families only," so any node in `us-central1-a` satisfies the affinity — including an older-family node that will refuse the disk at attach time.

That's the whole asymmetry, and it's worth stating as a rule:

> **Storage topology constrains zones, not machine types.** The scheduler filters on zone affinity and per-node CSI attach limits (`CSINode.spec.drivers[].allocatable.count`). Machine-family compatibility is not part of any scheduling predicate.

So the failure gets deferred: the pod schedules successfully, the attach/detach controller creates a `VolumeAttachment`, `ControllerPublishVolume` calls the cloud API, and the cloud returns an invalid-argument error. The pod sits in `ContainerCreating` with `FailedAttachVolume` while the PVC stays serenely `Bound` forever.

### The shift-left that partially fixes this

Newer CSI driver versions publish additional topology keys — labels advertising which *disk types* a node supports — and the driver then stamps them into the PV's `nodeAffinity`. When that's in play, the mismatch becomes a **scheduling** failure (`Pending`, "node(s) had volume node affinity conflict") instead of an **attach** failure. That's strictly better: `Pending` is honest, whereas `ContainerCreating` + a retrying attach loop looks like a transient blip. Check your driver version, but don't treat it as the fix — the real fix belongs on the pod.

## Bound ≠ Attached: separate the two worlds

Before reaching for remediation, establish which situation you're actually in, because one of them is not a problem at all.

**Case A — the incompatibility is real.** Pod is `ContainerCreating`; events show `FailedAttachVolume` / `AttachVolume.Attach failed` with an invalid-argument style message. The disk type genuinely cannot attach to that machine family. The pod will never start there, no matter how long you wait.

**Case B — nothing is wrong.** The pod is `Running` on the older-family node. Then the attach **succeeded**, which proves the disk type is attachable from that family — most likely it's still a legacy SSD/balanced disk rather than the newer type you assumed. What got locked in by that long-ago first consumer was the **zone**, not the family. There is no bug and nothing to repair; the disk merely happens to have been created while a different node was the scheduling target.

Conflating these is the actual trap. `Bound` looks like evidence of a decision about a node, so it invites Case-B situations to be misdiagnosed as Case-A ones.

## How the mismatch gets created

The reliable recipe is a **machine-family migration that also changes disk type** — old family with old disks, new family with new disks, run side by side during the cutover. The storage half and the compute half of that migration commit at different times and by different mechanisms:

- The disk's type is fixed at `CreateVolume`, i.e. the first time any consumer schedules. Irreversible.
- The pod's machine family is re-decided on *every* scheduling event, and by default is constrained only by zone.

So the storage decision is sticky and the compute decision is fluid. Any eviction, node upgrade, spot reclamation, or scale-down during the migration window can drop a pod with a new-type disk onto a leftover old-family node. There's no guardrail unless you add one.

The other common route: `volumeClaimTemplates` in a StatefulSet is **immutable**, so you cannot change `storageClassName` on an existing set. Teams work around it by creating a *new* nodeSet/StatefulSet with the new class while the old one still exists — and end up with two disk generations and two machine families in one cluster, sharing a scheduler that doesn't know they're incompatible.

## The fix belongs on the pod, not the storage

You cannot express machine-family requirements through storage objects. `allowedTopologies` on a StorageClass only speaks zone-shaped topology keys; it has no vocabulary for machine family. The constraint has to live where the scheduling decision is made:

```yaml
# in the pod template of the StatefulSet / operator nodeSet
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: cloud.google.com/machine-family
          operator: In
          values: ["c4a"]
```

With that in place, WFFC finally does what you wanted all along: the *only* nodes eligible to be the first consumer are compatible ones, so the disk is provisioned correctly **and** every subsequent reschedule stays on a compatible node. Complement it with either of:

- **Taint the old node pool** (`NoSchedule`) so nothing can drift back, then cordon and drain it.
- **Delete the old pool** once migration completes. The most durable guardrail is the absence of incompatible capacity.

Note the ordering discipline this implies: **add the pod-level family constraint before switching the StorageClass**, not after. Otherwise there's a window where new-type disks can be born attached to the wrong family.

### Repairing a PVC that's already mis-bound

There is no in-place fix — the binding is immutable and the disk type can't be swapped underneath it. Options, cheapest first:

1. **Let the data rebuild.** Cordon the incompatible nodes, delete the PVC and its pod, let the controller re-provision. For a replicated store this is routine: the new node re-syncs from replicas. Verify replication first; do it one node at a time.
2. **Snapshot and restore.** `VolumeSnapshot` the existing disk, create a new PVC from it with the correct class. Preserves data at the cost of a longer procedure.
3. **Reduce scope.** Sometimes the honest answer is to stop using the mixed pool and rebuild the whole set on the new family.

## Diagnostics

Six commands settle it:

```bash
# 1. Which node, and did attach actually succeed?
kubectl get pod <pod> -o wide
kubectl describe pod <pod> | sed -n '/Events/,$p'

# 2. Is there a live attachment? (Case A vs Case B)
kubectl get volumeattachment | grep <pv-name>
#   attached=true  -> Case B, the disk works on this family

# 3. What disk type is it actually?
kubectl get pv <pv> -o jsonpath='{.spec.csi.volumeAttributes.type}{"\n"}'

# 4. What does the scheduler actually enforce?
kubectl get pv <pv> -o jsonpath='{.spec.nodeAffinity}' | jq

# 5. Which node was the original first consumer?
kubectl get pvc <pvc> -o jsonpath='{.metadata.annotations}' | jq
#   volume.kubernetes.io/selected-node

# 6. What families exist in the cluster?
kubectl get nodes -L node.kubernetes.io/instance-type,topology.kubernetes.io/zone
```

Step 5 is the archaeology: the `selected-node` annotation names the node that fixed the disk's fate, even if that node was deleted long ago.

## Key takeaways

- **`WaitForFirstConsumer` defers provisioning until the first scheduling decision, once, permanently.** It's a zone-placement optimization, not a compatibility gate, and it has no say over any pod after the first.
- **`Bound` is a control-plane pairing between PVC and PV.** It proves a disk exists and this claim owns it. It says nothing about whether any node can attach it.
- **Provision → Bind → Attach → Mount are four stages with four owners.** Machine-family incompatibility lives in Attach, which no PVC field can report on.
- **PV topology is zone-shaped.** The scheduler enforces zone affinity and attach limits; machine family is invisible to it. A same-zone but wrong-family node looks perfectly valid.
- **Check for a `VolumeAttachment` before diagnosing.** If the pod is `Running`, attach succeeded and there is no incompatibility — only a zone that was pinned earlier than you expected.
- **Express compatibility on the pod:** `nodeAffinity` on machine family, plus taints on the outgoing pool. Storage objects cannot carry this constraint, and `allowedTopologies` won't help.
- **In a family + disk-type migration, constrain compute first, then switch the StorageClass.** The storage decision is irreversible; the compute decision is re-made on every eviction.

## Related notes

- [[notes/K8s/elasticsearch-as-a-service-on-kubernetes|Elasticsearch as a Service on Kubernetes]] — why storage is the rigid layer of a stateful platform, and how disk choices constrain everything above them
- [[notes/K8s/kubernetes-autoscaling|Kubernetes Autoscaling]] — the scheduling and node-provisioning machinery that decides where a pod actually lands
- [[notes/K8s/vpa-eviction-loops|The VPA Eviction Loop]] — another failure where a reschedule you didn't ask for moves a pod somewhere it shouldn't be
