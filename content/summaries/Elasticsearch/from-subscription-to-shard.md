---
title: "Summary: From Subscription to Shard"
---

> **Full notes:** [[notes/Elasticsearch/from-subscription-to-shard|From Subscription to Shard: The Write Path of a Managed Search Platform -->]]

## Key Concepts

### The write path
- Producer publishes **full documents + attributes** to one topic (7d retention); each index tier = a **subscription with a server-side attribute filter** (7d/90d/365d tiers off one topic).
- A subscription is a **queue, not a broadcast**: one subscription per consuming index, or clones split the stream. It's also **passive** — creation triggers nothing, just starts buffering backlog.
- Consumer = a **streaming Dataflow "feeder"**, one per index per cluster. Launched by the lifecycle Job's init container, which *is* the Beam driver: `DataflowRunner` self-submits and exits. Workers are VMs **outside the k8s cluster** → ES endpoint must be an internal LB IP, not cluster DNS.
- Pipeline: `ReadFromPubSub → parse → BatchElements(20000, 3s) → bulk`. Worker sizing: ~1 worker vCPU per 5 data-node vCPUs.

### Delivery semantics
- **Ack after bundle commit** ⇒ at-least-once; failed bulk fails the bundle ⇒ redelivery.
- **External versioning** (`_version_type: external` / `external_gte` at 0) makes redelivery + reordering harmless: stale writes bounce as **409 = success**, delete-404 = success, 429 retried, rest fail the bundle.
- **Completeness proof chain:** backlog → ~0, oldest-unacked-age → seconds, watermark age → seconds, `index_failed` = 0, doc count converging, spot-check `_version` of a recent doc on both clusters. Proves delivery of what the *subscription* received — nothing before `T_sub`.

### Bootstrap arithmetic (cloning an index)
- Snapshot covers `(…, T_snap]`; subscription backlog covers `[T_sub, …)`. **Rule: `T_sub < T_snap`** (create subscription, wait one snapshot interval, then create index).
- Overlap `[T_sub, T_snap]` is replayed blind and deduped by versioning — **a 409 spike during catch-up is the mechanism working**.
- Gap repair: re-restore from a newer snapshot, or `seek` the subscription back past `T_snap` (topic-level retention allows seeking before the subscription's creation).

### Restore internals (who does what)
- HTTP lands on a **coordinating node** (holds the connection under `wait_for_completion=true`) → transport call to the **elected master**.
- **Master**: validates (incl. "open index already exists" rejection), reads snapshot metadata, creates the index in cluster state, records `RestoreInProgress`, allocates primaries with recovery source `SNAPSHOT`, flips to `SUCCESS` — logged on the *master*.
- **Data nodes** pull segment files **directly from object storage, in parallel**. Replicas are **not** restored — peer recovery from the live primary afterwards.
- Coordinating holds *connections*, master holds *state*, data holds *bytes* ⇒ restores survive caller death **and** master failover.
- Bulk path: coordinating hashes `_id` → shard → primary → sync replica → ack; visible at next refresh (1m).

### The incident
- `restore-snapshot` init container died twice with **curl exit 52** (~5 min in) = clean FIN, zero response bytes — someone *chose* to close.
- Hypothesis "envoy idle timeout" falsified in one command: `istio-injection: disabled`, no sidecar. **Verify the suspect exists.**
- Real cause via events: coordinating tier is **derived from the indices' zone list**; cluster oscillating 0↔1 indices renamed/rebuilt the client StatefulSet on each deploy → 5 fresh idle clients → HPA scaled 5→4→3 minutes later → killed the pod holding the silent connection.
- Master log: restore `SUCCESS` **2.5 min after curl gave up** — state lives in replicated cluster state, decoupled from the caller.
- Aggravator: `check-nodes` gates on **≥1** node, so restore started with 1/12 data nodes and stretched to 7m20s.

### Fire-and-forget scripts
- Restore step ends with `curl -s` (no `--fail`, no body check); `bash -c` exit = last command ⇒ HTTP 500 rejection → **container exits 0**. Rerun-with-existing-index passes: rejected restore swallowed, HEAD/200 checks only prove existence, reroute idempotent.
- "Already there" and "just restored" indistinguishable — a defect that was also the recovery path.
- Verify teammates' claims against **current code**; weigh risk asymmetry (rejected restore = zero side effects ⇒ trying beats debating).

### The fix
- **Poll, don't hold:** submit `_restore` async, loop `_cluster/health/<index>?wait_for_status=green&wait_for_no_initializing_shards=true&timeout=30s`; parse `status` + `timed_out`; exhaust budget ⇒ exit 1.
- No request outlives 30s (dropped response = one iteration); green + no-init = real completion gate (primaries *and* peer-recovered replicas); 404 body fails closed (numeric `status` ≠ quoted regex). Budget = `retry × (timeout + interval)` — size for your biggest index.
- Generalizes to every `wait_for_completion`-shaped API once expected duration > ~1 min.

### Layering
- Terraform/bus ↛ snapshots; CUE ↛ runtime; operator ↛ indices; Jobs ↛ their own history (spec-hash names); ES ↛ the nodeSet scheme; feeder ↛ why the index exists. Glue = node attrs ↔ allocation filters, external versions, naming conventions.
- **Deletion needs two actors in order** (REST deletes index → operator reclaims nodeSet); deleting config outright wedges the drain.
- **Spec-hash revert collision:** `init → deleted → init` regenerates the *same* Job name; a lingering failed Job blocks the re-run — delete the object first.

## Quick Reference

```bash
# Where is the lifecycle Job stuck?
kubectl -n <ns> get pods -l job-name=<job> \
  -o jsonpath='{range .status.initContainerStatuses[*]}{.name}{"  "}{.state}{"\n"}{end}'
kubectl -n <ns> logs <pod> -c restore-snapshot

# Restore progress / completion (per shard: type=snapshot, stage=done)
GET /_cat/recovery/<index>?v&h=shard,type,stage,bytes_percent
GET /_cluster/health/<index>?wait_for_status=green&wait_for_no_initializing_shards=true&timeout=30s

# Who killed my connection? (events timeline around the failure)
kubectl -n <ns> get events --sort-by=.lastTimestamp | grep -iE "client|Killing|Rescale"

# Feeder health
gcp.dataflow.job.data_watermark_age     # declining=catch-up, rising+backlog=stuck, 1s/s=draining
pubsub num_undelivered_messages / oldest_unacked_message_age
Beam counters: index_success / version_conflict(=ok) / not_found(=ok) / index_failed(=bad)

# Rewind a subscription past the snapshot cut (gap repair)
gcloud pubsub subscriptions seek <sub> --time=<T_snap>
```

| Symptom | Likely meaning |
|---|---|
| curl exit 52 mid-wait | peer chose to close: LB / sidecar / **autoscaler killed the pod** |
| 409 spike during bootstrap | snapshot∪backlog overlap being deduped — healthy |
| watermark age ramping 1s/s | drain in progress, watermark frozen |
| restore "failed" but master log says SUCCESS | only the *connection* died; state survived |
| rerun passes instantly | fire-and-forget script swallowed "already exists" |
