---
title: "Why a BLAS Worker Thread Doubled Our Inference CPU"
---

## The short version

A CPU inference service shipped a new model version and its CPU cost per request roughly doubled overnight. Nothing in the request path looked heavier. The model was the same architecture with a wider embedding (dimension 32 went to 128), latency was fine, and the code hadn't changed.

The extra CPU was not doing inference. It was OpenBLAS worker threads that had finished their share of a matrix-vector multiply and were waiting for the next one by calling `sched_yield()` in a tight loop. They're allowed to wait like that for about 2^28 CPU cycles (roughly 100 ms) before going to sleep. Requests reached each model instance every 33 to 40 ms, so the workers never got as far as sleeping. They spun all day, and 70% of the container's CPU time was kernel time spent entering and leaving `sched_yield`.

The wider embedding mattered because OpenBLAS only uses its thread pool for a matrix-vector multiply when `m * n` is at least `2304 * GEMM_MULTITHREAD_THRESHOLD` (9,216 with the default build). The old matrices were under that line and ran single-threaded, so the pool never woke up. The new ones were over it.

Then Kubernetes amplified it. The HPA targeted an absolute CPU value per pod, the spin put a fixed floor under every pod that was higher than the target, and so the deployment sat at `maxReplicas` all day. A rightsizing tool sized requests from the p98 of the inflated usage.

Fixing it took one environment variable, once we'd found the place in the code that was overwriting it. Results:

| | Before | After |
|---|---|---|
| Peak cores used | ~580 | ~210 |
| CPU per request | ~59 ms | ~26 ms |
| Daytime pod count | 180 (pinned at max) | ~90 |
| Per-pod fixed CPU floor | ~1.4 cores | ~0.3 cores |
| Kernel share of CPU | ~70% | small |

The rest of this note goes through the Linux mechanics underneath each step: threads, `sched_yield`, spin versus futex sleep, how cgroups account CPU, how to measure all of it, and why the autoscaler and the dashboards kept hiding the problem.

---

## Part 1: the Linux pieces you need

### Threads are tasks, and the scheduler only sees tasks

On Linux a thread is a task created by `clone()` with flags that share the address space (`CLONE_VM`, `CLONE_THREAD`, and friends). The scheduler doesn't know about "processes" as units of work. It schedules tasks, and each has its own run state, its own CPU time counters, and its own context switch counters.

That's why "the process uses 3 cores" is the wrong level of detail. A Python model server with a BLAS library loaded might have 50 or 60 threads, and the useful questions are which threads are burning CPU and whether that time is user or kernel.

```
 /proc/<pid>/task/<tid>/stat      utime (field 14), stime (field 15), in clock ticks
 /proc/<pid>/task/<tid>/status    voluntary_ctxt_switches, nonvoluntary_ctxt_switches
 /proc/<pid>/task/<tid>/comm      thread name (15 chars max)
```

OpenBLAS doesn't name its workers, so they show up with the parent's name (`python3` or similar). You recognise them by count: one fewer than the number of CPUs the library thinks it has.

### Three ways to wait

A thread that has nothing to do can wait in three ways, and they cost very different amounts of CPU:

```
 1. pure spin        while (!flag) { }                     user time, 100% of a core
 2. yield spin       while (!flag) { sched_yield(); }      mostly KERNEL time, 100% of a core
                                                           if nothing else wants the CPU
 3. sleep            futex(FUTEX_WAIT) / pthread_cond_wait  zero CPU, thread is in state S
```

The reason anyone spins at all is wake-up latency. Waking a sleeping thread means another thread calls `futex(FUTEX_WAKE)`, the kernel moves the sleeper onto a run queue, and the scheduler eventually runs it. That costs somewhere between a few microseconds and tens of microseconds, more if the target CPU is in a deep idle state. If you're a numerical library running thousands of back-to-back matrix operations in a tight HPC loop, paying that on every call adds up, so you keep workers hot for a while after each job.

That trade is sound in a benchmark loop. It's a bad trade in a request server where the next job arrives tens of milliseconds later, because you pay the full spin and still don't save anything meaningful.

### What `sched_yield()` actually does

`sched_yield()` asks the scheduler to run something else on this CPU if something else is runnable. In CFS (and EEVDF, which replaced CFS as the fair scheduler in Linux 6.6) a yield marks the current task so that the scheduler prefers another runnable task on the same run queue for the next pick. It does not put the task to sleep and it doesn't move it to the back of some global line.

So the result depends on what else is on that CPU:

```
 CPU has other runnable tasks       yield gives the CPU away briefly, counted as an
                                    involuntary context switch, then the spinner
                                    comes straight back

 CPU has nothing else runnable      yield returns almost immediately, no context switch,
                                    the loop goes round again
```

