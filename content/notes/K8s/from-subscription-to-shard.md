---
title: "From Subscription to Shard: The Write Path of a Managed Search Platform"
---

## One Terraform resource, one failed Job, every layer

The task looked administrative: enable a scaling feature on a production Elasticsearch index. The catch was that flipping the feature triggers mass shard relocation on a live index, so the rollout plan was a cluster-level blue/green — clone the index onto a sibling cluster, shift search traffic over, make the disruptive change on the now-idle original, shift traffic back, delete the clone.

Executing that plan meant touching every layer of the platform's *write path*: a Terraform PR creating a single Pub/Sub subscription, a GitOps index entry that hydrates into Kubernetes Jobs, a Dataflow streaming job nobody starts by hand, an internal load balancer, a coordinating node, a master, and finally the data nodes. And because a Job failed twice in the middle of it — with `curl` exit 52, while the operation it was waiting for succeeded both times — I ended up having to actually understand each hop instead of trusting the runbook.

This note is the companion to [[notes/K8s/elasticsearch-as-a-service-on-kubernetes|Elasticsearch as a Service on Kubernetes]], which covers the cluster anatomy: nodeSets, shard pinning, the index lifecycle state machine, autoscaling. This one covers how a document *gets into* that cluster — the ingestion pipeline, the snapshot-bootstrap arithmetic, what each Elasticsearch node role actually does during a restore and a bulk write, and the production race we hit at the seam between them.

## The write path at 10,000 feet

Two chains, one data and one control, meeting at a Dataflow job:

```
DATA                                          CONTROL
producer/analyzer                             GitOps index entry (CUE)
      │ publish: full doc + attributes              │ hydrate + deploy
      ▼                                             ▼
topic (7d retention)                          one-shot k8s Job (initContainer chain)
      │ server-side attribute filter                │ last init step: "run-feeder"
      ▼                                             │ = the Beam driver, self-submits
subscription (cursor + backlog)                     ▼
      │ streaming pull                        Dataflow streaming job
      └────────────────────────────────────► "feeder-<cluster>-<index>-<ts>"
                                                    │ bulk writes
                                                    ▼
                                        internal LB (es-http service, :9200)
                                                    ▼
                                        coordinating node (client tier)
                                                    │ hash(_id) → shard, route
                                                    ▼
                                        primary shard ──sync──► replica
                                                    ▼
                                        visible at next refresh (1m)
```

Everything below is a walk down this diagram, with the failure story where it actually happened.

## A subscription is a cursor, not a trigger

The upstream producer (an analyzer service) publishes the **full analyzed document** for every entity change onto one topic, with message attributes describing it: `is_deleted`, `status`, `within_7d` / `within_90d` / `within_365d` freshness flags, and so on. The topic retains messages for 7 days regardless of consumption.

Every index tier is then just a **subscription with a server-side attribute filter** on that one topic:

```
(NOT attributes:index OR attributes.index = "listings")
AND ( attributes.is_deleted = "true"
   OR attributes.status = "on_sale"
   OR attributes.creation_timestamp = "unknown"
   OR attributes.within_90d = "true" )
```

One topic, N filtered subscriptions, N indices with different retention windows. The filter runs broker-side, so a 90-day index's consumer never even receives the messages it would drop.

Three properties of this arrangement do all the work later:

1. **A subscription is a queue, not a broadcast.** Pub/Sub delivers each message on a subscription to *one* subscriber. Two consumers sharing a subscription split the stream between them — each index copy gets half the updates, silently. So the rule is structural: **one subscription per consuming index, always.** Cloning an index onto a sibling cluster starts with cloning its subscription.
2. **A subscription is passive.** Creating it triggers nothing. From the moment it exists it accumulates every matching message as unacked backlog — a buffer waiting for a consumer that doesn't exist yet. That's not an oversight; it's the bootstrap mechanism (next section).
3. **Topic-level retention enables `seek`.** Because the *topic* retains 7 days, a subscription can be rewound to a timestamp before its own creation (`subscriptions seek --time=...`). That's the escape hatch when the bootstrap ordering goes wrong.

Nothing watches the infrastructure-as-code layer. Merging the Terraform PR that creates the subscription changes nothing at runtime — a fact that matters because everyone's mental model wants "adding the subscription creates the pipeline," and it simply doesn't.

## One feeder per index per cluster, launched by a Job that exits

