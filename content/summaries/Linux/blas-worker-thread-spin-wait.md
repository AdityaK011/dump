---
title: "Summary: Why a BLAS Worker Thread Doubled Our Inference CPU"
---

> **Full notes:** [[notes/Linux/blas-worker-thread-spin-wait|Why a BLAS Worker Thread Doubled Our Inference CPU -->]]

The mechanism in one line: a wider embedding (32 to 128) pushed `gemv` over OpenBLAS's single-thread threshold, so the worker pool woke on every call and then **yield-spun ~100 ms** waiting for more work. Requests arrived every 33 to 40 ms, so workers never slept. The result was a fixed ~1.1-core floor per pod, 70% system time, and an HPA pinned at max.

## Key Concepts

### Waiting costs
- Pure spin `while(!flag){}`: a full core, mostly **user** time.
- Yield spin `while(!flag) sched_yield();`: a full core, mostly **system** time (a syscall per iteration).
- Futex sleep (`pthread_cond_wait`): zero CPU, state `S`, a few to tens of µs to wake.
- Libraries spin to avoid wake latency. That's right for HPC loops and wrong for request servers.

### `sched_yield` semantics
- Gives the CPU away only if another task is runnable on the **same run queue** (CFS, and EEVDF from Linux 6.6).
- Idle CPU: returns immediately, no switch, loop again.
- Busy CPU: switch counted as **nonvoluntary** (the task was still runnable). Futex blocks count as voluntary.
- Signature: a full core per thread, high `stime`, ~0 voluntary switches.

### OpenBLAS pool
- Pool = `blas_cpu_number - 1` workers, created at library load.
- Size precedence: `OPENBLAS_NUM_THREADS` > `GOTO_NUM_THREADS` > `OMP_NUM_THREADS` > affinity-mask CPUs.
- Affinity mask = **all node CPUs** unless the pod is Guaranteed under the static CPU manager. Requests and quota are ignored.
- Env is read **once** in the constructor. Setting it after `import numpy` does nothing.
- Worker wait loop: `sched_yield` until `rdtsc` delta > `1 << THREAD_TIMEOUT` (default **28**), then `pthread_cond_wait`.
- 2^28 TSC ticks = ~110 ms at 2.45 GHz, ~89 ms at 3.0 GHz.
- `OPENBLAS_THREAD_TIMEOUT` is clamped to **4..30**.

### The threshold
- `gemv`: if `m*n < 2304 * GEMM_MULTITHREAD_THRESHOLD` (default 4, so **9,216**), 1 thread, else `num_cpu_avail()`.
- `gemm`: 1 thread if `m*n*k <= 65536 * threshold`.
- ~120 candidates: `120x32 = 3,840` runs single-threaded. `120x128 = 15,360` uses the pool (crosses at 72 candidates vs 288 before).
- Cost is a **step function** of request gap vs the ~100 ms window. Light-traffic environments won't reproduce it.

### Fixes
- `OPENBLAS_NUM_THREADS=1`: no pool. Best default for request-parallel servers, and small matrices aren't slower.
- `OPENBLAS_THREAD_TIMEOUT=4`: keeps the pool, workers sleep at once, a futex wake per call. Use it when something else owns the thread count.
- The trap: app code overwrote the variable before import. **Verify the effective thread count** (`ls /proc/<pid>/task | wc -l`, `threadpoolctl.threadpool_info()`).
- `os.cpu_count()` returns node CPUs. `os.process_cpu_count()` (3.13) respects affinity but not quota.

### Measuring
- CPU/request = `1000 * cores / rps`, computed as a **ratio of sums** at 60 s resolution.
- Fit `pod_cpu = F + c * pod_rps`. Floor up with flat slope means a background thread.
- `cpu.stat` `user_usec` vs `system_usec`. 70% sys in a compute workload means syscalls.
- `top -H`, `pidstat -t -u 1`, `pidstat -t -w 1`, `/proc/<pid>/task/*/stat` fields 14/15 (ticks, cumulative).
- `strace -f -c` (sched_yield dominates), `perf top` (`blas_thread_server`, `do_sched_yield`).

### Kubernetes amplification
- HPA `AverageValue` target T: settles only if `F + c*R/N <= T`, so it needs **F < T**. F = 1.4 > T, so pinned at `maxReplicas` (180).
- Total = `N*F + c*R`. Scale-out multiplies the floor: 180 x 1.4 = ~252 cores of waiting.
- p98 over 24 h = the top ~29 min. It rises after ~29 min and **falls after ~23.5 h**. Measure savings after the window rolls.
- No CPU limit: spin looks free but is charged to the pod. With a limit: quota burn, then throttling and latency.

### Dashboard artifacts
- Avg across containers mixes in sidecars and roughly halves per-pod CPU. Sum by pod or filter the container.
- Coarse rollups over churning pods: avg over-weights short-lived pods, sum counts ghost series.
- Avg of per-pod ratios is not a ratio of sums. Low-traffic pods dominate.
- Cumulative CPU counters need a rate.

## Quick Reference

```
 request gap < ~100 ms        workers never reach cond_wait
 m*n >= 9,216 (gemv)          pool used every call
 affinity = node CPUs         pool = node vCPUs - 1
 ------------------------------------------------------------
 all three  ->  fixed floor per pod, high stime, HPA at max
```

| Signal | Yield spin | Pure spin | Healthy idle pool |
|---|---|---|---|
| CPU per thread | ~1 core | ~1 core | ~0 |
| user vs sys | mostly sys | mostly user | n/a |
| voluntary switches | ~0 | ~0 | yes, on wake |
| state | R | R | S |

| Runtime | Spin knob | Default |
|---|---|---|
| OpenBLAS | `OPENBLAS_THREAD_TIMEOUT` | 2^28 cycles |
| Intel OpenMP / MKL | `KMP_BLOCKTIME` | 200 ms |
| libgomp | `OMP_WAIT_POLICY`, `GOMP_SPINCOUNT` | bounded spin |
| ONNX Runtime | `session.intra_op.allow_spinning` | on |

| | Before | After |
|---|---|---|
| Peak cores | ~580 | ~210 |
| CPU/request | ~59 ms | ~26 ms (pre-change ~27) |
| Floor/pod | ~1.4 | ~0.3 |
| Replicas | 180 (max) | ~90 |