Either way the thread stays in state `R` and consumes a full core of CPU time. The only thing that changes is how that time is split. Each iteration is a syscall: entry, a pass through `schedule()`, exit. With modern speculative execution mitigations on the syscall path, that's hundreds of nanoseconds of kernel work per iteration against a handful of user-mode instructions (read the TSC, compare, loop). That's why a yield spin shows up as mostly `stime`, and why our container was at 70% system time.

One subtlety about the context switch counters: a switch caused by `sched_yield` is counted as nonvoluntary, because the task was still runnable when it gave up the CPU. Voluntary switches are counted when a task blocks (a futex wait, a read on an empty socket). So a healthy idle thread pool racks up voluntary switches and almost no CPU, and a yield-spinning pool racks up nonvoluntary switches (on a busy node) or nearly none at all (on an idle one), with a full core each.

### How the cgroup accounts it

A container is a cgroup, and the kernel charges every nanosecond a task runs to its cgroup. On cgroup v2 you can read it directly:

```
 $ cat /sys/fs/cgroup/cpu.stat
 usage_usec      ...     <- what metrics-server, cAdvisor and every dashboard reports
 user_usec       ...
 system_usec     ...
 nr_periods      ...
 nr_throttled    ...     <- only meaningful if cpu.max sets a quota
 throttled_usec  ...
```

The accounting doesn't care whether the time was useful. A spinning BLAS worker is charged exactly like a thread doing a matrix multiply. That's the whole problem in one sentence: the scheduler, the cgroup, the kubelet's metrics, and the HPA all see "this pod uses 1.4 cores more than it used to", and none of them can tell that the extra 1.4 cores is a loop waiting for work.

What happens next depends on whether the pod has a CPU limit:

```
 no CPU limit (requests only)    spinners take idle cores on the node. It looks "free",
                                 but it's charged to the pod, so autoscaling and
                                 rightsizing react to it.

 CPU limit (cpu.max quota)       spinners eat quota. Once the pod hits its quota for
                                 the 100 ms CFS period, the whole cgroup is throttled,
                                 including the thread doing real inference. You get
                                 latency spikes, not just cost.
```

Our pods had no CPU limit, so the cost showed up as cost and the latency stayed fine. That's part of why it went unnoticed.

---

## Part 2: OpenBLAS's thread pool

### Where the threads come from

When OpenBLAS is built with threading (the pthreads build, which is what NumPy and SciPy wheels ship), it creates a pool of worker threads when the library loads. The pool size is `blas_cpu_number - 1`, because the calling thread does a share of the work itself.

`blas_cpu_number` comes from the first of these that is set:

```
 OPENBLAS_NUM_THREADS   >   GOTO_NUM_THREADS   >   OMP_NUM_THREADS   >   detected CPUs
```

