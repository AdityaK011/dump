---
title: "Kubernetes Autoscaling"
---

## What actually happens when a traffic surge hits your cluster

The gap between "HPA detected a spike" and "new pods are serving traffic" is where platform engineering lives. Understanding every source of delay is essential for designing systems that survive surges.

### The HPA control loop and scaling algorithm

The Horizontal Pod Autoscaler runs a control loop every **15 seconds** (configurable via `--horizontal-pod-autoscaler-sync-period`). It fetches metrics from the metrics-server (which scrapes kubelet every ~15s), then applies the formula: `desiredReplicas = ceil[currentReplicas × (currentMetricValue / desiredMetricValue)]`. For multiple metrics, HPA calculates desired replicas for each and takes the maximum.

The `behavior` field in `autoscaling/v2` allows independent configuration of scale-up and scale-down policies. Scale-up has a default stabilization window of **0 seconds** (immediate), while scale-down defaults to **300 seconds** (5 minutes), looking at all recommendations in the window and picking the highest. You can configure policies by Pods (fixed number per period) or Percent (percentage of current replicas), and use `selectPolicy: Max` for aggressive scaling or `Min` for conservative behavior. GKE's Performance HPA profile (v1.31+) supports up to 5,000 HPA objects with parallel processing.

### The VPA: right-sizing pods without horizontal scaling

The Vertical Pod Autoscaler has three components that form a pipeline. The **Recommender** continuously analyzes current and historical resource consumption, building an exponential histogram model that produces four values per container: target, lowerBound, upperBound, and uncappedTarget. The **Updater** watches running pods and evicts those whose requests differ significantly from recommendations (respecting PodDisruptionBudgets via the Eviction API). The **Admission Controller** is a mutating webhook that intercepts pod creation and overrides resource requests with the Recommender's target values.

VPA operates in four modes: **Off** (recommendations only), **Initial** (sets requests at creation only), **Recreate** (evicts and recreates pods), and **InPlaceOrRecreate** (attempts in-place resize without restart, GA in Kubernetes 1.35). The critical constraint is that **VPA and HPA must not target the same metrics** — doing so creates a feedback loop where VPA shrinks resources → CPU spikes → HPA scales out → VPA sees lower utilization → shrinks again. GKE's Multidimensional Pod Autoscaling (MPA) solves this by coordinating horizontal scaling on CPU with vertical scaling on memory in a single controller.

### Minute-by-minute anatomy of a traffic surge

**T+0s**: Traffic surge begins. Existing pods absorb the load. Latency increases, errors may start appearing.

**T+0–30s**: Metrics collection delay. The metrics-server scrapes kubelet every ~15s. HPA checks metrics every 15s. Worst case: **~30 seconds** before HPA sees the surge (15s metrics lag + 15s HPA poll alignment).

**T+15–30s**: HPA computes new desired replica count, patches the Deployment's scale subresource, and the ReplicaSet controller creates new Pod objects.

**T+30–35s**: If capacity exists on current nodes, the scheduler binds pods in seconds. If not, pods enter `Pending` with `Unschedulable` condition.

**T+35s–5+ minutes** (if node scaling needed): The Cluster Autoscaler scans every 10 seconds for unschedulable pods. Decision takes under 30 seconds. GCE VM provisioning typically takes **3–4 minutes**. Total from CA trigger to pods schedulable: approximately 5 minutes.

**After scheduling**: Container image pull (seconds if cached, 10–120s if not — GKE Image Streaming can reduce a 5.4GB image from 191s to 30s). Application startup and readiness probe passage. NEG controller adds pod to NEG. GCP LB health check passes (default: 5s interval × 2 healthy threshold = **10 seconds minimum**).

**Total end-to-end**: Best case with existing capacity: **30–60 seconds**. Typical case with image pull: **1–2 minutes**. New node needed: **4–7 minutes**. New node pool via NAP: **5–8 minutes**.

### Why there is always a delay and how to compress it

Every layer contributes to the delay: metrics collection latency (up to 30s), HPA decision time (sub-second), scheduler binding (seconds), node provisioning (3–5 minutes), image pull (seconds to minutes), application startup (variable), readiness probe (depends on configuration), NEG registration (seconds), and LB health check (10–30s). The **single largest contributor** is node provisioning when existing capacity is exhausted.

The most effective mitigation is **overprovisioning with low-priority pause pods**. Deploy a set of `registry.k8s.io/pause:3.6` containers with requests matching your workload profile (e.g., 1 CPU, 2Gi memory) at priority `-1000`. When real workload pods need scheduling, pause pods are preempted instantly — no waiting for node provisioning. The preempted pause pods then trigger the Cluster Autoscaler to add nodes for future capacity. This pattern effectively eliminates the 3–5 minute node provisioning delay from the critical path.

### Compute Classes

GKE **Compute Classes** (ComputeClass CRD) provide priority-ordered node provisioning rules. You can specify preferred machine families with automatic fallback — for example, prefer Spot e2 instances, fall back to on-demand e2 if unavailable. Compute Classes support active migration (workloads automatically move to higher-priority nodes as availability allows) and per-class autoscaling policies.

### KEDA: event-driven scaling beyond resource metrics

KEDA (CNCF graduated project) extends HPA with **70+ event-driven triggers** including Prometheus queries, Pub/Sub message backlog, Kafka consumer lag, and cron schedules. Its killer feature is **scale-to-zero**: KEDA's operator handles 0→1 and 1→0 transitions directly (HPA cannot natively scale to zero), then delegates 1→N scaling to a standard HPA it creates and manages. A ScaledObject CRD maps event sources to workloads with configurable polling intervals and cooldown periods.

## Related notes

- [[notes/K8s/service-mesh-multi-cluster-and-advanced-patterns|Service Mesh, Multi-Cluster, and Advanced Patterns]]
- [[notes/K8s/gke-networking-and-load-balancing|GKE Networking and Load Balancing]]
- [[notes/K8s/kubernetes-networking-internals|Kubernetes Networking Internals]]
- [[notes/K8s/gke-ingress-and-gateway-api|GKE Ingress and Gateway API]]
