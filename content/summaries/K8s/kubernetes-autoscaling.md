---
title: "Summary: Kubernetes Autoscaling"
---

> **Full notes:** [[notes/K8s/kubernetes-autoscaling|Kubernetes Autoscaling -->]]

## Key Concepts

### HPA Control Loop
- Runs every **15 seconds** (`--horizontal-pod-autoscaler-sync-period`)
- Formula: `desiredReplicas = ceil[currentReplicas x (currentMetric / desiredMetric)]`
- Multiple metrics: calculates per-metric, takes the **maximum**
- Scale-up stabilization: **0s** (immediate); scale-down: **300s** (5 min)
- Policies by `Pods` (fixed count) or `Percent` (of current replicas)
- GKE Performance HPA: up to **5,000 HPA objects** with parallel processing

### VPA (Vertical Pod Autoscaler)
- Three components: **Recommender** (histogram model) -> **Updater** (evicts pods) -> **Admission Controller** (mutating webhook overrides requests)
- Modes: Off (recommend only), Initial (creation only), Recreate (evict+recreate), InPlaceOrRecreate (GA in K8s 1.35)
- **Never target same metrics as HPA** -- creates feedback loop
- GKE MPA: coordinates HPA on CPU + VPA on memory in one controller

### Traffic Surge Timeline
- **T+0-30s**: Metrics lag (15s scrape + 15s HPA poll worst case)
- **T+30-35s**: Scheduler binds pods (if node capacity exists)
- **T+35s-5min**: If no capacity -- Cluster Autoscaler scans every 10s, GCE VM provisioning takes **3-4 min**
- **After scheduling**: Image pull (seconds if cached, 10-120s if not), app startup, readiness probe, NEG registration, LB health check (**10s minimum**: 5s interval x 2 threshold)

### End-to-End Scaling Latency
- Best case (existing capacity): **30-60 seconds**
- Typical (image pull needed): **1-2 minutes**
- New node needed: **4-7 minutes**
- New node pool (NAP): **5-8 minutes**

### Overprovisioning Pattern (Key Mitigation)
- Deploy `pause:3.6` containers at priority **-1000** matching workload resource profile
- Real pods preempt pause pods instantly -- no node provisioning wait
- Preempted pause pods then trigger Cluster Autoscaler for future capacity
- Eliminates the 3-5 min node provisioning from the critical path

### Compute Classes
- ComputeClass CRD: priority-ordered machine family provisioning with automatic fallback
- Supports active migration to higher-priority nodes as availability returns

### KEDA (Event-Driven Scaling)
- **70+ triggers**: Prometheus, Pub/Sub, Kafka lag, cron, etc.
- Killer feature: **scale-to-zero** (HPA cannot natively do 0 replicas)
- KEDA handles 0->1 and 1->0, delegates 1->N to standard HPA

## Quick Reference

```
Scaling Timeline:
  T+0s     Traffic spike begins
  T+30s    HPA detects (worst case)
  T+35s    Pods scheduled (if capacity exists)
  T+5min   New node ready (if needed)
  T+5-7min Pod serving traffic (image pull + health check)

  Biggest delay: NODE PROVISIONING (3-5 min)
  Fix: Pause pod overprovisioning

HPA Formula:
  desired = ceil(current x (actualMetric / targetMetric))

VPA + HPA Danger:
  VPA shrinks resources -> CPU spikes -> HPA scales out
  -> VPA sees lower util -> shrinks again (feedback loop!)
  Solution: GKE MPA or different metrics

KEDA Scaling Range:
  KEDA: 0 <-> 1 (operator)
  HPA:  1 <-> N (delegated)
```
