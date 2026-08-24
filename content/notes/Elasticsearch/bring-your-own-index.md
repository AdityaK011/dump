---
title: "Bring Your Own Index: The Custom-Tenant Model of a Managed Search Platform"
---

## The tier where the platform stops

The previous two notes in this series walked a managed Elasticsearch platform from the top down: [[notes/Elasticsearch/elasticsearch-as-a-service-on-kubernetes|the cluster anatomy]] (operator, nodeSets, lifecycle state machine, autoscaling) and [[notes/Elasticsearch/from-subscription-to-shard|the write path]] (filtered subscriptions, streaming feeders, snapshot bootstrap, restore internals). Both described the *fully-managed* experience: declare an index in config, get a nodeSet, a feeder, snapshots, retention, an autoscaler, and a per-index SLO.

This note is about the other end of the slider — the tenants for whom the platform provides a cluster and *nothing else*. I ended up mapping this tier while reviewing a data-node migration for one such cluster (a recommendations team's), and it turned out to be the most instructive tier of the three: every abstraction the platform withholds is an Elasticsearch internal the tenant now touches bare-handed. If you want to learn what a search platform actually does for you, study the tenant that doesn't get it.

## The ownership slider

One platform, three tenancy tiers, distinguished by who owns the ES *data plane*:

| Tier | Platform provides | Tenant provides | Fits when |
|---|---|---|---|
| **Fully-managed index** | nodeSet + lifecycle Jobs + feeder + retention purge + snapshots + autoscaler + per-index SLO | an index entry in config, a message-bus subscription | one big stream-fed index that fits the mold |
| **Index-template** | nodeSet + an ES index template pinning `<prefix>*` onto it | runtime index creation, ingestion | app creates its own (rollover/dated) indices but wants hardware isolation |
| **Custom cluster** | the cluster itself, fixed sizing | indices, indexer, retention, backups, capacity PRs | many small indices, exotic mappings, existing ingestion stack |

(A fourth path sits orthogonal to these: a gRPC indexing/query gateway with an `index_id` parameter, for teams that don't want to speak Elasticsearch at all. Different trade — you give up the query DSL's full surface for never operating anything.)

The rest of this note dissects the third row.

## What gets rendered for a custom cluster — and what doesn't

A custom cluster's entire config is ~60 lines: a `cluster` block (image, owner team, compute class), `master` (3 nodes), `client` (coordinating nodes), and a `data` list of **hand-declared nodeSets**. No index entries, no templates. And the hydrated manifest makes the boundary concrete: the only generated Job in the entire output is a cluster-settings applier. Compare, machinery by machinery:

| Machinery | Fully-managed tier | Custom cluster |
|---|---|---|
| index creation / restore Jobs | generated per index entry, hash-named | none — tenant's app creates indices via REST |
| streaming feeder | self-submitting launcher per index | none — tenant runs their own indexer service |
| retention purge CronJobs | per index entry | none |
| snapshot CronJobs | per cluster with indices | **none — no backups, no restore path** |
| index-aware autoscaler | per index entry | none — `count` is fixed; scaling is a PR |
| allocation pinning (`include.group`) | set by lifecycle Jobs | not set — shards float freely |
| alerting | per-index availability SLO | cluster green/yellow/red only |

The last row is the platform admitting the boundary in its own telemetry: every node carries an alert-mode tag rendered from config — `index` for clusters whose indices the platform knows, `cluster` for the rest. A custom cluster is monitored at cluster granularity *because the platform literally doesn't know what indices exist inside it.*

One dormant detail worth noticing: every data pod still gets its `node.attr.group` attribute injected, even though no index references it. Attribute-based pinning is plumbing the platform installs everywhere and activates only where an index declares a filter. That dormancy turns out to matter for migrations (below).

## How a tenant reaches the cluster

Three deliberate choices define the access model:

**Reachability is the credential.** The rendered ES config enables anonymous authentication (`xpack.security.authc.anonymous`) and disables TLS on the HTTP layer. Any client that can reach `:9200` can read and write. Authentication is therefore *network-level* — VPC and namespace policy — not Elasticsearch-level. This is also what lets platform components curl the cluster with no secret management at all. It's a coherent stance (one enforcement layer instead of two half-enforced ones), but it means "who can write to prod search" is a networking question, and you should know that's where the answer lives.

**In-cluster callers use the coordinating Services.** Per-zone client Services (ClusterIP) front the coordinating pods; Kubernetes Services are cluster-wide addressable, so a tenant's indexer in its own namespace connects cross-namespace by DNS.

**Out-of-cluster callers get an internal LB.** The cluster's HTTP Service is exposed as an internal LoadBalancer on the VPC — that's how anything running as raw VMs (the platform's own Dataflow feeders, a tenant's batch jobs) reaches ES without being in the Kubernetes cluster.

### The mesh config that executes in someone else's pod

The subtlest piece: the ES pods run with **no service-mesh sidecars** (the namespace disables injection), yet the platform renders Istio `DestinationRule` and `VirtualService` objects for the cluster. Not a contradiction — mesh routing config attaches to the *destination hostname* but executes in the *caller's* sidecar. So a meshed tenant calling the cluster gets, without the cluster participating in the mesh at all:

- `DestinationRule`: `LEAST_REQUEST` load balancing with a 60-second slow-start warmup — new coordinating pods ramp traffic gradually instead of being slammed while the JVM is cold;
- `VirtualService`: **URI-prefix routing per index** — `/<index>/_search` is matched on the path and routed to the client Service in that index's zone, keeping search traffic zone-local; the same mechanism carries weighted routes for shifting an index's traffic to a sibling cluster during migrations.

On a custom cluster the per-index match list is empty and the VirtualService collapses to a default route — but the warmup and least-request behavior still protect it. The design generalizes: **you can give callers L7 smarts about a backend that has no sidecar, because the config's execution point is the client.** Worth remembering any time someone says a backend "can't be in the mesh."

## What the tenant rebuilds

Everything in the fully-managed column becomes ordinary application infrastructure on the tenant's side:

**Ingestion.** Their own indexer service consuming *their own* message-bus subscriptions and bulk-writing to the cluster. Structurally identical to the platform feeder (queue → transform → `_bulk`), but every property the platform feeder embodies is now a design decision they had to make and maintain themselves: idempotence under at-least-once delivery (the platform uses external versioning — see [[notes/Elasticsearch/from-subscription-to-shard|the write-path note]]; whatever the tenant's indexer does is their code), back-pressure on 429s, freshness monitoring (no data-watermark metric unless they build one), and the one-consumer-per-subscription rule.

**Schema and lifecycle.** Index creation, mappings, settings, aliases, reindexes — via the REST API from their app's startup/migration code or their tooling. No state machine, no one-shot Jobs; their deploys *are* the lifecycle.

**Retention and backups.** Their own delete-by-query jobs, if any. And no snapshot repository is configured, so point-in-time recovery **does not exist** unless they build it. Said plainly: on a custom cluster, losing the data nodes loses the data, modulo replicas. The delete-and-re-restore recovery move that saves fully-managed rollouts isn't available here.

**Capacity.** A human edits `count` and opens a PR. That's the autoscaling story.

## The ES internals you now face directly

The pedagogically interesting part. Tier by tier, the platform hides internals; at the custom tier, they're all exposed:

### Cluster state and the master

Creating an index, changing settings, adding an alias, registering a snapshot repo — all cluster-state mutations, executed by the *elected master* no matter which node received the HTTP request (coordinating nodes forward them over the transport layer). A tenant app that creates indices at startup is doing master work on every deploy. The role split to internalize — **coordinating nodes hold connections, the master holds state, data nodes hold bytes** — predicts every failure mode: kill any one and only its own layer is lost.

### Allocation without filters

Fully-managed indices are pinned (`index.routing.allocation.include.group` ↔ `node.attr.group`, one index per nodeSet). A custom cluster's indices typically set no filter, so ES's **balancer** governs placement: equalize shard counts and disk across all eligible data nodes, under the hard invariant that a primary and its replica never share a node — which is why a 1-replica index needs ≥2 data nodes before anything schedules. Freedom cuts both ways: no noisy-neighbor isolation between the tenant's indices, but also no wedging when nodes change, because filters are what block drains. (Related knob spotted in the config: coordinating nodes can run with `es.search.ignore_awareness_attributes=true`, trading zone-affinity for pure load balancing on the search path — a platform-level latency choice.)

### The write and read fan-outs

A bulk request hits a coordinating pod → split per document → `hash(_id)` → shard → primary indexes and *synchronously* replicates → coordinating node reassembles the response. Visibility comes at the next refresh — and on a custom cluster `refresh_interval` is whatever the tenant set, including the 1s default, which on a write-heavy index is a real indexing tax the managed tier's 1m default avoids. Search is the mirror image: scatter the query phase to one copy of each shard, gather, fetch. Coordinating-tier sizing is downstream of these two fan-outs and nothing else.

### Relocation is peer recovery, and the operator drives it

Shard relocation — behind rebalancing, drains, and migrations — is the same peer-recovery machinery that builds replicas: the destination node copies segment files from the source over the transport layer, the master flips the routing table, the old copy is deleted. The operator (ECK) drives it declaratively on downscale: for each departing node it sets `cluster.routing.allocation.exclude._name`, waits for the node to hold **zero shards**, then deletes the pod. On an unfiltered cluster this always converges — any node is a legal destination. On a filtered one it wedges unless the filter admits somewhere to go.

### Replicas are the scaling quantum

No autoscaler doesn't repeal the arithmetic: ES read capacity scales by whole replica copies. You can't hold 1.5 copies, so a tenant scaling for reads goes 2 → 4 → 6 data nodes, never 2 → 3. The managed tier's autoscaler encodes this quantization; custom tenants re-derive it with a calculator and a PR.

### Disk watermarks are the backstop

Fixed capacity meets organic growth at ES's own disk watermarks: at the low mark (85%) a node stops accepting new shards; at high (90%) shards actively relocate away — nowhere to go on a 2-node group; at flood-stage (95%) indices flip read-only. On the managed tier the autoscaler outruns this; on a custom cluster a human PR has to.

## Case study: an architecture + storage migration in two PRs

The migration that prompted this note: move a custom cluster's data nodes from an x86 machine family on standard SSD to an arm64 family on a newer unbundled-billing disk type. On the custom tier this was *two declarative PRs*:

1. **Add alongside.** A second data nodeSet (`nodeGroup: "item"`, `arch: arm64`) beside the old one (`nodeGroup: "items"`), plus a platform rule mapping `arch: arm64` to the new storage class. The operator creates the StatefulSet; two arm64 pods join; and because no index carries an allocation filter, the balancer immediately treats them as eligible and starts spreading shards.
2. **Drain by deletion.** Remove the old nodeSet from config. The operator's shard-aware downscale excludes each departing node, waits for zero shards (relocation = peer segment copy to the new nodes), then deletes the pod. **Merging the deletion PR *is* the data migration.**

Lucene segment files are architecture-independent, so x86 → arm64 required nothing on the data side — only a multi-arch ES image. The one real trap was scheduling, not data: in testing, a pod got its new-type PVC provisioned while the pod itself landed on an old-family node and hung at `Init` forever, because the new disk type can't attach to the old machine family and the scheduler can't see machine-family/disk compatibility. Bound is not attached — the full anatomy of that trap is in [[notes/K8s/waitforfirstconsumer-bound-vs-attached|its own note]]. The fix was an explicit contract label (a `<disk-type>-support=true` nodeSelector plus an arch toleration) rather than trusting storage class + compute class to imply placement.

**Why it was clean, and when it wouldn't be.** Unfiltered shards plus no autoscaler meant nothing fought the drain. On a fully-managed index the same two PRs would wedge at the drain step — shards excluded from the old nodes but not *included* on the new group — so you'd first widen `include.group` to cover both group values (it takes a list; the index setting is the migration control plane) or give the new nodeSet the same attribute value. And you'd need one autoscaler spanning both pools, because a controller that writes shared state (the index's replica count) derived from local state (its own pool's node count) cannot be duplicated. The custom tier's freedom is exactly the absence of those two constraints.

## The operational bill

What the tenant gave up is invisible until something breaks:

- **Alert granularity.** Cluster green/yellow/red is the only platform signal. An index gone red inside a yellow cluster, a lagging indexer, a mapping explosion — all invisible to the platform, all the tenant's pager.
- **No restore path.** Recovery is replicas and re-ingestion from source, full stop.
- **Health gates the operator.** The operator's rolling operations (image bumps, migrations like the above) wait on cluster health. A perpetually-yellow custom cluster — which the platform may not even notice — silently blocks its own upgrades.
- **Every scaling decision has a human in the loop**, against a system whose failure backstop (disk watermarks → read-only) is abrupt.

## Key takeaways

- **A platform's tenancy tiers are defined by which internals they hide.** Fully-managed hides allocation, versioning, lifecycle, and scaling; index-template hides hardware placement; custom hides nothing. Debugging any tier means knowing the internals it hides from you.
- **Reachability-as-credential is a coherent security stance** — one enforcement layer (network) instead of two half-maintained ones — but know that "who can write to search" is then a networking question.
- **Mesh config executes in the caller's sidecar**, so a sidecar-less backend can still give meshed callers least-request balancing, slow-start warmup, and L7 per-index routing. The backend doesn't need to join the mesh for its clients to be smart about it.
- **Dormant plumbing is a design pattern:** install the pinning attribute on every nodeSet, activate it only where an index declares a filter. Uniform mechanism, per-tenant policy.
- **On an unfiltered cluster, the operator's exclude → drain → delete downscale *is* a data migration tool.** Add the new nodeSet, delete the old one, done. Allocation filters are what turn the same move into a multi-step dance.
- **Segments are portable; images aren't.** CPU-architecture migrations need a multi-arch image and correct scheduling, not data surgery — and the scheduling trap is bound-vs-attached, not anything Elasticsearch.
- **The custom tier's true costs are the absences:** no snapshots means no point-in-time recovery; cluster-level alerting means index failures page nobody; fixed counts mean disk watermarks are the autoscaler.
- **Choose the tier by who should own the data plane**, not by who wants control. Control of ingestion, schema, and retention is also ownership of their failure modes.
