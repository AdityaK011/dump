---
title: Elasticsearch
---

Notes on running Elasticsearch as a platform — cluster anatomy on Kubernetes, the ingestion write path, tenancy models, and the ES internals underneath them.

## Platform anatomy
- [[notes/Elasticsearch/elasticsearch-as-a-service-on-kubernetes|Elasticsearch as a Service on Kubernetes]] — Anatomy of a multi-tenant search platform: operator vs platform ownership, attribute-based shard pinning, why storage is the rigid layer, the index lifecycle state machine, replica-copy autoscaling quantisation, why node-level disk metrics can't attribute load to a tenant, and the economics of unbundled IOPS billing

## The write path
- [[notes/Elasticsearch/from-subscription-to-shard|From Subscription to Shard]] — The write path of a managed search platform: filtered Pub/Sub subscriptions, self-submitting Dataflow feeders, external versioning as the glue for at-least-once delivery and snapshot∪backlog bootstrap, who does what in a restore (coordinating holds connections, master holds state, data nodes hold bytes), and a production race where the autoscaler killed the pod holding a `wait_for_completion` connection — twice

## Tenancy models
- [[notes/Elasticsearch/bring-your-own-index|Bring Your Own Index]] — The custom-tenant tier of a managed search platform: the ownership slider, what the platform stops rendering (no feeders, snapshots, autoscalers, or per-index SLOs), reachability-as-credential access, mesh config that executes in the caller's sidecar, the ES internals each tier exposes, and a two-PR architecture + storage migration that worked precisely because nothing was pinned