The thing that consumes the subscription is a streaming Dataflow job (Apache Beam), one per index per cluster — the platform calls them *feeders*. Nobody deploys them by hand, and there's no controller. The launch mechanism is worth stealing:

The index's lifecycle Job (the hash-named, one-shot, initContainer-chained Job described in the companion note) has as one of its final init steps a container whose image is **the Beam driver program itself**. Run with `--runner=DataflowRunner`, a Beam driver doesn't process data — it builds the pipeline options and asks the Dataflow API to run the pipeline on its behalf:

- job name: `feeder-<cluster>-<index>-<unix-ts>` (the prefix is how the *stop* step later finds it)
- project/region/zone, a shared-VPC subnetwork, a custom worker image, streaming engine on
- worker sizing heuristic: `maxNumWorkers = ceil(dataNodeCPU / workerCPU / 5)` — one worker vCPU per five data-node vCPUs, on the logic that indexing throughput should scale with the cluster it feeds
- labels (`cluster-name`, `index-name`, `index-status`, `owner-team`) that the monitoring later groups by

The container submits and exits; the Kubernetes Job completes. From then on the feeder is GCE VMs managed by Dataflow — **outside the Kubernetes cluster entirely**. That's also why the Elasticsearch endpoint it writes to must be an *internal load balancer IP* rather than a cluster DNS name: the workers are on the same VPC but not in the cluster, so `elasticsearch-<cluster>-es-http` is exposed as an internal `LoadBalancer` service on `:9200`.

The step *before* launch is a drain of any existing feeder matching the name prefix, so re-running a lifecycle Job replaces the feeder rather than racing a second one onto the same subscription (see property 1 above for why that would corrupt the stream).

The pipeline itself is four steps:

```
ReadFromPubSub(subscription)        # streaming pull, with attributes
  | parse                           # json.loads; dict or list-of-dicts → docs
  | BatchElements(20000, 3s)        # best effort; streaming batches run small
  | WriteToElasticsearch            # docs → bulk actions → helpers.bulk
```

`BatchElements` with a 20k target sounds aggressive, but in streaming mode batches are bounded by the runner's bundle size and mostly run tiny. The interesting step is the last one.

### Freshness is a watermark, not a backlog count

A streaming pipeline never "finishes," so its progress metric is the **data watermark**: a timestamp T meaning *every message older than T has been fully processed through the sink*. The exported metric is its age, `now − T`. For a feeder, "fully processed" means pulled, bulk-written, and acked — so watermark age *is* index freshness, in time units, accounting for in-flight work in a way a backlog count can't.

The operational signatures are worth memorizing:

| Signature | Meaning |
|---|---|
| age high but **declining** | feeder recently (re)started, chewing through backlog — expected |
| age and backlog **both rising monotonically** | pipeline stuck (cluster red, index read-only, worker crash) |
| age ramping at **exactly 1s/s** | the watermark is *frozen* — a drain is in progress |

That last one has a sharp consequence: draining a feeder freezes its watermark, so a graceful drain of a job you're decommissioning fires the freshness alert for the entire drain window (~30+ minutes observed). If the index is about to be deleted anyway, flushing in-flight data into it is pointless — **cancel, don't drain, when the destination is doomed.** The deletion path here encodes exactly that choice.

## External versioning makes at-least-once safe

The write transform converts each document into a Bulk API action. Every document carries its own intent and its own ordering token:

- `_op_type`: `index` (default), `update`, or `delete`
- `_id`: the entity id
- `_version`: mapped to Elasticsearch **external versioning** (`_version_type: external`, or `external_gte` when the version is 0)

External versioning means the *producer's* version number is authoritative: Elasticsearch accepts a write only if the incoming version is newer than what's on disk, and rejects it with a **409 version conflict** otherwise. That single mechanism is what makes the whole pipeline's delivery semantics safe:

