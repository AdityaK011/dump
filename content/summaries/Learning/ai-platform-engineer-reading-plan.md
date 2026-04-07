---
title: "Summary: AI Platform Engineer — 16-Week Reading Plan"
---

> **Full notes:** [[notes/Learning/ai-platform-engineer-reading-plan|AI Platform Engineer — 16-Week Reading Plan -->]]

## Key Concepts

### Month 1 -- Distributed Systems Foundations (Weeks 1-4)
- **Data Models & Storage:** DDIA Ch.1-3 covering reliability, data models, LSM-Trees, B-Trees; Uber Docstore case study
- **Replication & Consistency:** DDIA Ch.4-5 on encoding, leader-based and leaderless replication; Jepsen consistency models
- **Partitioning & Transactions:** DDIA Ch.6-7 on sharding, ACID, isolation levels, serializability; CockroachDB distributed transactions
- **Consensus & Fault Tolerance:** DDIA Ch.8-9 on failure modes, linearizability, consensus; Raft paper (Sections 1-5)
- **Video series:** MIT 6.824 lectures on distributed systems, RPC, and Raft

### Month 2 -- Data Pipelines, Batch & Stream Processing (Weeks 5-8)
- **Batch Processing:** DDIA Ch.10 + Google MapReduce paper; Netflix Spark ETL case study
- **Stream Processing:** DDIA Ch.11-12 on message brokers, stream joins; Kafka at LinkedIn, Flink at Uber
- **Networking & Load Balancing:** UDS (Vitillo) Ch.1-8 on TCP/UDP, TLS, DNS, APIs; Envoy load balancing blog; Discord message storage
- **Reliability & Observability:** Google SRE Book Ch.1, 4, 6, 11 on SLOs, monitoring, on-call; Airbnb observability

### Month 3 -- Machine Learning Systems & MLOps (Weeks 9-12)
- **ML Systems Overview:** DMLS Ch.1-3 on ML system design and data engineering; Uber Michelangelo, Meta AI infra
- **Feature Engineering & Training:** DMLS Ch.4-6 on training data, features, model development; Tecton feature stores, Spotify ML
- **Deployment & Serving:** DMLS Ch.7-8 on prediction services and distribution shifts; vLLM, Ray, OpenAI K8s scaling
- **MLOps & Evaluation:** DMLS Ch.9-11 on continual learning and tooling; Google Cloud MLOps pipelines, LLMOps

### Month 4 -- AI Infrastructure & Synthesis (Weeks 13-16)
- **GPU Orchestration:** GPU programming basics, LLM serving at scale (Anyscale), SkyPilot, Tim Dettmers GPU guide, Transformer inference paper
- **Caching, Queues & Cost:** KV-cache, LLM inference cost reduction, rate limiter design, Temporal durable execution, Cloudflare architecture
- **RAG & Vector DBs:** Pinecone RAG intro, vector DB benchmarks, Qdrant architecture, LlamaIndex RAG pipelines, AI gateway patterns
- **System Design Practice:** Five exercises -- ML serving platform, feature store, RAG at 10K qps, multi-tenant AI platform, A/B testing pipeline

## Quick Reference

### Books (in reading order)

| Abbr. | Book | Author | Weeks |
|-------|------|--------|-------|
| DDIA | Designing Data-Intensive Applications | Martin Kleppmann | 1-6 |
| UDS | Understanding Distributed Systems | Roberto Vitillo | 7 |
| SRE | Site Reliability Engineering | Google (free) | 8 |
| DMLS | Designing Machine Learning Systems | Chip Huyen | 9-12 |

### Weekly Themes at a Glance

| Week | Theme |
|------|-------|
| 1 | Data Models & Storage |
| 2 | Encoding, Replication & Consistency |
| 3 | Partitioning & Transactions |
| 4 | Consensus & Fault Tolerance |
| 5 | Batch Processing & MapReduce |
| 6 | Stream Processing |
| 7 | Networking & Load Balancing |
| 8 | Reliability & Observability |
| 9 | ML Systems Overview |
| 10 | Feature Engineering & Training |
| 11 | Model Deployment & Serving |
| 12 | MLOps & Evaluation |
| 13 | GPU Orchestration & Compute |
| 14 | Caching, Queues & Cost Optimization |
| 15 | RAG, Vector Databases & AI Pipelines |
| 16 | System Design Practice & Synthesis |

### Side Project Phases

| Weeks | Phase | Focus |
|-------|-------|-------|
| 1-3 | 1 | Deploy open-source LLM behind REST API, containerize |
| 4-6 | 2 | Add Kubernetes, autoscaling, model registry |
| 7-10 | 3 | Observability: Prometheus, Grafana, logging |
| 11-14 | 4 | Multi-model routing, caching, queuing |
| 15-16 | 5 | RAG pipeline, vector DB, or GPU resource mgmt |
