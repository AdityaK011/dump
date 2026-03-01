# AI Platform Engineer — 16-Week Serialized Reading Plan

> **How to use this:** Follow one reading per day, Monday through Friday. Each week has a theme. Readings are sequenced so that concepts build on each other — don't skip ahead. Check off items as you go. Weekends are for the side project.

---

## MONTH 1: Distributed Systems Foundations

### Week 1 — Data Models & Storage
- **Mon:** DDIA Ch.1 — Reliable, Scalable, and Maintainable Applications
- **Tue:** DDIA Ch.2 — Data Models and Query Languages
- **Wed:** DDIA Ch.3 — Storage and Retrieval (Part 1: up to LSM-Trees)
- **Thu:** DDIA Ch.3 — Storage and Retrieval (Part 2: B-Trees onward)
- **Fri:** Blog — [How Uber Serves Over 40M Reads Per Second from Online Storage](https://www.uber.com/blog/how-uber-serves-over-40-million-reads-per-second-using-an-integrated-cache/) (search "Uber Docstore" if link breaks)

### Week 2 — Encoding, Replication & Consistency
- **Mon:** DDIA Ch.4 — Encoding and Evolution
- **Tue:** DDIA Ch.5 — Replication (Part 1: Leader-based replication)
- **Wed:** DDIA Ch.5 — Replication (Part 2: Multi-leader and leaderless)
- **Thu:** Blog — [Consistency Models by Jepsen](https://jepsen.io/consistency) (read the overview page)
- **Fri:** Video — MIT 6.824 Lecture 1: Introduction to Distributed Systems (YouTube)

### Week 3 — Partitioning & Transactions
- **Mon:** DDIA Ch.6 — Partitioning
- **Tue:** DDIA Ch.7 — Transactions (Part 1: ACID and isolation levels)
- **Wed:** DDIA Ch.7 — Transactions (Part 2: Serializability)
- **Thu:** Blog — [How CockroachDB Does Distributed Transactions](https://www.cockroachlabs.com/blog/how-cockroachdb-distributes-atomic-transactions/)
- **Fri:** Video — MIT 6.824 Lecture 2: RPC and Threads

### Week 4 — Consensus & Fault Tolerance
- **Mon:** DDIA Ch.8 — The Trouble with Distributed Systems
- **Tue:** DDIA Ch.9 — Consistency and Consensus (Part 1: Linearizability)
- **Wed:** DDIA Ch.9 — Consistency and Consensus (Part 2: Consensus algorithms)
- **Thu:** Paper — [In Search of an Understandable Consensus Algorithm (Raft)](https://raft.github.io/raft.pdf) — read Sections 1–5
- **Fri:** Video — MIT 6.824 Lecture on Raft (Lecture 5 or 6, check playlist)

---

## MONTH 2: Data Pipelines, Batch & Stream Processing

### Week 5 — Batch Processing & MapReduce
- **Mon:** DDIA Ch.10 — Batch Processing (Part 1: Unix philosophy & MapReduce)
- **Tue:** DDIA Ch.10 — Batch Processing (Part 2: Beyond MapReduce)
- **Wed:** Paper — [MapReduce: Simplified Data Processing on Large Clusters](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf) — Google's original paper
- **Thu:** Blog — [How Netflix Uses Spark for ETL](https://netflixtechblog.com/notebook-innovation-591ee3221233) (search Netflix + Spark if link breaks)
- **Fri:** Video — MIT 6.824 Lecture on MapReduce

### Week 6 — Stream Processing
- **Mon:** DDIA Ch.11 — Stream Processing (Part 1: Message brokers)
- **Tue:** DDIA Ch.11 — Stream Processing (Part 2: Stream joins and processing)
- **Wed:** DDIA Ch.12 — The Future of Data Systems
- **Thu:** Blog — [Kafka at LinkedIn: How It All Began](https://engineering.linkedin.com/blog/topic/kafka)
- **Fri:** Blog — [Flink at Uber: Streaming Platform Overview](https://www.uber.com/blog/journey-with-apache-kafka/) (search "Uber streaming platform" if link breaks)

### Week 7 — Networking & Load Balancing
- **Mon:** UDS Ch.1–3 — Communication basics, TCP/UDP, and TLS (*Understanding Distributed Systems* by Vitillo)
- **Tue:** UDS Ch.4–5 — DNS and APIs
- **Wed:** Blog — [Introduction to Modern Load Balancing](https://blog.envoyproxy.io/introduction-to-modern-network-load-balancing-and-proxying-a57f6ff80236) by Matt Klein (Envoy creator)
- **Thu:** UDS Ch.7–8 — Replication and Coordination (Vitillo's concise treatment)
- **Fri:** Blog — [How Discord Stores Trillions of Messages](https://discord.com/blog/how-discord-stores-trillions-of-messages)

### Week 8 — Reliability & Observability
- **Mon:** Google SRE Book Ch.1 — Introduction (free: sre.google/sre-book)
- **Tue:** Google SRE Book Ch.4 — Service Level Objectives
- **Wed:** Google SRE Book Ch.6 — Monitoring Distributed Systems
- **Thu:** Blog — [Observability at Airbnb](https://medium.com/airbnb-engineering/monitoring-at-airbnb-c428fe9aaa79) (search "Airbnb observability" if link breaks)
- **Fri:** Google SRE Book Ch.11 — Being On-Call

---

## MONTH 3: Machine Learning Systems & MLOps

### Week 9 — ML Systems Overview
- **Mon:** DMLS Ch.1 — Overview of Machine Learning Systems (*Designing Machine Learning Systems* by Chip Huyen)
- **Tue:** DMLS Ch.2 — Introduction to ML Systems Design
- **Wed:** DMLS Ch.3 — Data Engineering Fundamentals
- **Thu:** Blog — [Uber Michelangelo: ML Platform](https://www.uber.com/blog/michelangelo-machine-learning-platform/) (search "Uber Michelangelo" if link breaks)
- **Fri:** Blog — [Meta's AI Infrastructure for Large-Scale Training](https://engineering.fb.com/category/ml-applications/)

### Week 10 — Feature Engineering & Training
- **Mon:** DMLS Ch.4 — Training Data
- **Tue:** DMLS Ch.5 — Feature Engineering
- **Wed:** DMLS Ch.6 — Model Development and Training
- **Thu:** Blog — [Feature Stores: A Complete Guide](https://www.tecton.ai/blog/what-is-a-feature-store/) by Tecton
- **Fri:** Blog — [How Spotify Uses ML](https://engineering.atspotify.com/category/machine-learning/) — pick one recent post

### Week 11 — Model Deployment & Serving
- **Mon:** DMLS Ch.7 — Model Deployment and Prediction Service
- **Tue:** Blog — [vLLM: Easy, Fast, and Cheap LLM Serving](https://blog.vllm.ai/) — read the introductory blog post
- **Wed:** Blog — [Scaling Kubernetes for ML at OpenAI](https://openai.com/research) (search "OpenAI Kubernetes scaling" if link breaks)
- **Thu:** DMLS Ch.8 — Data Distribution Shifts and Monitoring
- **Fri:** Blog — [Ray: A Distributed Framework for Emerging AI Applications](https://docs.ray.io/en/latest/ray-overview/index.html) — read the architecture overview

### Week 12 — MLOps & Evaluation
- **Mon:** DMLS Ch.9 — Continual Learning and Test in Production
- **Tue:** DMLS Ch.10 — Infrastructure and Tooling for MLOps
- **Wed:** DMLS Ch.11 — The Human Side of Machine Learning
- **Thu:** Blog — [MLOps: Continuous Delivery for ML](https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning) by Google Cloud
- **Fri:** Blog — [LLMOps: What's Different About Deploying LLMs](https://huyenchip.com/2023/10/10/multimodal.html) (search "Chip Huyen LLMOps" for latest)

---

## MONTH 4: AI Infrastructure, GPU Systems & Putting It All Together

### Week 13 — GPU Orchestration & Compute
- **Mon:** Blog — [A Beginner's Guide to GPU Programming](https://modal.com/blog) (search "Modal GPU guide")
- **Tue:** Blog — [How to Serve LLMs at Scale](https://www.anyscale.com/blog) — Anyscale/Ray blog on LLM serving
- **Wed:** Docs — [SkyPilot: Run AI on Any Cloud](https://skypilot.readthedocs.io/) — read the overview and quickstart
- **Thu:** Blog — [GPU Clusters for Distributed Training](https://timdettmers.com/2023/01/30/which-gpu-for-deep-learning/) by Tim Dettmers
- **Fri:** Paper — [Efficiently Scaling Transformer Inference](https://arxiv.org/abs/2211.05102) — read abstract + sections 1–3

### Week 14 — Caching, Queues & Cost Optimization
- **Mon:** Blog — [Prompt Caching and KV-Cache Explained](https://developer.nvidia.com/blog/) (search "NVIDIA KV cache")
- **Tue:** Blog — [How to Reduce LLM Inference Costs](https://www.semianalysis.com/) — search for inference cost analysis
- **Wed:** Blog — [Designing a Rate Limiter](https://blog.bytebytego.com/) — Alex Xu's system design breakdown
- **Thu:** Blog — [Temporal: Durable Execution Explained](https://docs.temporal.io/concepts) — read core concepts
- **Fri:** Blog — [How Cloudflare Handles Millions of Requests](https://blog.cloudflare.com/) — pick a recent architecture post

### Week 15 — RAG, Vector Databases & AI Pipelines
- **Mon:** Blog — [What is Retrieval-Augmented Generation?](https://www.pinecone.io/learn/retrieval-augmented-generation/) by Pinecone
- **Tue:** Blog — [Vector Database Comparison](https://benchmark.vectorview.ai/) — read benchmarks and trade-offs
- **Wed:** Docs — [Qdrant: Vector Search Engine](https://qdrant.tech/documentation/) — architecture overview
- **Thu:** Blog — [Building Production RAG Pipelines](https://www.llamaindex.ai/blog) — pick a recent post on RAG architecture
- **Fri:** Blog — [AI Gateway Patterns](https://www.anthropic.com/research) or search "AI gateway architecture patterns"

### Week 16 — System Design Practice & Synthesis
- **Mon:** Exercise — Design an ML model serving platform (write it out on paper/doc)
- **Tue:** Exercise — Design a feature store with real-time and batch capabilities
- **Wed:** Exercise — Design a RAG system that serves 10K queries/sec
- **Thu:** Exercise — Design a multi-tenant AI platform with cost attribution
- **Fri:** Exercise — Design a model evaluation and A/B testing pipeline

---

## Side Project Schedule (Weekends)

| Weeks   | Phase | Focus |
|---------|-------|-------|
| 1–3     | 1     | Deploy an open-source LLM behind a REST API, containerize |
| 4–6     | 2     | Add Kubernetes, autoscaling, model registry |
| 7–10    | 3     | Observability: Prometheus, Grafana, logging |
| 11–14   | 4     | Multi-model routing, caching, queuing |
| 15–16   | 5     | RAG pipeline, vector DB, or GPU resource mgmt |

---

## Book List (in reading order)

| Abbr. | Book | Author | When |
|-------|------|--------|------|
| DDIA  | Designing Data-Intensive Applications | Martin Kleppmann | Weeks 1–6 |
| UDS   | Understanding Distributed Systems | Roberto Vitillo | Week 7 |
| SRE   | Site Reliability Engineering | Google (free online) | Week 8 |
| DMLS  | Designing Machine Learning Systems | Chip Huyen | Weeks 9–12 |

---

## Tips for Sticking With It

1. **Read on your commute or before bed** — most readings are 20–40 min
2. **Take one-line notes** — after each reading, write a single sentence summary in your notes app
3. **Connect to the project** — ask yourself "how does today's reading apply to what I'm building?"
4. **Don't stress about missed days** — just pick up where you left off, don't restart the week
5. **Share what you learn** — post a short take on LinkedIn or a dev community once a week

---

*Last updated: March 2026. Revisit and adjust after completing the 16 weeks.*
