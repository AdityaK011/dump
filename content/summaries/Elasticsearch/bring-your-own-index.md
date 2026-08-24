---
title: "Summary: Bring Your Own Index"
---

> **Full notes:** [[notes/Elasticsearch/bring-your-own-index|Bring Your Own Index: The Custom-Tenant Model of a Managed Search Platform -->]]

## Key Concepts

### The ownership slider
- Three tiers by who owns the ES **data plane**: **fully-managed index** (platform: nodeSet + Jobs + feeder + purge + snapshots + autoscaler + per-index SLO), **index-template** (platform: nodeSet + template pinning `<prefix>*`; tenant creates indices at runtime), **custom cluster** (platform: hardware only).
- Orthogonal fourth path: a gRPC indexing/query gateway (`index_id` param) for teams that won't speak ES at all.
- Custom cluster config ≈ 60 lines: cluster/master/client + hand-declared `data` nodeSets. Hydrated output contains **one** Job (cluster-settings applier) — no index Jobs, feeders, purges, snapshots, or autoscalers.
- Platform telemetry admits the boundary: alert-mode tag = `index` for known indices, `cluster` for custom → **green/yellow/red is the only platform signal** on custom clusters.
- Dormant plumbing pattern: `node.attr.group` still injected on every data pod; inert until an index sets `include.group`. Uniform mechanism, per-tenant policy.

### Access model
- **Reachability is the credential:** anonymous auth (`xpack.security.authc.anonymous`) + TLS off ⇒ authn/z is network-level (VPC/namespace policy), not ES-level.
- In-cluster: per-zone coordinating (client) Services, cross-namespace by DNS. Out-of-cluster (VM workers, batch): internal LoadBalancer on `:9200`.
- **Mesh config executes in the caller's sidecar:** ES pods have no sidecars, yet rendered `DestinationRule` (LEAST_REQUEST + 60s slow-start warmup) and `VirtualService` (per-index URI-prefix routing to zone-local client Services; weighted routes for cluster-migration traffic shifts) give meshed callers L7 smarts about a meshless backend. Custom cluster ⇒ match list empty ⇒ collapses to default route, warmup still applies.

### What the tenant rebuilds
- Own indexer (queue → transform → `_bulk`): idempotence under at-least-once (platform uses external versioning; tenant's is their code), 429 back-pressure, freshness metrics, one-consumer-per-subscription.
- Schema/lifecycle via REST from their deploys — no state machine.
- Retention: their delete-by-query. **Backups: none exist** — no snapshot repo ⇒ no point-in-time recovery; recovery = replicas + re-ingestion.
- Capacity: a human edits `count` in a PR.

### ES internals exposed at this tier
- **Master/cluster state:** index create/settings/alias/repo = cluster-state mutations executed by the elected master (coordinating nodes forward via transport). Coordinating holds *connections*, master holds *state*, data holds *bytes*.
- **Allocation without filters:** the balancer equalizes shards/disk across all data nodes; hard invariant primary ≠ replica node ⇒ 1-replica index needs ≥2 nodes. No isolation, but no drain-wedging either.
- **Write path:** coordinating → `hash(_id)` → primary → sync replica → ack; visible at next refresh (`refresh_interval` is tenant's — 1s default is an indexing tax vs the managed 1m).
- **Relocation = peer recovery** (segment copy over `:9300`, master flips routing table). Operator downscale = per-node `exclude._name` → wait for 0 shards → delete pod; always converges unfiltered.
- **Replicas are the scaling quantum:** whole copies only, 2 → 4 → 6 nodes, never 2 → 3.
- **Disk watermarks are the backstop:** 85% no new shards / 90% relocate away / 95% read-only. No autoscaler outruns it here.

### Case study: arch + storage migration in two PRs
1. **Add alongside:** new nodeSet (`arch: arm64`, new storage class via platform rule) — balancer immediately spreads shards (no filters).
2. **Drain by deletion:** remove old nodeSet — operator exclude→drain→delete does the migration. **Merging the deletion PR is the data migration.**
- Segments are architecture-independent; only a multi-arch image needed. The real trap: PVC of the new disk type bound while the pod scheduled to an incompatible machine family → stuck `Init` (scheduler can't see disk/machine compatibility). Fix = explicit `<disk-type>-support=true` nodeSelector + arch toleration. See [[notes/K8s/waitforfirstconsumer-bound-vs-attached|Bound Is Not Attached]].
- On a fully-managed index the same move wedges: must widen `include.group` (it's a list — the index setting is the migration control plane) or reuse the attr value, and span both pools with **one** autoscaler (shared-state-from-local-state controllers can't be duplicated).

## Quick Reference

```bash
# Is anything pinned? (empty = balancer governs)
GET /<index>/_settings/index.routing.allocation.include.group

# Who's coordinating / what roles exist
GET /_cat/nodes?v&h=name,node.role,master

# Drain progress during a nodeSet removal
GET /_cluster/settings?filter_path=*.cluster.routing.allocation.exclude
GET /_cat/shards?v | grep RELOCATING

# Disk watermark danger check
GET /_cat/allocation?v&h=node,disk.percent,shards

# Replica quantum math
nodes_needed = num_primaries × (replicas + 1) / shards_per_node
```

| Situation | Fully-managed tier | Custom tier |
|---|---|---|
| index red | per-index SLO pages platform | invisible to platform — tenant's pager |
| data loss | restore from snapshot + backlog replay | replicas + re-ingest from source |
| scale reads | autoscaler adds a copy | human PR, whole-copy steps |
| nodeSet migration | widen `include.group`, single spanning autoscaler | add nodeSet + delete old = done |
| disk pressure | autoscaler outruns watermarks | watermarks: 85/90/95% → read-only |