Detected CPUs means the CPUs in the process's affinity mask (`sched_getaffinity`), capped at the build-time maximum. It does not mean the cgroup CPU quota or the pod's CPU request. In a Kubernetes pod without exclusive CPUs (anything that isn't a Guaranteed pod under the static CPU manager policy), the affinity mask is every CPU on the node. A pod that requests 2 cores on a 48-vCPU node gets a 47-thread BLAS pool.

The environment variables are read once, in the library's load-time constructor. Setting `os.environ["OPENBLAS_NUM_THREADS"]` after `import numpy` does nothing to the pool that already exists. Runtime changes have to go through `openblas_set_num_threads()` or a wrapper like `threadpoolctl`.

### The worker's wait loop

This is the heart of the issue, from `driver/others/blas_server.c`. Simplified:

```c
/* default build: THREAD_TIMEOUT = 28, so the window is 2^28 TSC ticks */
static unsigned int thread_timeout = (1U << THREAD_TIMEOUT);

static void *blas_thread_server(void *arg) {
    ...
    for (;;) {
        last_tick = (unsigned int)rpcc();              /* rdtsc */

        while (!thread_status[cpu].queue) {            /* no job yet */
            YIELDING;                                  /* sched_yield() on x86-64 Linux */

            if ((unsigned int)rpcc() - last_tick > thread_timeout) {
                pthread_mutex_lock(&thread_status[cpu].lock);
                if (!thread_status[cpu].queue) {
                    thread_status[cpu].status = THREAD_STATUS_SLEEP;
                    while (thread_status[cpu].status == THREAD_STATUS_SLEEP)
                        pthread_cond_wait(&thread_status[cpu].wakeup,
                                          &thread_status[cpu].lock);   /* futex sleep */
                }
                pthread_mutex_unlock(&thread_status[cpu].lock);
                last_tick = (unsigned int)rpcc();
            }
        }

        /* run the job, signal completion, loop back and wait again */
    }
}
```

After each job, a worker yield-spins until either a new job lands in its queue or 2^28 TSC ticks pass, and only then does it go to sleep on a condition variable.

`rpcc()` reads the TSC, which on current x86 ticks at a constant nominal rate whatever the actual clock frequency is. 2^28 is 268,435,456 ticks:

```
 TSC at 2.45 GHz  ->  ~110 ms
 TSC at 3.0 GHz   ->   ~89 ms
```

So "about 100 ms" is the right mental model, and it varies a little with the host CPU.

The environment variable `OPENBLAS_THREAD_TIMEOUT` overrides the exponent. The library clamps it to the range 4 to 30, so the shortest possible window is 2^4 = 16 ticks, which is effectively "sleep immediately".

### The timeline that never reaches the sleep branch

```
 time ->         0 ms           35 ms          70 ms          105 ms
 requests        |R1            |R2            |R3            |R4
                 v              v              v              v
 caller thread   [gemv]......   [gemv]......   [gemv]......   [gemv]
 worker 1        [part]~~~~~~~~~[part]~~~~~~~~~[part]~~~~~~~~~[part]~~~~
 worker 2        [part]~~~~~~~~~[part]~~~~~~~~~[part]~~~~~~~~~[part]~~~~
 ...
                       ~~~~ = sched_yield spin (state R, full core, mostly stime)

 spin window:    |<------------------ ~100 ms ------------------>|
                 never expires, because the next job arrives at ~35 ms
```

With jobs every 33 to 40 ms, each worker resets its timer long before the 100 ms window expires. The `pthread_cond_wait` branch never runs. Every worker in the pool burns a core around the clock for as long as traffic keeps arriving faster than one request every ~100 ms per model instance.

Drop the traffic below that and the cost disappears suddenly. At night, with longer gaps, the workers reached the sleep branch and the pods went quiet. That's also why a staging environment with light traffic would never show it: the bug needs traffic, and the cost is a step function of inter-arrival time, not a smooth function of load.

```
 per-pod spin cost
      ^
 full |__________________________
      |                          |
      |                          |
      |                          |__________________
    0 +--------------------------+------------------>  gap between requests
      0                       ~100 ms                   to one model instance
```

### Why an embedding change crossed the line

OpenBLAS doesn't parallelise every call. For a single-vector matrix multiply (`gemv`), which is what scoring one query vector against a list of candidate embeddings turns into (`candidates @ query`, the core of a cosine-similarity feature), the interface code decides like this (`interface/gemv.c`, simplified):

```c
#ifdef SMP
    if (1L * m * n < 2304L * GEMM_MULTITHREAD_THRESHOLD)   /* default threshold = 4 */
        nthreads = 1;                                      /* do it on the calling thread */
    else
        nthreads = num_cpu_avail(2);                       /* use the pool */
#endif
```

`GEMM_MULTITHREAD_THRESHOLD` is a build option with a default of 4, so the line sits at `m * n = 9,216`. `num_cpu_avail()` returns `blas_cpu_number` (or 1 if the library has been told to use one thread, or if it's being called from inside an OpenMP parallel region in an OpenMP build).

Now take a ranking request that scores ~120 candidates, so the matrix is `candidates x embedding_dim`:

```
 embedding_dim = 32     m * n = 120 * 32  =  3,840   <  9,216   ->  1 thread, pool never wakes
 embedding_dim = 128    m * n = 120 * 128 = 15,360   >= 9,216   ->  pool wakes on every call
```

Put differently, with 32-dim embeddings a request needed 288+ candidates to cross the line, which almost never happened. With 128-dim it crosses at 72, which is below a typical page. The shape of the failure is general. A model change that looks like "a bit more arithmetic" also flips the library from the single-threaded path to the multithreaded path. The new path does the multiply slightly faster in wall-clock terms and then leaves the whole pool spinning for 100 ms afterwards.

Matrix-matrix multiplies (`gemm`, what you get with batch sizes above 1) have their own threshold on `m * n * k` (65,536 times the same build constant), so batching changes which side of the line you're on too.

Small matrices are also exactly where threading helps least. A 120 x 128 single-precision `gemv` is 60 KiB of data and about 31 thousand floating point operations, a few microseconds on one core. Splitting it across even 2 threads (the cap in the incident) saves very little, and across 47 even less, even before you count the wake-up and the spin.

---

## Part 3: how to measure it

### Step 1: CPU per request, not CPU

Total CPU goes up whenever traffic goes up, so it can't tell you whether something got worse. Divide it out:

```
 CPU per request (ms) = 1000 * (cores used) / (requests per second)

 before the model change    ~27 ms
 after the model change     ~58-59 ms
 after the fix              ~26 ms
```

Compute it as a ratio of sums over the whole fleet: total cores divided by total requests per second, at a fine time resolution. Averaging per-pod ratios weights an idle pod the same as a busy one, and the spin floor makes idle pods look terrible.

### Step 2: split per-pod CPU into a floor and a slope

For one pod, plot CPU against that pod's request rate and fit a line:

```
 pod_cpu  =  F  +  c * pod_rps

 F   fixed floor: CPU the pod uses regardless of load (runtime, sidecars, ... and spinners)
 c   marginal CPU per request
```

For this service the floor went from about 0.3 cores to about 1.4 cores per pod, while the slope barely moved. That's the fingerprint of something load-independent. Real inference work scales with requests. A spin that's always on, as long as requests keep coming, does not.

This is the most useful single step in the whole investigation, because it turns "the model is more expensive" (which nobody can disprove) into "each pod has a new 1.1-core fixed cost" (which points straight at a background thread).

### Step 3: user versus system time

```bash
# cgroup totals from inside the container (cgroup v2)
cat /sys/fs/cgroup/cpu.stat                     # sample twice, diff user_usec vs system_usec

# or from the node
pidstat -u -p <pid> 1                           # %usr vs %system for the process
```

70% system time in a numerical workload is absurd. Inference is arithmetic, which is user mode. A high system share says the CPU is going into syscalls, which narrows it to I/O, lock contention, memory management, or a syscall-based spin.

### Step 4: find the threads

```bash
top -H -p <pid>                                 # per-thread view; spinners sit at ~100% each
ls /proc/<pid>/task | wc -l                     # thread count; compare with node vCPUs

pidstat -t -u -p <pid> 1                        # per thread: %usr %system
pidstat -t -w -p <pid> 1                        # per thread: cswch/s (voluntary), nvcswch/s

# raw counters, no tools needed (works in a distroless debug container with busybox)
for t in /proc/<pid>/task/*; do
  printf '%s ' "${t##*/}"
  awk '{print "utime="$14, "stime="$15}' "$t/stat"
done
grep ctxt /proc/<pid>/task/<tid>/status
```

`utime` and `stime` are in clock ticks (`getconf CLK_TCK`, normally 100), and they're cumulative, so sample twice and take the difference. One caveat: `awk` field numbers in `/proc/<tid>/stat` are wrong if the thread name contains a space, because field 2 is the name in parentheses. Worker threads named `python3` are fine.

What we saw, in shape:

```
 thread               %usr   %system   cswch/s   nvcswch/s
 main / request loop   20       5          ~50        low
 BLAS worker x N       ~30      ~70         ~0       varies   <- each one a full core
```

A thread at a full core, mostly system time, with almost no voluntary switches, is a yield spinner. A healthy idle pool would show near-zero CPU and voluntary switches when it wakes up for work.

### Step 5: confirm the syscall and the function

```bash
strace -f -c -p <pid>          # summary: sched_yield dominates the syscall count
perf top -p <pid>              # hot symbols: blas_thread_server, do_sched_yield,
                               # entry_SYSCALL_64, schedule
```

Both need `ptrace` or perf permissions, which in a Kubernetes pod usually means an ephemeral debug container with `--profile=sysadmin` (see [[notes/Linux/anatomy-of-a-userspace-hang|Anatomy of a Userspace Hang]] for what that profile is actually unlocking). The `/proc` counters in steps 3 and 4 don't need any of that, and they're usually enough to reach the conclusion.

### Step 6: prove it with the knob

The cheapest proof is to change one variable on one pod and watch the floor drop. If `OPENBLAS_THREAD_TIMEOUT=4` or `OPENBLAS_NUM_THREADS=1` takes the per-pod floor from 1.4 cores back to 0.3 and leaves the per-request slope and latency alone, the mechanism is confirmed.

---

## Part 4: the fixes and the trap

### Option A: `OPENBLAS_NUM_THREADS=1`

No pool at all. Every BLAS call runs on the calling thread.

This is the right default for a request-parallel server. Concurrency comes from serving many requests at once, from multiple model instances or worker processes, and there's no idle core for an intra-op thread pool to use anyway. For small matrices it's no slower, often faster, because there's no fork-join overhead. It also makes CPU usage predictable: one request, one thread.

The cost is that a genuinely large single multiply (big batch, big layer) loses parallelism. If you need that, set the thread count to something sized to the pod's CPU request instead of the node's CPU count.

### Option B: `OPENBLAS_THREAD_TIMEOUT=4`

Keep the pool and its parallelism, but make workers sleep almost as soon as they finish. Each multithreaded call now pays a futex wake for every worker it uses, microseconds per call, in exchange for no spin between calls.

This is the knob to reach for when you can't control the thread count, for example because a framework or the application itself sets it.

### The trap: the variable was being set again in code

The first manifest change appeared to do nothing. The variable was set on the container, but the application's startup code set the thread count itself before the model runtime loaded, so the value that reached OpenBLAS's constructor wasn't the one in the manifest.

This happens a lot in Python model servers. Typical culprits:

```python
# somewhere in an entrypoint, a config loader, or a vendored helper
import os
os.environ["OPENBLAS_NUM_THREADS"] = str(os.cpu_count())   # overwrites the manifest value
os.environ["OMP_NUM_THREADS"] = str(os.cpu_count())        # also feeds OpenBLAS if
                                                           # OPENBLAS_NUM_THREADS is unset
import numpy                                                # constructor reads env NOW
```

Two lessons from it:

1. Never trust that an environment variable took effect. Verify the result, not the input. For a BLAS pool that's the thread count: `ls /proc/<pid>/task | wc -l` before and after, or `threadpoolctl.threadpool_info()` from inside the process, which reports each loaded BLAS library and its actual thread count.
2. `os.cpu_count()` in a container returns the node's CPU count, not the pod's share. Any code that sizes a pool from it is wrong by default on Kubernetes. Python 3.13 added `os.process_cpu_count()`, which respects the affinity mask, but even that ignores a CFS quota.

### The same pattern elsewhere

OpenBLAS isn't special. Most runtimes that keep a worker pool make the same spin-versus-sleep trade, and most of them pick the HPC answer by default:

| Runtime | Spin knob | Default behaviour |
|---|---|---|
| OpenBLAS (pthreads) | `OPENBLAS_THREAD_TIMEOUT` | ~2^28 cycles of `sched_yield` |
| Intel OpenMP (MKL, PyTorch on x86) | `KMP_BLOCKTIME` | 200 ms spin before sleeping |
| GNU OpenMP (libgomp) | `OMP_WAIT_POLICY`, `GOMP_SPINCOUNT` | spins for a bounded count before futex sleep |
| ONNX Runtime | session option `session.intra_op.allow_spinning` | spinning enabled |
| Go runtime | GOMAXPROCS bounds it | idle Ps spin briefly looking for work, capped at half of GOMAXPROCS |

Every one of these sizes its pool from the CPUs it can see, and in a container that's usually the node's CPUs. If you run inference on shared Kubernetes nodes, setting thread counts explicitly to match the pod's CPU request is the single most valuable habit here.

---

## Part 5: how Kubernetes made it worse

### The HPA was targeting a value the pod could not reach

The HPA used an absolute CPU target per pod (`type: AverageValue`) rather than a percentage of requests:

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: AverageValue
        averageValue: "1"          # illustrative: 1 core per pod
```

The HPA's rule is `desired = ceil(current_replicas * current_value / target)`. With a floor `F` and a per-request cost `c`, a pod's CPU at `N` replicas and total rate `R` is:

```
 per_pod_cpu(N) = F + c * R / N

 the HPA can only settle if   F + c * R / N  <=  target
 which needs                  F < target
```

Before the change, `F` was about 0.3 cores and the HPA could always find an `N` that worked. After it, `F` was about 1.4 cores, above the target on its own. No replica count satisfies the inequality, so every evaluation says "scale up", and the deployment sits at `maxReplicas` (180) whenever traffic is high enough to keep the spin going.

Adding pods also didn't help the way it normally would. Each new pod brought its own spinning pool, so total CPU grew by roughly `F` per pod while the useful work stayed the same:

```
 total_cpu = N * F + c * R

 N = 90,  F = 0.3   ->   27 cores of floor
 N = 180, F = 1.4   ->  252 cores of floor
```

That's most of the difference between 580 and 210 cores.

There's a nasty feedback loop hiding in there. Spreading traffic over more pods lengthens the gap between requests at each pod. If scale-out pushed that gap past ~100 ms, the spin would stop and the floor would collapse. It didn't get there: 180 pods still meant 33 to 40 ms between requests per model instance. A higher `maxReplicas` might eventually have "fixed" it by accident, at enormous cost, and then flapped as the HPA scaled back down and re-armed the spin.

A utilization target (`type: Utilization`, a percentage of the request) would have had the same problem in a different form, since the request itself was being sized from the inflated usage. See the next section.

### Rightsizing chased the spin

The service's CPU requests came from a recommender that takes the p98 of usage over a trailing 24-hour window. The spin raised p98, so requests went up, so the scheduler reserved more node capacity per pod. Nobody saw the requests go up as a problem, because "the new model needs more CPU" was the story everyone already believed.

After the fix, a percentile over a trailing window comes down slowly, and the lag is asymmetric:

```
 p98 over 24 h = the value exceeded by the top 2% of samples = top ~29 minutes

 usage goes UP      p98 follows once ~29 minutes of high samples are in the window
 usage goes DOWN    p98 stays high until fewer than ~29 minutes of old samples remain,
                    i.e. ~23.5 hours after the change
```

A percentile recommender reacts to increases within half an hour and to decreases a day later. That's deliberate, since it's safer to over-provision, but it means the savings from a fix show up a day late, and a before/after comparison taken the morning after the deploy will understate them.

### The dashboards kept averaging it away

Three averaging artifacts made the problem harder to see and the fix harder to quantify.

The first was averaging across containers. A query like "average CPU of containers in this service" with no grouping averages the application container together with every sidecar (proxy, log shipper, metrics agent), most of which use a few millicores. A pod with one container at 3 cores and one sidecar at 0.05 cores reports about 1.5 cores. Group by pod and sum containers first, or filter to the application container.

The second was rollups over churning pods. When the HPA moves between 90 and 180 replicas during the day, a coarse rollup (an hour per point) does strange things. An average across pods gives a pod that lived for 10 minutes the same weight as one that lived all hour. A sum across pods counts every pod that existed at any point in the bucket, including ones that were never alive at the same time. Compute fleet totals at 60-second resolution and average over time afterwards. [[notes/K8s/when-gauge-sums-lie|When Gauge Sums Lie]] covers this ghost-series effect in detail.

The third was averaging ratios. "Average CPU per request across pods" is not CPU per request. The pods with the lowest traffic have the highest ratio because the floor dominates them, so an average of ratios moves with the traffic distribution. Use a ratio of sums.

Related to the second: CPU time counters are cumulative. If a dashboard ever shows a CPU metric that only goes up, it's a counter being plotted as a gauge, and you need its rate.

---

## Part 6: results and what to keep

| | Before fix | After fix |
|---|---|---|
| Peak cores used | ~580 | ~210 |
| CPU per request | ~59 ms | ~26 ms |
| Per-pod floor | ~1.4 cores | ~0.3 cores |
| Daytime replicas | 180 (HPA at max) | ~90 |
| System CPU share | ~70% | small |
| Latency | unchanged | unchanged |

CPU per request came back to the pre-model-change level (~26 ms against ~27 ms). The wider embedding itself cost almost nothing. All of the regression was the thread pool.

Things worth keeping from this:

1. A load-independent per-pod floor is the signature of a background thread. Fit floor plus slope before you argue about whether the model is "heavier".
2. High system time in a compute workload means syscalls. A full core per thread with high system time and no voluntary switches means a yield spin.
3. Libraries size their pools from the CPUs they can see, and in a container that's usually the node. Set thread counts explicitly to match the pod's CPU request.
4. Spin windows turn cost into a step function of inter-arrival time. Light-traffic environments won't reproduce it.
5. Verify the effective thread count in the running process, not the environment variable in the manifest.
6. An absolute HPA target below a pod's fixed floor pins the deployment at `maxReplicas`, and scaling out multiplies the floor.
7. Percentile-window recommenders take nearly a full window to come down after a fix. Measure the outcome after the window has rolled.

---

## Interview Prep

### Q: CPU per request doubled after a model update but latency didn't change. How do you investigate?

**A:** I start by making the number load-independent. Total cores move with traffic, so I compute CPU per request as fleet cores divided by fleet requests per second, at 60-second resolution, before and after the change. That confirms the regression is real and not a traffic shift.

Then I split per-pod CPU into a floor and a slope by regressing pod CPU against pod request rate:

```
 pod_cpu = F + c * pod_rps
```

If the slope went up, the model's arithmetic really is heavier. If the floor went up and the slope didn't, something is burning CPU independently of the work, and that's almost always a background thread: a pool spinning, a polling loop, a GC, a sidecar.

Then I go to the threads. `top -H` or `pidstat -t -u` to see which threads are hot and whether it's user or system time, and the context switch counters in `/proc/<pid>/task/<tid>/status`. In this case a few dozen identical threads each at a full core, 70% system time, with almost no voluntary switches, which says a `sched_yield` spin. `strace -c` confirms `sched_yield` dominates the syscall count and `perf top` names `blas_thread_server`.

Latency staying flat fits too: the spinners were on otherwise idle cores and the pod had no CPU limit, so they never competed with the request thread.

### Q: Why would a thread pool spin instead of sleeping, and when is that the wrong choice?

**A:** To avoid wake-up latency. A sleeping worker needs a `futex(FUTEX_WAKE)` from the producer, a run queue insertion, and a scheduling decision before it runs, which is microseconds to tens of microseconds and worse if the CPU was in a deep C-state. A numerical library doing thousands of back-to-back operations in a tight loop pays that on every call, so it keeps workers hot for a window after each job.

```
 spin   costs a core for the whole window, saves the wake-up
 sleep  costs nothing while idle, pays the wake-up per job
```

Spinning only pays off when the next job usually arrives within the window and the wake-up cost is a meaningful fraction of the job. A request server breaks both conditions. Jobs arrive tens of milliseconds apart, so you pay nearly the whole window between them, and the jobs are small matrix-vector multiplies where the wake-up cost is negligible compared with the per-request overhead elsewhere. OpenBLAS's default window is 2^28 TSC ticks, about 100 ms, so with requests every 35 ms the workers never sleep at all.

### Q: Why does a `sched_yield` spin show up as system time and not user time?

**A:** Because each iteration is a syscall. The user-mode part of the loop is tiny: read the TSC, compare against a deadline, branch. Then `sched_yield()` enters the kernel, goes through `schedule()`, finds either another task to run or nothing, and returns. With speculative execution mitigations on syscall entry and exit, that kernel round trip is far longer than the user-mode instructions around it. So the thread is at 100% of a core and most of those cycles are charged as `stime`.

Compare a pure `while (!flag) {}` spin, which would be nearly all `utime`, and a futex sleep, which would be near zero of both. The user/system split is a cheap, reliable way to tell these apart from `/proc` alone.

The context switch counters add another dimension. A yield that hands the CPU to something else is counted as a nonvoluntary switch because the task was still runnable. A block on a futex counts as voluntary. So a yield spinner on a busy node shows high nonvoluntary switches, and on an idle node shows almost none, because the yield returns immediately with nothing to switch to.

### Q: How does OpenBLAS decide how many threads to use, and why is that dangerous in a container?

**A:** At load time it sets its pool size from `OPENBLAS_NUM_THREADS`, then `GOTO_NUM_THREADS`, then `OMP_NUM_THREADS`, and failing all of those, the number of CPUs in the process's affinity mask, capped at a build-time maximum. It spawns that many minus one workers.

Per call, it decides whether to use the pool. For `gemv` the rule is: if `m * n < 2304 * GEMM_MULTITHREAD_THRESHOLD` (9,216 with the default threshold of 4), run on one thread; otherwise use all available threads. `gemm` has a similar threshold on `m * n * k`.

In a container, the affinity mask is usually every CPU on the node, because only Guaranteed pods under the static CPU manager policy get exclusive cpusets. CPU requests and CFS quotas don't change what `sched_getaffinity` returns. So a pod requesting 2 cores on a 48-vCPU node gets 47 workers, and once a matrix crosses the threshold, all of them participate and all of them spin afterwards.

The other danger is that the variables are read once in the library constructor. Setting them in Python after `import numpy` is too late, and if application code overwrites them before the import, the manifest value never gets there. I verify the effective value from the running process, with `threadpoolctl` or by counting threads in `/proc/<pid>/task`.

### Q: Why did changing the embedding dimension from 32 to 128 cause this?

**A:** Because it moved a cosine-similarity feature (one query vector against ~120 candidate embeddings) across OpenBLAS's single-threaded threshold. The matrix went from `120 x 32 = 3,840` elements to `120 x 128 = 15,360`. The threshold for `gemv` is 9,216, so the old feature always ran on the calling thread and the pool never woke up, and the new one used the pool on almost every call.

Using the pool isn't the expensive part. A 120 x 128 multiply is microseconds on one core, so splitting it saves almost nothing. The expensive part is what happens afterwards: every worker that took part spins for about 100 ms waiting for more work. With requests every 33 to 40 ms, that window never expires.

So the cost is a step function of two variables: whether the matrices cross the threshold, and whether the gap between requests is shorter than the spin window. A light-traffic test environment fails the second condition and looks perfectly fine.

### Q: What's the difference between `OPENBLAS_NUM_THREADS=1` and `OPENBLAS_THREAD_TIMEOUT=4`, and which would you pick?

**A:** `OPENBLAS_NUM_THREADS=1` removes the pool. Every BLAS call runs on the caller's thread, there are no workers to spin, and CPU usage is one thread per in-flight request. `OPENBLAS_THREAD_TIMEOUT=4` keeps the pool but shrinks the spin window to 2^4 = 16 TSC ticks (the minimum the library allows), so workers go to a condition-variable sleep almost immediately and each multithreaded call pays a futex wake per worker.

For a request-parallel server I'd pick `NUM_THREADS=1`, or a small number sized to the pod's CPU request. Concurrency comes from serving requests in parallel, the matrices are too small to benefit from fork-join, and one thread per request makes capacity planning predictable.

I'd use `THREAD_TIMEOUT=4` when something else owns the thread count, like a framework or application code that sets it, or when there really are large multiplies that benefit from the pool. It's also a good fallback when the thread count variable is being overwritten, which is exactly the trap here: the manifest set the thread count, the startup code set it again, and the first rollout appeared to change nothing.

### Q: The HPA was pinned at maxReplicas. Why didn't scaling out bring per-pod CPU down?

**A:** Because most of each pod's CPU was a fixed floor, not load. With an `AverageValue` target `T`, the HPA needs a replica count `N` where:

```
 F + c * R / N  <=  T
```

The second term shrinks as you add pods, the first doesn't. Once the floor `F` (about 1.4 cores per pod) exceeded the target, no `N` satisfied it, so the HPA kept asking for more and sat at max.

And scaling out made total cost worse, because each new pod brought its own spinning pool: total CPU is `N * F + c * R`, so going from 90 to 180 pods at 1.4 cores of floor costs about 250 cores of pure waiting.

The general point is that HPA assumes per-pod CPU is roughly proportional to per-pod load. Any load-independent cost (spinning pools, oversized sidecars, warm caches being rebuilt) breaks that assumption, and you should check the floor before trusting an absolute CPU target.

### Q: After the fix, the CPU dropped immediately, but requests and node count took a day to follow. Why?

**A:** The requests came from a rightsizing recommender using the p98 of usage over a trailing 24-hour window. A p98 over 24 hours is set by the top 2% of samples, about 29 minutes' worth. When usage rises, it only takes 29 minutes of high samples to move it, so it tracks increases quickly. When usage falls, the old high samples have to age out until fewer than 29 minutes of them remain, which takes about 23.5 hours.

So requests stayed inflated for nearly a day after the fix, and the scheduler kept reserving node capacity for CPU nobody was using. That's a deliberate safety bias, but it means you shouldn't measure savings until the window has fully rolled. If the savings are urgent, you can reset the recommender's history or pin the request by hand for that window.

### Q: What dashboard mistakes can hide or exaggerate a regression like this?

**A:** Three common ones.

Averaging across all containers in a service mixes the application container with sidecars that use a few millicores, which roughly halves the reported per-pod CPU and dilutes any change. Group by pod and sum containers, or filter to the application container.

Coarse rollups over a fleet that's scaling up and down. With an hour per point and pods coming and going, an average gives short-lived pods full weight, and a sum counts pods that never existed at the same time. Aggregate across pods at 60-second resolution first, then average over time.

Averaging per-pod ratios instead of taking a ratio of sums. "CPU per request" should be total cores over total requests. Low-traffic pods have huge ratios because the floor dominates, so averaging ratios makes the number depend on how traffic happens to be spread.

And plot CPU time counters as rates. A line that only goes up is a cumulative counter, not usage.

### Q: How would you prevent this class of problem across a platform?

**A:** Set thread counts explicitly for every runtime that keeps a pool, sized to the pod's CPU request rather than the node: `OPENBLAS_NUM_THREADS`, `OMP_NUM_THREADS`, `MKL_NUM_THREADS`, and the framework-level settings like ONNX Runtime's intra-op threads. A platform can inject defaults per language or framework, the same way many platforms already set `GOMAXPROCS` for Go services. Spin settings deserve the same treatment: `KMP_BLOCKTIME`, `OMP_WAIT_POLICY`, `OPENBLAS_THREAD_TIMEOUT`.

Then make the regression visible. Track CPU per request and the per-pod floor as first-class service metrics, so a model rollout that changes either gets noticed in hours, not in the monthly bill. Alert on system CPU share for workloads that should be pure compute.

And check the autoscaling config against the floor: an `AverageValue` target below a pod's idle CPU can never be satisfied, and that's cheap to lint for.

---

## Related Notes

- [[notes/Linux/anatomy-of-a-userspace-hang|Anatomy of a Userspace Hang]] covers the sibling failure: one thread at a full core of pure user time with no syscalls, and how `utime`/`stime`, `wchan` and `ptrace` sampling tell the cases apart
- [[notes/Linux/linux-fundamentals-architecture-and-debugging|Linux Fundamentals, Architecture & Debugging]] is the wider map of processes, scheduling, cgroups and the debugging toolbox
- [[notes/K8s/kubernetes-autoscaling|Kubernetes Autoscaling]] explains the HPA control loop and scaling formula that the floor broke
- [[notes/K8s/vpa-eviction-loops|The VPA Eviction Loop]] is another way rightsizing and the workload's real behaviour fight each other
- [[notes/K8s/when-gauge-sums-lie|When Gauge Sums Lie]] covers ghost series and the sum-versus-avg traps behind the dashboard artifacts here
- [[notes/K8s/istio-and-envoy-internals|Istio & Envoy Internals]] includes Envoy's threading model, a useful contrast: a fixed worker-per-core event loop that blocks in `epoll` instead of spinning