- **The stream has no ordering guarantee.** A stale update arriving after a newer one bounces off as a 409 instead of overwriting fresh data with old.
- **Delivery is at-least-once.** Pub/Sub acks only after the Beam bundle commits, and a bundle commits only after the bulk write succeeded. A failed bulk fails the bundle → runner retries the bundle → no ack → redelivery. Redelivery is harmless because the replayed writes are version-checked.
- The bulk writer therefore treats **409 as success** (stale, correctly dropped — it's a health signal, not an error), **404 on delete as success** (already gone), retries 429s with backoff, and raises on anything else — which fails the bundle and loops back to redelivery.

The pipeline exports counters for each class (`index_success`, `version_conflict`, `not_found`, `index_failed`), and the conflict counter turns out to be diagnostic during bootstrap — more below.

### The completeness proof chain

"Did the feeder pull everything and get it into the cluster?" decomposes into signals that each prove one link, and the ack semantics are what glue them together — backlog empty doesn't just mean *read*, it means *written and acknowledged*:

1. `num_undelivered_messages` on the subscription → drains to ~0 and stays flat.
2. `oldest_unacked_message_age` → seconds.
3. Watermark age → seconds.
4. `index_failed` counter → 0 (conflicts and not-founds don't count).
5. Doc count on the new index converging with the source index — never exactly equal, since both are moving targets with retention purges trimming every minute, but tracking within noise.
6. Sharpest spot check: pick an entity updated after the snapshot cut, `GET /<index>/_doc/<id>` on both clusters, compare `_version`. Equal versions = the update flowed through both pipelines.

One boundary: all of this proves delivery of everything *the subscription ever received*. It says nothing about messages published before the subscription existed. That window is someone else's job:

## Bootstrapping a clone: snapshot ∪ backlog, glued by arithmetic

To clone a live index onto a sibling cluster you need its history *and* its ongoing updates, and the two come from systems that know nothing about each other:

- A **snapshot** (taken every 15 minutes on the source cluster) is a point-in-time copy: complete up to its cut time `T_snap`.
- The **subscription** buffers everything from its creation time `T_sub` onward. Messages published before `T_sub` were never put in that buffer, full stop — Pub/Sub has no idea snapshots exist.

There is no coordination, no handshake, no dedup service. The gaplessness is pure interval arithmetic, forced by one runbook rule: **create the subscription before the snapshot you'll restore from.** If `T_sub < T_snap`, then snapshot covers `(…, T_snap]`, backlog covers `[T_sub, …)`, and the union covers everything.

Nobody deduplicates the overlap `[T_sub, T_snap]` either. The feeder blindly replays those messages against data the restore already contains in newer-or-equal form — and external versioning discards them as 409s. **A version-conflict spike during bootstrap catch-up is the overlap being safely dropped.** It looks alarming on a dashboard and is in fact the proof the mechanism works.

If the ordering rule is violated (`T_sub > T_snap`), there's a hole `(T_snap, T_sub)` that neither side captured. Two repairs, both correct: re-restore from a snapshot newer than the subscription (boring, procedure-shaped), or `seek` the subscription back past `T_snap` and let versioning re-idempotent the re-covered overlap (needs the topic-retention window to still reach back that far).

The practical checklist that falls out: merge the subscription change, **wait one snapshot interval**, then merge the index entry. A fifteen-minute wait encoded in a runbook, justified by set theory.

## What each node role actually does in a restore

This is the part I had exactly backwards until the incident forced the issue. A snapshot restore *feels* like a data operation, so intuition says the data nodes run it. In fact it's a three-role choreography, and knowing who holds what predicts every failure mode.

The API path first. The restore request (`POST /_snapshot/<repo>/<snap>/_restore`) arrives at the cluster's HTTP endpoint — a service selecting the **coordinating-only pods** (every node role set to `false`; pure fan-out/merge nodes, called the "client" tier here). With `?wait_for_completion=true`, whichever coordinating node accepted the request holds the HTTP connection open until the whole restore finishes. Internally it forwards the request over the transport layer to the **elected master**, because a restore is a cluster-state mutation, and only masters mutate cluster state.

The master then does everything decision-shaped:

1. **Validates** — repository reachable, snapshot exists, and the check that matters later: *is there already an open index with this name?* (that rejection is master-side);
2. reads the snapshot **metadata** from object storage;
3. **creates the index in the cluster state**, applying renames and `ignore_index_settings` from the request;
4. records a **`RestoreInProgress`** entry in the cluster state;
5. **allocates each primary shard** to a data node, with recovery source `SNAPSHOT`.

Only then do the **data nodes** move bytes — and they pull their assigned shards' segment files **directly from object storage, in parallel**. The 137 GB in our restore flowed bucket → data nodes; not one byte transited the master or the coordinating node. Each node reports its shard `STARTED`; when the last one lands, the master flips the restore to `SUCCESS` — a line that appears in the *master's* log and nowhere else. Which is exactly where a teammate later found the proof our "failed" restore had actually completed.

Replicas are the detail everyone misses: **replicas are not restored from the snapshot.** Once a primary is up, its replica is built by ordinary peer recovery — a copy from the live primary. A snapshot restore of a 6-primary, 1-replica index is six parallel bucket downloads followed by six peer recoveries.

Who holds what, and what that predicts:

| Role | Holds | If it dies mid-restore |
|---|---|---|
| coordinating (client) | the HTTP connection | **caller loses the response; restore unaffected** |
| master | `RestoreInProgress` in replicated cluster state | new master is elected, picks up tracking — restores survive failover |
| data | the bytes | its shards' recoveries reassign/retry |

The same role-split runs the steady-state write path, faster: a bulk request lands on a coordinating node, which splits the body per document, hashes each `_id` onto a shard number, consults the cluster-state routing table, and fans sub-requests to the primaries. Each primary indexes (checking the external version), forwards to its replica, and acks only after both have it. The coordinating node reassembles per-doc results into the bulk response. Documents become searchable at the next refresh — `1m` here, so worst-case visibility is about a minute after the ack. (Ingest nodes — the fourth role — would sit before the primary to run pipeline transforms; this platform does its transformation in the stream processor instead, so its ingest tier is idle in this path.)

The pattern to internalize: **coordinating nodes hold connections, the master holds state, data nodes hold bytes.** State and bytes are durable; connections are not. Which brings us to the incident.

## The war story: curl exit 52, twice

The clone's `create-index` Job — an initContainer chain of check-cluster → check-nodes → register-repository → **restore-snapshot** → verify → reroute-shards → check-subscription → stop-feeder → **run-feeder** — failed at `restore-snapshot`. Exit code 52: curl's *empty reply from server*. The container had submitted the restore (its log ends with `Restore from snapshot <name>`), then the `wait_for_completion=true` connection sat silent for ~4–5 minutes and was closed cleanly by the far end with zero response bytes. With `backoffLimit: 0`, one container failure is terminal: everything downstream — including the feeder launch — never ran.

Exit codes narrow the field before any other evidence: **52 is a clean FIN with no response** (someone deliberately closed), where a conntrack drop would hang into a timeout (28) and a crash usually RSTs (56). Something *chose* to close a silent connection.

### Hypothesis 1, falsified in one command

A ~5-minute kill on an idle HTTP stream smells like a service-mesh sidecar's idle timeout — Envoy's default stream-idle is exactly 300s. One check destroyed it: the namespace carried `istio-injection: disabled` and the Job pod had no sidecar container. **No mesh in the path.** (Worth the humility note: I'd already said "envoy" out loud with confidence. The user asked "which envoy, exactly?" — and looking for it is what broke the theory. Name the component and go verify it exists before blaming it.)

### The events timeline had the real killer

The deploy that (re)added the index didn't only create the index's data nodeSet. The coordinating tier's nodeSets are *derived from the zone list of the cluster's indices* — and this cluster oscillated between **zero and one** indices during the blue/green dance. Adding the first index flips the zone list from `[]` to `[zone-a]`, which **renames the client nodeSet**, which replaces the entire coordinating StatefulSet. The Kubernetes events told it plainly:

```
T+0s     Service  <cluster>-client-http           ADD
T+6s     StatefulSet <cluster>-es-client          create pods 0..4   (5 fresh clients)
T+70s    restore-snapshot container starts; POST /_restore?wait_for_completion=true
T+6m     HPA: SuccessfulRescale  New size: 4      "cpu utilization below target"
T+6m5s   Killing  <cluster>-es-client-4
T+8m     HPA: SuccessfulRescale  New size: 3
T+8m38s  Killing  <cluster>-es-client-3
```

Five brand-new, completely idle coordinating pods; an HPA that immediately and correctly judged them over-provisioned; and a multi-minute, bytes-silent HTTP connection pinned to one of them. The restore request's connection died at T+5m51s — inside the churn window, held by a pod the platform's own autoscaling was busy deleting. Both Job failures matched the same shape; it only *looked* like a deterministic 5-minute timeout because the HPA's reaction time to a fresh over-sized tier is itself fairly deterministic.

And the restore? The master's log: `started restore of snapshot […] / completed restore […] with state [SUCCESS]` — finishing **2.5 minutes after curl gave up.** Of course it did: the operation's state lives in replicated cluster state owned by the master, fully decoupled from whoever asked for it. A master was *also* replaced during that same deploy window, and even that wouldn't have mattered.

One aggravator worth its own line: the chain's earlier `check-nodes` step gates on **≥1 node** of the index's nodeSet existing, not all of them — so the restore began with 1 of 12 data nodes joined and stretched to 7m20s as the rest arrived. A completeness check that passes at "one" quietly doubles the window in which a fragile connection can die.

> **Root cause, in one sentence: a multi-minute silent HTTP connection, held by a disposable coordinating pod, raced against autoscaler churn that the very same deploy had caused — and lost, twice.**

## Fire-and-forget scripts, and the bug that was also the recovery path

Before re-running anything, a teammate warned: *"if the index already exists, the next step will fail."* Plausible — a restore onto an existing open index **is** rejected by Elasticsearch. Verifying the claim against the actual scripts taught more than the incident did.

The restore container's last line is, in shape:

```bash
curl -s -X POST "${ES_URL}/_snapshot/${REPO}/${SNAP}/_restore?wait_for_completion=true" \
     -H "Content-Type: application/json" -d "${BODY}"
```

No `--fail`. No check of the response body. And a `bash -c` script's exit code is its **last command's** exit code. `curl` without `--fail` exits 0 on an HTTP 500 as long as it received *a* response — trivially demonstrable with a local server returning the exact `snapshot_restore_exception` body: response printed, `exit code: 0`. So on a re-run, Elasticsearch *rejects* the restore and the container *reports success*, having restored nothing.

Walk the "next steps" and the claim fully dissolves: the post-restore check is `curl --head` grepping for HTTP 200 — an **existence** check that passes the moment the index exists, saying nothing about recovery; the reroute is an idempotent settings PUT with no success check at all; the settings check wants replicas ≥ 1 and the right refresh interval, both true. **No container in the chain has a pass-condition violated by the index pre-existing.** The pipeline is fire-and-forget end to end: creates and restores don't verify their own effect, and the checks that follow only verify existence.

That's a defect — "already there" and "just restored" are indistinguishable — and it was also precisely the recovery path: with the restore already completed server-side, re-running the Job would sail past the rejected restore and finally reach the feeder launch. Two lessons tangled together:

- **Verify a claim against the current code, not against anyone's memory — including your own.** The claim was reasonable, recently relevant, and wrong about this version of the script. Ten minutes of reading beat an afternoon of debate.
- **Weigh the risk asymmetry before litigating further.** A rejected restore has zero side effects. If the claim was right, a re-run cost one more failed Job with data untouched; if wrong, we were done. Trying was strictly cheaper than being sure.

## The fix: poll, don't hold

The proper fix landed as a change to the restore step, and its shape is the transferable part. Instead of one connection awaiting completion:

```bash
# submit without waiting
curl -s -X POST ".../_restore?pretty" -d "${BODY}"

# then poll a bounded completion gate
while [ ${COUNT} -le ${RETRY} ]; do
  RESULT=$(curl -s "${ES_URL}/_cluster/health/${INDEX}?wait_for_status=green&wait_for_no_initializing_shards=true&timeout=30s")
  # parse "status" (quoted string) and "timed_out"; break on green + false
  ...
done
[ ${COUNT} -gt ${RETRY} ] && { echo "restore did not complete"; exit 1; }
```

Every property the old version lacked:

- **No request outlives 30 seconds.** `_cluster/health?wait_for_status=…&timeout=30s` is a *bounded* long-poll: Elasticsearch answers within the timeout either way, with a `timed_out` flag. A dropped response now costs one loop iteration instead of the Job.
- **It's a real completion gate.** Green plus no initializing shards means primaries restored *and* replicas peer-recovered — the thing the HEAD/200 check never measured.
- **It fails closed.** If the restore never created the index, the health call's 404 body carries a *numeric* `status` field that doesn't match the quoted-string regex, so the loop retries until the budget exhausts and exits 1. The old script literally could not fail in that scenario.
- **The budget is arithmetic:** `retry × (timeout + interval)` — 30 × 40s = 20 minutes by default. Which invites the follow-up question every bounded gate deserves: our 137 GB restore took 7m20s *with* the slow node-join; a 3–4× bigger index eats the budget. Since the loop exits early on success, a generous default is free insurance.

The general rule, and it's bigger than Elasticsearch: **in an autoscaled environment, a long-lived silent connection is a bet that both endpoints outlive the wait — and you don't control either.** Any `wait_for_completion=true`-shaped API (reindex, forcemerge, snapshot, restore, task APIs everywhere) should be consumed as submit-then-poll the moment the expected duration exceeds a minute. The server-side operation surviving the dropped connection is what *saves* you; the client believing its exit code is what *bites* you.

## Who knows what: the layering recap

The whole system works because each layer is ignorant of the others, glued by conventions:

| Layer | Knows about | Ignorant of |
|---|---|---|
| Terraform / message bus | topics, subscriptions, filters | snapshots, indices, consumers' existence |
| GitOps config (CUE) | the *desired* index: sizing, status, subscription name | anything's runtime state |
| Operator (ECK) | nodeSets, pods, PVCs, certs | indices — entirely |
| One-shot Jobs | REST calls to create/restore/verify | whether they already ran (spec-hash name is the memory) |
| Elasticsearch | indices, allocation filters, versions | the one-index-per-nodeSet scheme, the queue, the feeder |
| Feeder (Dataflow) | subscription in, bulk actions out | why the index exists, what a snapshot is |

The glue points are exactly three: **node attributes ↔ allocation filters** (pins shards to nodeSets), **external versions** (stitches snapshot to backlog and makes redelivery safe), and **name conventions** (feeder job prefixes, subscription-per-index). Nothing coordinates; everything composes.

Two operational corollaries from the incident's periphery, both consequences of the split:

- **Deletion needs two actors in sequence.** The index must die via REST (a Job) *before* the operator reclaims the nodeSet — which is why the lifecycle uses a `status: deleted` state that generates the deletion Job *and* drops the nodeSet, and why deleting the config block outright wedges: the operator's graceful downscale waits forever for shards that have no legal home elsewhere.
- **The spec-hash Job name has a revert collision.** Flip `init → deleted → init` and the third state hydrates a Job with a name *identical* to the first — so if the failed original still exists, the apply matches it and nothing re-runs. Reverting config restores the *name*, not the *execution*. Delete the old Job object first.

## Key takeaways

- **One subscription per consuming index.** A subscription is a queue that splits messages across its subscribers; a clone needs its own. And it's a *cursor with a buffer*, not a trigger — creating it starts the recording, nothing else.
- **Bootstrap = snapshot ∪ backlog, and the union is arithmetic.** Subscription before snapshot, always; the overlap is deduped by external versioning; a 409 spike during catch-up is the mechanism working.
- **External versioning is what buys at-least-once safety.** Ack-after-commit makes redelivery inevitable; producer-owned versions make it harmless; 409 and delete-404 are successes.
- **Watermark age is freshness; know its three signatures.** Declining = catch-up; rising with backlog = stuck; exactly 1s/s = frozen by a drain. Cancel rather than drain a feeder whose destination is doomed.
- **Coordinating nodes hold connections, masters hold state, data nodes hold bytes.** Restores survive client death and master failover; replicas come from peer recovery, not the snapshot; the bytes flow object-store → data nodes directly.
- **`curl` exit 52 is a clean close by someone who chose to.** Enumerate who can choose: mesh sidecars, LBs, and — as here — the autoscaler deleting the pod that held your connection. Then verify the suspect *exists* before blaming it.
- **A deploy's blast radius includes derived resources.** A coordinating tier derived from a zone list means "add the first index" rebuilds the client StatefulSet — and a fresh over-sized tier plus an HPA is churn, scheduled during your most fragile window.
- **Fire-and-forget scripts make "already done" and "just did it" indistinguishable.** `curl -s` without `--fail` swallows HTTP errors; existence checks are not completion gates. Sometimes that bug is your recovery path — take it, then fix it.
- **Poll, don't hold.** Submit async, then loop a bounded gate (`_cluster/health?wait_for_status=green&wait_for_no_initializing_shards=true&timeout=30s`) with an exhaust-and-fail budget. Applies to every long-running task API on every platform.
- **Verify claims against current code, and weigh the risk asymmetry of just trying.** A zero-side-effect retry outruns any amount of debate.
