---
title: "Alive But Not Shipping: Node-Level Log Pipelines and Their Silent Failure Modes"
---

## The symptom that shouldn't be possible

An analytics query that has run every morning for a year returns nothing for a slice of traffic. Someone digs in and finds that the BigQuery table backing it is missing rows — not all rows, just the ones from **every pod on one node**, starting at 03:14 and never resuming.

Everything you'd normally check looks fine:

```
$ kubectl get pods -o wide | grep gke-prod-pool-a-7f3c
api-7d9f-2xk4v      1/1     Running   0     6d    10.4.2.11   gke-prod-pool-a-7f3c
worker-55b8-qnp7z   1/1     Running   0     6d    10.4.2.19   gke-prod-pool-a-7f3c
...

$ kubectl logs api-7d9f-2xk4v --tail=3
{"ts":"2026-09-04T09:12:44Z","level":"info","msg":"handled request"}   # fresh
```

- The application pods are healthy and *are* writing logs. `kubectl logs` returns current output.
- The node is `Ready`, not under pressure, not cordoned.
- The log agent DaemonSet reports `1/1 Running` on that node with **zero restarts on the agent container**.
- And the kicker: a **second, unrelated log agent** on the same node — a vendor observability agent tailing the same files — is still shipping those pods' logs to its own backend, happily, the whole time.

So the logs exist. The files exist. One agent reads them fine. The other agent is "running." And yet an entire node's worth of logs vanished from the warehouse for hours.

Three separate mechanisms conspired here, and pulling them apart is the whole point of this note:

1. A **hung agent engine** — a live process, pinned at exactly one core, flushing nothing, logging nothing, and never restarting because nothing was watching for *progress*. The confirmed cause is a **userspace infinite loop**: a once-per-second bookkeeping walk over the agent's own metrics went round a circular linked list that no longer contained its own head, because one of its nodes had been freed while still linked and then reallocated during an error burst. Not a network retry spin, not a deadlock, not backpressure.
2. **Segfaults in the metrics path** — the *same* defect, in the *same* function, on the *same* thread, landing differently. A stale pointer into unmapped memory faults and the container restarts; a stale pointer into a valid ring loops forever and nothing restarts. So the crashes are loud, counted and self-healing while the hangs are silent and permanent, which makes the crashes read as a decoy. They are better understood as a **leading indicator** of #1. And in multi-container logging DaemonSets the same class of bug can also blind the one alert that could have caught #1.
3. A **schema mismatch at the export sink**, downstream of everything above, quietly diverting a different set of entries into a dead-letter table nobody has ever queried, for a completely different reason and with a completely different blast-radius shape.

The shapes are the diagnostic. One is node-shaped and total, one is self-healing, one is field-shaped and partial. Getting good at reading blast-radius shape is most of what this note teaches.

> The OS-level half of #1 and #2 — threads, `/proc`, tick accounting, the heap, what `free()` actually does, intrusive circular lists, and the `ptrace` sampling that proved it — is built up from scratch in [[notes/Linux/anatomy-of-a-userspace-hang|Anatomy of a Userspace Hang]]. This note stays on the operational side.

---

## The pipeline, end to end

Before anything else: know every hop, who owns it, and what evidence it leaves. Almost every log-loss incident is a *localisation* problem, not a mystery.

```
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ NODE                                                                    │
 │                                                                         │
 │  ┌──────────┐   stdout/stderr (fd 1/2)                                  │
 │  │ app      │──────────────┐                                            │
 │  │ container│              │                                            │
 │  └──────────┘              ▼                                            │
 │                     ┌─────────────┐                                     │
 │                     │ containerd  │  CRI logger: timestamps, tags,       │
 │                     │  (CRI)      │  rotation (kubelet-configured)       │
 │                     └──────┬──────┘                                     │
 │                            ▼                                            │
 │        /var/log/pods/<ns>_<pod>_<uid>/<container>/<restartCount>.log     │
 │        /var/log/containers/<pod>_<ns>_<container>-<id>.log  (symlink)    │
 │                            │                                            │
 │              ┌─────────────┴──────────────┐                             │
 │              ▼                            ▼                             │
 │   ┌────────────────────┐        ┌────────────────────┐                  │
 │   │ log agent A        │        │ log agent B        │  independent     │
 │   │ (fluent-bit DS)    │        │ (vendor DS)        │  tail state,     │
 │   │  in_tail + offsetdb│        │                    │  buffers, dest   │
 │   │  filters (k8s meta)│        └─────────┬──────────┘                  │
 │   │  buffer (mem/fs)   │                  │                             │
 │   │  out_* plugin      │                  │                             │
 │   └─────────┬──────────┘                  │                             │
 └─────────────┼─────────────────────────────┼─────────────────────────────┘
               │ entries.write (batched, gzip, HTTPS)
               ▼                             ▼
     ┌───────────────────────┐        (vendor backend)
     │  Cloud Logging API    │
     │  ingest + quota + IAM │
     └───────────┬───────────┘
                 ▼
     ┌───────────────────────────────────────────────┐
     │  LOG ROUTER  (every sink evaluated against    │
     │  every entry, independently, in parallel)     │
     └───┬───────────────┬───────────────┬───────────┘
         ▼               ▼               ▼
   _Required        _Default        user sink
   bucket 400d      bucket 30d      filter: resource.type="k8s_container"
   (immutable)      (excludable)    destination: bigquery.googleapis.com/...
                                            │
                                            ▼
                                  ┌──────────────────────┐
                                  │ BigQuery dataset     │
                                  │  table per log ID    │
                                  │  schema from the     │◄── type conflict or
                                  │  FIRST entry seen    │    column overflow
                                  └──────────┬───────────┘    diverts entries
                                             ▼               to export_errors
                                     export_errors[_YYYYMMDD]
```

Two structural facts to carry through the rest of the note:

**The node half and the cloud half fail differently.** On the node, one process serves *every pod on that node*, so its failures are node-shaped and total. In the cloud, one sink serves *every node in the project*, so its failures are project-shaped — usually partial (specific entries, specific fields), occasionally total (quota exhaustion stops a sink outright).

**Fan-out is independent.** The Log Router evaluates each sink separately. A BigQuery sink failing tells you nothing about the bucket, and logs being visible in Logs Explorer tells you nothing about whether the sink wrote them. That independence is what makes "it's in Logs Explorer but not in BigQuery" a *precise* localisation rather than a contradiction.

**And the router is not a replay buffer.** It holds entries briefly to ride out transient destination trouble, but that buffer explicitly does not protect against *configuration* errors, and sinks are never retroactive: fix a broken sink and it resumes with newly arriving entries only, with nothing backfilled. So every second a sink is misconfigured is a second of rows you'll have to replay from a bucket by hand.

**And on a managed platform there's an extra hop inside the pod.** GKE's logging DaemonSet (`fluentbit-gke` and its `-256pd` / `-max` / Autopilot variants, all labelled `k8s-app=fluentbit-gke`) splits the work across containers: the `fluentbit` container tails the files and its *only* output is an `http` plugin posting msgpack to `127.0.0.1:2021` with `Retry_Limit 2`; a second container (`fluentbit-gke`) receives that and makes the gRPC `WriteLogEntries` calls to Cloud Logging. There's also a `fluentbit-metrics-collector` container scraping `127.0.0.1:2020`.

That topology matters for two reasons. It adds a hop that can fail on its own, between two containers with independent lifecycles inside one pod. And the DaemonSet runs in addon-manager `Reconcile` mode, so your edits get reverted and deletions undone — which is exactly why teams end up running a *second*, self-managed agent alongside it rather than tuning the managed one. That's how a node ends up with two independent log agents, and why the "one agent works, the other doesn't" clue is available at all.

---

## What the agent is actually reading

The left half of the pipeline is just files, and their details explain several classes of loss.

The kubelet builds the paths, and the shape matters:

```
/var/log/pods/<namespace>_<pod>_<uid>/<container>/<restartCount>.log     ← the real file
/var/log/containers/<pod>_<namespace>_<container>-<containerID>.log      ← kubelet symlink
```

Note `<restartCount>.log`, not `0.log` — every container restart starts a new file, and the symlink is recreated pointing at it. (The symlink name is truncated to 251 characters + `.log` to fit ext4's 255-byte limit, and the kubelet source still carries a TODO to remove the whole legacy symlink farm.)

When the container runtime captures stdout it writes one line per record in the **CRI log format**:

```
2026-09-04T03:14:02.115492871Z stdout F {"level":"info","msg":"tick"}
│                              │      │ │
│                              │      │ └─ the application's actual bytes
│                              │      └─── tag: F = full line, P = partial (line continues)
│                              └────────── stream: stdout | stderr
└───────────────────────────────────────── RFC3339Nano, runtime's clock
```

That `P` tag matters more than people expect. The runtime reads container output through a fixed-size buffer, so a single application log line longer than it is emitted as several `P`-tagged fragments terminated by one `F`. Whoever tails the file has to rejoin them — in Fluent Bit that's `multiline.parser cri` (the tag delimiter is `:`, reserved for future multi-tag use). Get it wrong and large JSON payloads arrive downstream truncated, which is also excellent fuel for the schema conflicts in Failure 3.

### Rotation is the kubelet's job, and it is rename-then-reopen

| Knob | Default | Meaning |
|---|---|---|
| `containerLogMaxSize` | `10Mi` | rotate the active file at this size |
| `containerLogMaxFiles` | `5` | keep this many, including the active one |
| `containerLogMonitorInterval` | `10s` | how often the kubelet checks sizes |
| `containerLogMaxWorkers` | `1` | rotation concurrency |

The mechanism is worth knowing exactly, because it decides whether a tailing agent loses data:

```
every containerLogMonitorInterval, if <n>.log >= containerLogMaxSize:
    rename  <n>.log  ->  <n>.log.20260904-031402
    CRI ReopenContainerLog   ── runtime opens a fresh <n>.log
        (if the reopen fails, the rename is rolled back)
    gzip older rotated files -> .log.<ts>.gz
    delete anything beyond containerLogMaxFiles
```

That's **rename + reopen**, not copytruncate. The consequence is good news for the agent: it holds an open fd on the *old inode*, so it can keep draining a rotated file after the rename — the bytes don't vanish the instant rotation happens. What kills them is the deletion step. On-node history is bounded at roughly `10Mi × 5` per container, and a chatty container can churn all five in **seconds**. Fall more than four rotations behind and the files are gone. There is no replay.

This is why "the agent was stuck for four hours" is not recoverable by restarting it: the offsets it resumes from point into inodes that were unlinked hours ago.

### The subtler loss: paused input plus rotation

There's a second, much less obvious way rotation eats data, and it's the one that produces *partial* gaps rather than a clean cliff.

When an input hits `Mem_Buf_Limit`, Fluent Bit pauses it and logs:

```
[warn] [input] tail.0 paused (mem buf overlimit)
[ info] [input] tail.0 resume  (mem buf overlimit)
```

But the pause is selective. It stops the reader collector and the filesystem watcher — and it deliberately does *not* stop the scan and purge collectors. So while the input is paused:

```
 reader collector      PAUSED   ── returns BUSY immediately, reads nothing
 rotated-file purge    STILL RUNNING every rotate_wait seconds
        └── when  file.rotated + rotate_wait <= now
              → drop the file from the watch list
              → delete its offset row from the sqlite db
              → if it still had pending bytes, log:
                  [warn] purged rotated file while data ingestion is paused,
                         consider increasing rotate_wait
```

Those pending bytes are gone. That warning line is one of the most valuable strings in the whole system, and it names its own fix.

Two practical consequences. First, `rotate_wait` needs to be larger than your worst realistic backpressure pause; with a default of 5 seconds and a kubelet rotating at 10Mi, a busy container plus a briefly wedged output is enough. Second, **filesystem buffering changes the outcome entirely**: with `storage.type filesystem` the input keeps writing chunks to disk instead of pausing (unless you opt into `storage.pause_on_chunks_overlimit`), so the reader never stops and the purge never catches pending data. Memory-only buffering is what turns backpressure into loss.

And one anti-pattern that follows directly: do **not** exclude the agent's own `mem buf overlimit` warnings from your log pipeline to reduce noise. That line is the backpressure signal. Filtering it out is filtering out the only warning you get before rotation starts eating data.

### The most commonly misread signal on this path

> `kubectl logs` reads those node-local files through the kubelet. It proves the application is producing output and the runtime is capturing it. **It proves nothing whatsoever about shipping.** Every failure in this note leaves it healthy.

It can also actively mislead in the other direction: `kubectl logs -f` does not read rotated files, so a container that just rotated can look quiet when it isn't.

## Fluent Bit's concurrency model: one thread does the work that matters

To understand why one wedged process takes down an entire node's logging — without dying, erroring, or restarting — you need the threading model. It is not single-threaded, which is exactly what makes the failure confusing.

```
 fluent-bit process
 ├── process main thread ──── spawns the engine, then blocks on shutdown/signals
 │
 ├── ENGINE thread  (flb_lib_worker → flb_engine_start)
 │     one event loop (epoll) owning, per the docs, "timers, receiving internal
 │     messages, scheduling flushes, and handling retries"
 │       ├── input collectors (in_tail read / scan / rotate / watcher timers)
 │       ├── ALL filters -- "filters always run in the main thread"
 │       ├── the retry scheduler
 │       └── the 1-second metrics-collection timer   ◄── remember this one
 │
 ├── log thread ──────────── drains the internal log ring and writes it out
 ├── HTTP monitoring thread  :2020  /api/v1/*
 └── output worker threads ─ one per `workers` (many network outputs default to 1),
                             each with its own event loop + scheduler timer;
                             flush coroutines (libco) run here and yield on network waits
```

Six-ish OS threads for a typical config, and the important asymmetry is that **only one of them can advance the pipeline**. Three consequences fall straight out:

**One non-yielding callback on the engine thread stalls everything on the node.** Scheduling here is cooperative: a coroutine yields voluntarily, typically when it hits a network wait, and a timer callback simply runs to completion. Anything that never returns — a retry loop, a parser, or a bookkeeping walk over a corrupted structure — is never preempted, and nothing else on that loop runs. Inputs stop being read, filters stop, retry timers never fire, and the metrics snapshot freezes. Every pod on the node stops shipping at the same instant, which is precisely the node-shaped, total blast radius we observed.

Worth noting which callback it turned out to be, because it is not the one anyone guesses: **the 1-second metrics timer**. A periodic walk over the agent's own counters is what took down logging for every application pod on the node.

**The monitoring HTTP server is a different thread, so `:2020` keeps answering while the engine is dead.** Worse, it answers *plausibly*. Two endpoints diverge in a way that is itself the diagnostic:

| Endpoint | Where the value comes from | During an engine stall |
|---|---|---|
| `/api/v1/uptime`, `fluentbit_uptime` | computed at request time in the HTTP thread (`now - init_time`) | **keeps climbing** |
| `/api/v1/metrics*` plugin counters | the **last snapshot the engine pushed**, via a 1-second timer *on the engine loop* | **frozen** |

That's the mechanism behind the fingerprint in the next section. The metrics exporter is not an independent scraper of live state; it's a producer/consumer handoff where the *producer runs on the thread that stalled*. So "uptime rising while record counters sit still" isn't a coincidence, it's structural — and it's the single cheapest remote signal for a wedged engine.

**Silence has a specific cause, and it isn't a dead logger.** Fluent Bit's log writer genuinely is its own thread, so the channel is alive. But log *lines* are produced by the code that stalled. The channel is fine and has nothing to carry. Expect silence from a hang; don't read it as "the logging is broken."

Two more defaults worth carrying into the failure sections. `flush` is **1 second**, so a healthy agent should show counter movement within a couple of scrapes — a stall is visible fast if you look. And retries use exponential backoff with jitter (`scheduler.base` 5, `scheduler.cap` 2000 by default: the Nth retry waits a random value in `[base, min(base·2^N, cap)]`), with `Retry_Limit` defaulting to **1**. That last number surprises people: by default a chunk gets one retry, then it's dropped and counted in `output_dropped_records_total`.

## Failure 1 — the hung engine

### The evidence profile

A hang has a fingerprint. Once you've seen it once you can recognise it in about thirty seconds:

| Signal | Healthy | Hung |
|---|---|---|
| Container `restartCount` | 0 | **0** (nothing killed it) |
| Process state | running | **running** |
| CPU | spiky, 5–50 m | **flat, pinned at ~1.00 core** |
| RSS | drifts with load | **flat, unchanging** |
| `fluentbit_uptime` | climbing | **climbing** (computed in the HTTP thread) |
| `..._input_records_total` | climbing | **frozen** |
| `..._output_proc_records_total` | climbing | **frozen** |
| `..._output_errors_total` | 0 | **0** |
| `..._output_retries_failed_total` | 0 | **0** |
| Agent's own stderr | periodic info lines | **nothing since T** |
| `/api/v1/health` | `ok` | **`ok`** — or `404`, see below |

Three rows define the class. **CPU pinned at exactly one core** is the signature of a busy loop on the engine's event loop: it is executing, at full speed, making no progress. Not a deadlock (that would sit near 0%), not a crash, not backpressure (that would show retries and rising buffer usage). **Zero errors** is what makes every error-rate alarm and health check useless. And **uptime climbing while record counters sit still** is the pair that proves it remotely, for the reason from the previous section: uptime is computed in the HTTP thread, while counters are pushed by a timer on the thread that stalled.

Plot the counters and you get the cleanest possible picture:

```
records/s
  │
  │ ╭╮╭─╮╭╮ ╭─╮╭╮╭╮
  │ ││││ ││││ │││││
  │─╯╰╯╰─╯╰╯╰─╯╰╯╰╯──────────────────────────────────────  0 for 4h
  └────────────────────┬───────────────────────────────────
                     03:14                              07:30 (restart)
CPU (cores)
  │                    ┌────────────────────────────────┐
  │ ▁▂▁▃▁▂▁▂▁▃▁▂▁▂▁▂▁▂ │  1.00  ─────────────────────── │
  └────────────────────┴────────────────────────────────┴──
```

Two flat lines that start at the same instant, one at zero and one at one core. Nothing else in this system produces that pattern.

One caveat before you build a detector on the CPU shape. A fleet sweep for zero output found **101 non-shipping nodes out of 852**, and they split into two shapes: **78 pinned at a flat one core** (the loop analysed below) and **23 sitting at exactly 0 m of CPU**, also `1/1 Ready`, also with zero restarts, also shipping nothing. Zero CPU with work outstanding is the deadlock signature, not the spin signature, so that is a *second* bug and it is still open. The consequence for detection is blunt: searching for pods near one core found 54 of the 101, while asking "which nodes ship nothing" found all of them. Symptom-shaped detectors under-count; the outcome-shaped one does not.

### Why nothing restarted it

Kubernetes restarts a container when the process exits or when a liveness probe fails. A hung-but-alive process satisfies neither, and the kubelet's entire view of that container is "the PID I started is still there" — true and useless. Three separate gaps conspire, and each has a different fix.

**Gap one: many logging DaemonSets ship no probe at all.** Self-managed agent DaemonSets very often have no `livenessProbe`, `readinessProbe` or `startupProbe` whatsoever. Absent a probe, the only things that will ever recover the agent are a human, a node rotation, or a DaemonSet rollout.

**Gap two: the probes that do exist test reachability, on the wrong thread.** GKE's managed agent, for instance, *does* have liveness probes: an `httpGet /` against `:2020` on the `fluentbit` container, and `/healthz` on `:2021` for the exporter (both `periodSeconds: 60`, `failureThreshold: 3`). But `/` on `:2020` is served by the monitoring thread. It asserts "the HTTP server is answering," which stays true throughout a hang. A probe that lives on a different thread from the work cannot observe the work stopping.

**Gap three: the purpose-built health endpoint doesn't do what its name suggests — in two different directions.**

`Health_Check` defaults to **`Off`**, and when it's off the endpoint isn't registered at all. So a liveness probe pointed at `/api/v1/health` on a default config gets a **404** and restarts a perfectly healthy agent every few minutes. People hit that, conclude the probe is broken, and remove it.

Turn it on and you get the opposite failure. Health is defined as an *error-rate* predicate over a rolling window:

```
unhealthy  ⟺   errors_in_window        > HC_Errors_Count          (default 5)
            OR  retry_failures_in_window > HC_Retry_Failure_Count   (default 5)
                                  within  HC_Period                (default 60s)
```

Read that as what it measures versus what you need:

```
health = NOT (too many output errors recently)   ← what it measures
       ≠ (records are still moving)               ← what you need
```

Two fail-open properties make this concrete for a hang, and both are visible in the implementation. The window is filled by that same **1-second timer on the engine loop**: when the engine stalls, the newest sample freezes, so the newest-minus-oldest delta becomes 0 and the endpoint keeps returning `200 ok` forever. And with no samples at all, the code returns healthy by default. Zero errors, zero failed retries, therefore healthy, indefinitely. Community reports confirm the same behaviour with a destination that is fully unreachable, as long as the error rate stays under the threshold.

Upstream is aware of the thread-separation part and has an integration test that asserts it, with a docstring reading roughly *"the monitoring server must not share the blocked collector loop"* — the server staying up while the engine is blocked is now a **guaranteed property**, not an accident. Which is right for a monitoring endpoint and fatal for a liveness probe pointed at it.

This generalises far beyond log agents:

> A liveness probe that tests **reachability** or **error rate** cannot detect a stall — especially when it's served by a thread that isn't the one doing the work. Liveness for a pipeline component must test **forward progress**: a monotonic counter that has moved inside a bounded window.

### A probe that actually works

Progress-based liveness, entirely in-container, no extra binaries:

```yaml
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - |
        # Fail if total output records haven't moved since the last check.
        M=/tmp/last_records
        CUR=$(curl -sf --max-time 5 localhost:2020/api/v1/metrics/prometheus \
              | awk '/^fluentbit_output_proc_records_total/ {s+=$2} END {printf "%.0f", s+0}')
        [ -n "$CUR" ] || exit 1                  # endpoint dead => unhealthy
        PREV=$(cat "$M" 2>/dev/null || echo -1)
        echo "$CUR" > "$M"
        [ "$PREV" = "-1" ] && exit 0             # first run: no baseline yet
        [ "$CUR" -gt "$PREV" ] && exit 0         # progress => healthy
        exit 1
  periodSeconds: 60
  failureThreshold: 5      # ~5 min of zero progress before restart
  timeoutSeconds: 10
```

The design details are the whole value here:

- **`failureThreshold` must exceed the longest legitimate idle period.** A node whose pods genuinely emit nothing at 04:00 will restart its agent forever otherwise. Five consecutive minutes of zero records on a busy node is safe; on a quiet node, either widen the window or make the check tolerant of a genuinely-empty input (compare `input_records` too, and only fail when input is moving while output isn't — that's unambiguous backpressure or stall).
- **Sum across outputs.** Multi-output configs will otherwise trip when one destination is idle by design.
- **Treat an unreachable endpoint as unhealthy**, since a dead monitoring thread is also a real failure. But note the ordering hazard: if the HTTP server dies while the engine is fine, you restart a working agent. That's an acceptable trade — restarting a log agent is cheap when its buffer is on disk.
- **Persistent buffering makes the restart safe.** With `storage.type filesystem`, in-flight chunks survive the kill; with memory buffering, a restart discards them. Fixing liveness without fixing buffering trades a silent stall for a smaller silent loss.

The stronger control, and the one I'd build first, is external: a **per-node minimum** alert on shipped records. Node-shaped failures are invisible to fleet aggregates — one node out of 1,200 going to zero moves the cluster total by 0.08%. You need `min by (node)`, not `sum`:

```promql
# Any node whose agent has shipped nothing for 10 minutes, while its pods are running.
min by (node) (
  rate(fluentbit_output_proc_records_total[10m])
) == 0
```

That single alert catches hangs, crash-loops, OOM kills, config-parse failures, and expired credentials, without knowing which one happened.

### Diagnosing a hang while it's still hung

Do not restart it first. A live hang is the only chance to learn the actual root cause, and the reflex to restore service destroys the evidence. Capture, then restart.

```bash
# 0. Which container, which PID. (Per-container -- see Failure 2.)
kubectl get pod -n kube-system "$POD" \
  -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.restartCount}{"\t"}{.state}{"\n"}{end}'

# 1. Frozen counters: is it a stall, or genuinely no input?
kubectl exec -n kube-system "$POD" -c "$AGENT" -- \
  curl -s localhost:2020/api/v1/metrics/prometheus | grep -E 'records_total|errors_total|retries|uptime'
# Run twice, 30s apart. Counters identical while uptime advanced = stall.

# 2. Buffer state: is it backpressure or a hang?
#    (needs `storage.metrics on` in the [SERVICE] section)
kubectl exec -n kube-system "$POD" -c "$AGENT" -- curl -s localhost:2020/api/v1/storage
# chunks piling up in fs/mem with zero output = engine not draining
# chunks NOT piling up either = the input side is stuck too (deeper hang)

# 3. Which thread is burning the core?
kubectl exec -n kube-system "$POD" -c "$AGENT" -- top -H -b -n1 -p 1
# one thread at ~100%, others idle => busy loop on a single thread confirmed
# minimal/distroless image with no `top`? read per-thread CPU straight from procfs:
#   for t in /proc/1/task/*; do awk '{print $1, $14, $15}' $t/stat; done
#   fields 14/15 = utime/stime in clock ticks; sample twice and diff

# 4. Is it looping in userspace or in syscalls? (the decisive test)
kubectl debug node/"$NODE" -it --image=ubuntu --profile=sysadmin -- bash
  timeout 5 strace -f -p "$PID" -c        # syscall summary over 5s
```

Step 4 is the one that splits the diagnosis cleanly:

| `strace` shows | Interpretation |
|---|---|
| Tens of thousands of `read`/`recvfrom` per second, **all `EAGAIN`** | The classic TLS/socket spin — see the next section. Most common real case |
| Tight `epoll_wait` returning immediately with timeout 0 | Event-loop spin: an fd is permanently ready or a stale node sits in the ready queue, and the handler never drains or unregisters it |
| **No syscalls at all**, CPU at 100% | Pure userspace loop — a parser, filter, or bookkeeping loop that never terminates. **This is what the confirmed hang looked like**, and an empty `strace -c` table is the *positive* finding, not "it's doing nothing". Escalate to `ptrace`/`perf` to find the loop |
| Blocked in one `write`/`read`/`futex`, CPU **~0%** | Deadlock or a saturated internal channel, *not* the busy-loop class |
| `nanosleep` + `epoll_wait` in a slow rhythm | Healthy idle |

Pair it with `ss` on the container's sockets. A connection that is `ESTABLISHED` with an empty receive queue and a `lastrcv` measured in *days* is the smoking gun for the spin case:

```bash
ss -tinp 2>/dev/null | grep -A1 :443 | head -20    # look at lastrcv / rx_queue
```

In every case, get a stack before you kill it:

```bash
gdb -p "$PID" -batch -ex "thread apply all bt" 2>/dev/null | head -60
# or, without gdb:
eu-stack -p "$PID"
perf top -p "$PID"        # which symbol is hot -- usually names the plugin
```

A hot symbol plus a stack turns "the agent hangs sometimes" into an upstream bug report with a fix. Without them you are pinned to the only remaining remedy: restart on a schedule and hope.

### What actually wedges the engine

"It hangs sometimes" is not a root cause, and the real mechanisms are worth knowing because most of them have a config-level mitigation available today.

**What it was here: a userspace infinite loop in the metrics-copy list walk.** This is the confirmed mechanism, and it is worth leading with because it is the one class in the whole family that has *no* config-level mitigation at all.

The agent's metrics library keeps each metric instance's label values in an **intrusive circular doubly linked list**, the same shape as the kernel's `list_head`, with the head embedded as a sentinel in the owning struct. Walking one is always `for (it = head->next; it != head; it = it->next)`, and the only termination condition that exists is arriving back at the head address. No NULL terminator, no length field, no iteration cap.

Once a second, on the engine loop, the metrics timer copies the entire metrics tree for the exporter, and copying a label list starts by counting it. So every metric's label list is walked in full, every second.

```
 a label node was freed WITHOUT being unlinked from its list
        │
        │  free() does not erase the chunk. it overwrites the first 16 bytes with
        │  allocator bookkeeping and leaves the rest verbatim, so the ring still
        │  walked correctly and there was NO SYMPTOM. for weeks.
        ▼
 the first 403 response created a new metric, requests_total{status="403", ...}
        │
        │  its label nodes needed the same size class, and glibc's tcache is LIFO,
        │  so the allocator handed back exactly that freed chunk
        ▼
 now two owners share one node:
        the 403 metric's list is a healthy ring:   B → C → A → B
        the stuck metric's head still points at A: head → A → B → C → A → ...
        head is OUTSIDE the ring it points into, so `it != head` is true forever
        ▼
 the next tick of the metrics timer enters the count loop and never returns.
 seven instructions, no calls, no syscalls, one core, permanently.
```

Two properties of that make it maximally hostile to operators. The loop makes **no system calls**, so `strace -c` prints an empty table and the reflex conclusion is "the process is doing nothing" (the correct reading of an empty table is the opposite: pure userspace loop). And every pointer involved is valid, so nothing faults, nothing errors, and nothing is logged.

The decisive measurement, if you get to a live one, takes three seconds and needs only `cat`: sample `/proc/<pid>/task/*/stat` twice and diff fields 14 and 15, which are user and kernel CPU time in 100 Hz ticks. The hung engine thread showed **+313 user ticks and +1 kernel tick over three seconds**, meaning 1.04 cores of pure userspace execution with 0.3% of the time in the kernel. That single ratio eliminates every syscall-spin theory below, including the next one.

The mitigation is a version upgrade, plus outcome-based detection so the next instance is a page rather than an archaeology project. There is no knob. Full derivation, including the `ptrace` sampling that proved it and the memory dump that pinned the reuse, is in [[notes/Linux/anatomy-of-a-userspace-hang|Anatomy of a Userspace Hang]].

**The best-known class, and the one to rule out first: the level-triggered TLS retry spin.** A TLS output has a connection that the peer has abandoned without a clean close — the socket stays `ESTABLISHED` with an empty receive queue. Then:

```
 SSL_read()  ──▶  read()  ──▶  EAGAIN            "no data right now"
        │
        ├── the code re-arms the fd for READ ── but it is ALREADY registered, so
        │   this is a no-op
        │
        ├── flb_coro_yield()  ── returns IMMEDIATELY, because epoll is
        │   LEVEL-TRIGGERED and the fd is still reported ready
        │
        └── goto retry_read  ──▶ ~8,500 iterations/second, forever
```

The loop never blocks and never yields usefully, so the thread pins one core and every flush behind it stops. Reported profiles match the fingerprint exactly: `strace -c` showing ~17,000 `read()` calls in two seconds all returning `EAGAIN`, `output_proc_records_total` frozen, `output_errors_total` at **0**, and a socket whose last receive was days earlier — a process stuck for three days without a single log line.

The mitigation is embarrassingly cheap and available on every version: **`net.io_timeout` defaults to `0s`, meaning no timeout at all.** Set it to something finite on every network output and the loop terminates with an error instead of spinning. Errors are recoverable; spins are not. (Upstream's eventual fix was architectural — coroutine-based event dispatch — but the generic retry-loop patch was never merged, so don't wait for a version bump.)

Note how cheaply this is distinguished from the confirmed mechanism above, and do the measurement before assuming either. Both pin one core and both report zero errors, but this one is **syscall-heavy** (roughly 8,500 `read()` calls a second) while the list-walk loop makes none. The user/kernel CPU split tells them apart in three seconds, and the socket state corroborates it. Set `net.io_timeout` regardless, because it costs nothing and closes a real class — just don't expect it to fix a loop that never touches a socket.

The rest of the family, briefly, because recognising the shape is what matters:

| Class | Signature | Where the fix lives |
|---|---|---|
| Stale connection node left in the event loop's ready queue after teardown | `epoll_wait` returns constantly, zero `recv`, socket in `CLOSE_WAIT` | upstream fix; bounded event-per-round handling |
| Internal message channel saturation | engine blocked in `write()`, CPU **~0%**, HTTP still answers | reduce input/output fan-out on one instance |
| Threaded input plus in-flight processors registered on the wrong event loop | 100% CPU essentially immediately after start | version-specific; avoid the combination |
| Long lines / multiline reassembly | CPU pegged for hours, then all inputs stop; sometimes memory growth to OOM | `Skip_Long_Lines`, sane `Buffer_Max_Size` |
| Filesystem-buffer chunk corruption or counter underflow | pod Running, forwards nothing, "no available chunk" forever | version pin; clear the buffer directory |
| Reload / shutdown loops | 100% CPU right after a `SIGHUP` or during termination | version pin |

Two adjacent failures that are *not* hangs but present almost identically from a dashboard, and are worth ruling in or out early:

**Permanent credential failure.** A single token-refresh error, then every flush fails with 401/403 forever. The process keeps running, `in_tail` keeps reading, and it never self-recovers — restart is the only fix. Distinguishing feature: `output_errors_total` is climbing, so it's the one member of this family an error-rate health check *would* catch.

**Silent offset-persistence failure.** `SQLITE_BUSY`/`SQLITE_LOCKED` on the tail offset database can fail without logging anything, so offsets stop being persisted. Shipping continues normally until a restart, at which point the agent re-reads from stale offsets and you get a burst of **duplicates** rather than loss.

### Prevention beyond the probe

- **Set `net.io_timeout` on every network output.** It defaults to `0s` (no timeout), which is what lets an abandoned connection spin a core forever instead of erroring out. This is the single highest-value config change in this note.
- **Cap the work per flush.** Unbounded `Mem_Buf_Limit` with a slow output invites the pathological states where hangs are found. Bound the buffer, enable filesystem storage, and let backpressure be *visible* (`pause`/`resume` events are logged) instead of unbounded.
- **Give `in_tail` room across rotations.** `Rotate_Wait` is how long the plugin keeps watching a rotated file for pending data; too small and every rotation clips the tail of a file, which looks like random gaps rather than an outage. Combined with `db` for durable offsets, it's what makes agent restarts non-lossy.
- **Move network I/O off the main loop.** Output plugins take `workers N` (dedicated threads per output); input plugins take `threaded: on`. Either way a slow or wedged destination can no longer stall the inputs, which shrinks the blast radius of a stall from "the node" to "one destination" — worth it for anything that talks to the network.
- **Pin the agent version and read the changelogs.** Hang and segfault classes cluster in specific releases; a managed DaemonSet upgrading itself under you is a real change to your reliability posture.
- **Set memory limits deliberately.** An OOM kill is a *good* outcome compared to a hang: it's loud, it restarts, and it shows up in `restartCount`. Fail-fast beats fail-silent.

---

## Failure 2 — segfaults in the metrics path, and the decoy they create

### The crash: SIGSEGV inside the metrics exporter, on the engine's own timer

The second mechanism produces all the visible symptoms and almost none of the impact. Its backtrace has a very recognisable shape:

```
 cfl_list_size()                      cfl_list.h          ← dereferences a corrupt list
   copy_label_values()                cmt_cat.c
     copy_map() / copy_counter()
       append_context()
         cmt_cat()                                        ← concatenating metric contexts
           flb_me_get_cmetrics()      flb_metrics_exporter.c
             collect_metrics()
               flb_me_fd_event()                          ← the 1-second metrics timer
                 flb_engine_handle_event()
                   flb_engine_start() flb_engine.c        ← on the ENGINE thread
```

Read naively, that says "the metrics exporter is buggy." Read the lines immediately *before* it and the story inverts:

```
[error] [output:...] unable to parse token body
[error] [output:...] cannot retrieve oauth2 token
[ warn] [engine] failed to flush chunk ... retry in 7 seconds
[engine] caught signal (SIGSEGV)
```

The credential-refresh path failed seconds before the fault, and there is a documented upstream report linking malformed HTTP requests to the metadata endpoint during token renewal to crashes. But that is a *prelude*, not a proven cause: fleet-wide, crashes outnumbered token-refresh errors by more than ten to one, so most segfaults had no such lead-in. What holds either way is the shape: the metrics timer walks *every* metric structure once a second, so it is the first thing to dereference whatever damage exists. It's the canary, not the gas leak.

> A periodic full-structure walk is a corruption **detector**. The top of the stack names whoever tripped over the damage, not whoever caused it. Read the lines before the backtrace, not the frames in it.

### The crash and the hang are one defect

This is the correction that reorganises the whole note, and it came out of getting a thread stack from a live hung pod.

The hung engine thread was spinning in `cfl_list_size`, reached through `copy_label_values ← copy_map ← copy_counter ← append_context ← cmt_cat ← flb_me_get_cmetrics ← collect_metrics ← flb_me_fd_event ← flb_engine_start`. Compare that against the crash backtrace above. **It is the same chain, frame for frame.** Same function, same call path, same thread, same one-second timer.

So these are not two adjacent bugs. It is one **use-after-free**, and which symptom you get is decided by heap-layout luck:

```
        a metrics label node is freed while still linked into its list
                              │
                the chunk is later reallocated as …
                              │
         ┌────────────────────┴────────────────────┐
         ▼                                         ▼
  something that leaves garbage           a valid node in a DIFFERENT,
  where `next` used to be                 perfectly healthy ring
         │                                         │
         ▼                                         ▼
  the next dereference hits an            every dereference succeeds, but the
  unmapped address                        walk never meets its own head
         │                                         │
         ▼                                         ▼
      SIGSEGV                               INFINITE LOOP
  exit 139, restart, seconds of loss    one core, no restart, months of loss
  LOUD · COUNTED · SELF-HEALING          SILENT · UNCOUNTED · PERMANENT
```

Two operational conclusions follow, and both invert how the incident was originally triaged.

**The crash rate is a leading indicator of the hang risk, not cosmetic noise.** A fleet doing thousands of segfaults a day inside a list walk is a fleet that will accumulate hung pods at whatever rate the allocator decides. Monitor it as a predictor.

**The fix category is lifetime ownership, not error-path hardening.** Unlink before free, or don't free while linked, plus the upstream lifecycle fixes. The free site itself is still **not pinned down**: the live candidates are the dangling-pointer bugs in the token and metadata parsing path (a string buffer reallocated with the old pointer retained) and the map/concatenate lifecycle in the pinned version of the metrics library, which has had a long run of fixes upstream since. Stating that gap is more useful than guessing.

### Retracted: "the error handling scribbled over the list"

An earlier version of this note said the error-handling path had written the string `403` over the list nodes and corrupted them. That is wrong, and the way it was wrong is instructive.

It was believable because a memory dump around the corrupted ring contained the string `403` and a logging API hostname, both of which the error path produces. That looks exactly like an error handler spraying response bytes over neighbouring structures.

```
 what we thought                        what it actually was
 ───────────────────────────────────    ────────────────────────────────────────
 the error path wrote "403" over        "403" is a LEGITIMATE LABEL VALUE. the
 the list nodes and corrupted them      plugin declares requests_total{status,
                                        name} and fills status from snprintf of
                                        the HTTP status. the first 403 response
                                        created a real new metric with a real
                                        label node holding the string "403".

 the nearby hostname string is          it is HEAP-NEIGHBOUR NOISE. the error
 more evidence of the overwrite         path allocated several strings at about
                                        the same time and they landed in
                                        adjacent chunks in the same arena.
```

Two lessons worth more than the specific bug. **Read the source for the label keys before theorising about the bytes**, because we mistook a feature for wreckage. And **adjacency in a memory dump is not causation**, ever; error paths allocate, so error-path strings will always be near whatever else was allocated then.

What settled it was what the dump did *not* show. A scribble leaves misaligned values, non-canonical addresses, half-overwritten pointers. Every pointer in that ring was well-formed, correctly aligned, and mutually consistent, forming a valid closed ring with valid `prev` links. **Well-formed structures encoding a wrong graph is the signature of reuse, not of overwriting.**

Two structural consequences worth internalising:

**Telemetry code is the crashiest part of a log agent, and it has nothing to do with logs.** Beyond this cmetrics path, the recurring SIGSEGV reports cluster in the metrics and events *inputs*: use-after-free on a reallocated HTTP response buffer in the Kubernetes-events input, NULL metric names from an OpenTelemetry histogram decoder, pointer arithmetic on a diskstats cache, `readdir()` on a process that exited. If an instance's job is shipping logs, don't also make it scrape node metrics, Kubernetes events, and Prometheus endpoints. Run telemetry in a separate deployment so a crash in the metrics path cannot take the log path with it.

**A poison chunk survives the restart.** If the crash happens while decoding buffered data, the same chunk is replayed on startup and the process crashes again — a genuine crash loop with an external cause, not a flapping process. That's the case for `storage.delete_irrecoverable_chunks` (and for checking whether your version discards corrupt chunks at startup) rather than staring at restart counts.

And the cruel juxtaposition that made the original incident confusing: crashes like this **do** restart the container, so they're loud, counted, and self-healing. Hangs are silent, uncounted, and permanent. Run a large fleet and you get both at once from the *same* cause — and the one that gets all the attention is the one that's fixing itself. Any time a self-healing failure sits next to a non-self-healing one, expect the investigation to anchor on the wrong one.

### The decoy: restart counts belong to containers, not pods

Node logging DaemonSets are multi-container by design. GKE's, for example, runs the tailing agent, a separate exporter that makes the actual Cloud Logging calls, and a `fluentbit-metrics-collector` that scrapes `127.0.0.1:2020` — the last of which has **no probes at all** and sits on no log path whatsoever.

```
 DaemonSet pod (kube-system)
 ├── container: agent              ← does the tailing.       HUNG.     restarts: 0
 ├── container: exporter           ← makes the API calls.              restarts: 0
 └── container: metrics-collector  ← publishes agent metrics. SEGFAULT. restarts: 214
```

Pod-level views, and most dashboards, surface a single restart number per pod. `214 restarts` on the logging DaemonSet reads as "we already know what's wrong," which is an anchoring trap: a loud, irrelevant failure sitting next to a silent, total one. The container is the restart unit in Kubernetes, so any view that aggregates restarts to the pod discards exactly the field you need:

```bash
# Per-container restart counts -- always look at this shape, never the pod total.
kubectl get pods -n kube-system -l k8s-app=fluentbit-gke -o custom-columns=\
'POD:.metadata.name,NODE:.spec.nodeName,C:.status.containerStatuses[*].name,R:.status.containerStatuses[*].restartCount'

# Why did it die? 139 = 128 + 11 => SIGSEGV.
kubectl get pod -n kube-system "$POD" -o jsonpath=\
'{range .status.containerStatuses[*]}{.name}{": "}{.lastState.terminated.reason}{" exit="}{.lastState.terminated.exitCode}{" sig="}{.lastState.terminated.signal}{"\n"}{end}'
```

Exit-code arithmetic is worth memorising, because it's the fastest way to classify a dead container:

| Exit code | Signal | Meaning |
|---|---|---|
| 137 | SIGKILL (9) | OOM-killed, or liveness-probe kill, or a graceful shutdown that overran |
| 139 | SIGSEGV (11) | Segfault — a memory-safety bug in the binary, not your config |
| 143 | SIGTERM (15) | Normal shutdown that didn't finish inside the grace period |
| 1 / 2 | — | Application-level exit: bad config, missing credentials |

`OOMKilled` and `Error/139` both terminate a container and mean opposite things about who owns the fix. A 137 is usually yours (limits, buffer sizing). A 139 is usually upstream's, and your levers are version pinning or not running the offending input.

One more thing the memory numbers tell you. Hung agents in the incident sat at 10–200 MiB against a 512 MiB limit, so the OOM killer — the one mechanism that would have recovered them for free — was never anywhere near firing. A hang is not a resource problem, which is why resource-based guardrails don't catch it.

### The deeper harm: the instrument shared the failure domain

This is the part that generalises furthest. That metrics container's job was to publish the agent's throughput counters. While it was crash-looping, those counters were **absent** — so the very series a progress alert would evaluate had no data. And a rule written `rate(...) == 0` **does not fire on absent data**; absent series are simply not evaluated.

```
 agent hung           ──▶ throughput = 0        ──▶ alert should fire
 exporter crash-loop  ──▶ throughput = ABSENT   ──▶ nothing to evaluate  ✗
```

The alert that would have caught the hang was silenced by an unrelated bug in its own reporting path. That's a design flaw, not bad luck: both containers live in the same pod, on the same node, sharing the same scheduling fate.

The managed-platform version of this problem is worse still. On GKE there is **no user-visible per-node metric** for the logging agent's throughput: `logging.googleapis.com/log_entry_count` carries `log` and `severity` labels but no node dimension, and the per-node `kubernetes.io/node/logs/input_bytes` is input-side only. The agent's own `input_records_total` / `output_proc_records_total` equivalents are collected into Google-internal metric names you can't query. So the only way to detect a stalled managed agent on a specific node is to scrape `:2020/api/v1/metrics/prometheus` yourself, per node, and store it. If nobody has built that, a hung managed agent is undetectable by construction.

### What to do about it

- **Alert on absence, not just on zero.** For any series that should exist on every node, no-data is a first-class alert:

  ```promql
  # No agent throughput series for this node for 15m -- covers exporter death too.
  absent_over_time(fluentbit_output_proc_records_total{node="..."}[15m])
  ```

  You want both rules, at the *same* severity, because "it's broken" and "I can't tell" have identical consequences for a log pipeline.

- **Alert per container on restarts, and route the two questions differently.** Growth on a telemetry container is a bug report. Growth on the agent is an outage.

- **Don't co-locate the instrument with the subject.** The end-to-end canary described later is deliberately *outside* the agent's pod: it measures the pipeline's output rather than the pipeline's self-report. Self-reported health is always suspect when the reporter shares a failure domain with the reported.

- **Segfault triage, when you need to escalate.** A 139 is reproducible evidence if you collect it. The node's `dmesg` records the fault address and faulting object, which is what an upstream issue needs — along with the log lines from *before* the crash, which are usually the actual cause.

  ```bash
  kubectl debug node/"$NODE" -it --image=ubuntu --profile=sysadmin -- \
    bash -c 'dmesg -T | grep -Ei "segfault|general protection|traps:" | tail -20'
  # exporter[12345]: segfault at 0 ip 00007f... sp 00007ff... error 4 in libX.so

  # And the previous container's logs -- where the real cause is.
  kubectl logs -n kube-system "$POD" -c "$CONTAINER" --previous | tail -50
  ```

  `segfault at 0` is a null dereference; `error 4` is a userspace read fault. With the image tag, that's a real bug report. The mitigation, meanwhile, is almost always to stop running the input that faults, or pin to the last good version.

## Failure 3 — the sink that diverts on schema mismatch

Now the failure that lives entirely in the cloud half, produces the same user-visible complaint ("rows missing in BigQuery"), and has nothing to do with the node.

### How a BigQuery log sink builds its schema

A sink is one inclusion filter, up to 50 exclusion filters, and a destination. For a BigQuery destination the Log Router maps each `LogEntry` onto table columns, and one rule governs everything that follows:

> The **first log entry** received by BigQuery determines the schema of the destination table.

Everything after that is evolution against that initial guess. Field names are rewritten on the way in, and the rewrite rules are lossy:

| Rule | Example |
|---|---|
| Non-alphanumeric characters become `_` | `jsonPayload.foo%%` → `jsonPayload.foo__` |
| Leading underscores are stripped | `_id` → `id` |
| Field names capped at **128** characters | (BigQuery itself allows 300) |
| `LogEntry`-native fields keep their names exactly | `httpRequest.status` → `httpRequest.status` |
| **User-supplied** field names are lowercased | `jsonPayload.MESSAGE` → `jsonPayload.message` |
| A payload `@type` renames the whole top-level column | `jsonPayload` + `@type: type.googleapis.com/abc.Xyz` → `jsonpayload_abc_xyz` |

Two traps fall out of that table. Sanitisation is **lossy**: `foo%%` and `foo__` both become `foo__`, and `MESSAGE` and `message` both become `message`, so two distinct application fields can end up fighting over one column — and the docs don't say who wins. And the `@type` row is nastier than it looks, because a service that *starts* emitting `@type` mid-stream renames its entire payload column tree; your dashboards keep querying a column that has quietly stopped receiving data.

Then the actual failure. The sink can **add** columns as new fields appear, but it can never **retype** an existing one — BigQuery has no in-place retyping path beyond `ALTER COLUMN SET DATA TYPE` for a narrow set of widening conversions. So:

```
 entry 1   jsonPayload.user_id = "u-4417"   ──▶ column user_id : STRING   (locked, forever)
 entry 2   jsonPayload.user_id = 4417       ──▶ INTEGER into a STRING column   ✗
 entry 3   jsonPayload.user_id = ["a","b"]  ──▶ ARRAY   into a STRING column   ✗
```

Exactly two conditions count as a schema mismatch:

1. A later entry **changes the type** of an existing field.
2. A new entry introduces a field that would push the table past BigQuery's **10,000-column** limit — and *deleted columns keep counting toward it*, which is a wonderful way to exhaust a limit you believe you're nowhere near.

### The rows aren't lost — they're in a table you've never queried

This is the part almost nobody knows, and it changes the entire incident response:

> On a schema mismatch the sink creates an **`export_errors`** table in the same dataset (`export_errors_YYYYMMDD` for date-sharded sinks) and writes the offending entries **there** instead of to your table.

It is a dead-letter table. Nobody ever looks at it, which makes the failure *operationally* silent even though it is technically well-instrumented.

The batching rule decides how much collateral damage each mismatch causes, and the asymmetry is severe:

| Mismatch cause | What lands in `export_errors` |
|---|---|
| **Field type change** | only the entries that actually conflict; the rest of the batch reaches the real table |
| **Column limit exceeded** | **the entire batch**, including entries that were perfectly fine |

So one application field flipping from string to int produces a thin, continuous trickle of diverted rows that nobody notices for months. One team using user IDs as JSON *keys* until the table hits 10,000 columns takes out every entry in every batch it appears in.

The error table's schema is a ready-made diagnosis: `logEntry` (the whole entry, JSON flattened into a string), `schemaErrorDetail` (BigQuery's own error message, passed straight through), `sink`, plus extracted `logName`, `timestamp`, `severity`, `insertId`, `resourceType` and `trace`. And the first-entry rule applies recursively — the first error defines the error table's schema too.

```sql
-- Highest-fidelity detector available, and unlike the error log it isn't rate-limited.
SELECT timestamp, logName, resourceType, schemaErrorDetail, insertId
FROM   `project.dataset.export_errors`          -- or export_errors_YYYYMMDD
WHERE  timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)
ORDER  BY timestamp DESC
```

### Why this looks nothing like the node failure

This is the discriminator table I reach for first:

| | Hung node agent | Sink schema mismatch |
|---|---|---|
| **Blast radius shape** | one **node**, all its pods | one **field/service**, all nodes |
| **Completeness** | total: 100% of that node | partial: the conflicting entries — or whole batches, on column overflow |
| **Logs Explorer** | also missing (never ingested) | **present** — ingestion succeeded |
| **Onset** | a sharp cliff at one timestamp | begins with a deploy that changed a log line |
| **Signals** | agent throughput → 0 | rows in `export_errors`, `exports/error_count`, a `sink_error` entry |
| **Recoverable?** | no — rotation ate the files | yes — the entries sit in `export_errors` *and* the log bucket |

That third row is the single most valuable query in a log incident: **is the entry in Logs Explorer?** It bisects the pipeline precisely at the ingestion boundary. Present means the node half worked and the problem is a sink. Absent means the problem is upstream of ingestion and the sink is innocent.

The last row is why this failure, though embarrassing, is far less serious. The bytes still exist in two places, so a schema mismatch is a repair job. A node-side loss is not, once rotation has cycled.

### Detecting it, and the three ways detection lets you down

The signals, best first:

```bash
# 1. The error table -- per-entry, immediate, not throttled. Query it (above).

# 2. The metric. DELTA, resource type logging_sink, zero baseline.
#      logging.googleapis.com/exports/error_count      "log entries that failed to be exported"
#      logging.googleapis.com/exports/log_entry_count  "log entries that were exported"
#      logging.googleapis.com/exports/byte_count
#    Monitoring filter:
#      metric.type="logging.googleapis.com/exports/error_count"
#      resource.type="logging_sink"
#      resource.label."name"="SINK_NAME"

# 3. The sink error log entry.
gcloud logging read \
  'logName:"logging.googleapis.com%2Fsink_error" AND resource.type="logging_sink"' \
  --freshness=7d \
  --format='table(timestamp, labels.sink_id, labels.error_code, labels.error_detail)'
# textPayload: "Cloud Logging sink configuration error in <PROJECT>, sink <SINK>: <error_code> (<detail>)"

# 4. Sanity-check the sink itself: filter, destination, writer identity.
gcloud logging sinks describe "$SINK"
# Writer identity for sinks created on/after 2023-05-22 is shared:
#   serviceAccount:service-<PROJECT_NUMBER>@gcp-sa-logging.iam.gserviceaccount.com
# and it needs roles/bigquery.dataEditor on the dataset.
```

Now the three ways this instrumentation quietly fails you, all of which are documented and all of which surprise people:

**Sink errors are reported once per day, per error group.** Both the `sink_error` log entry and the email that Essential Contacts (or, failing that, project owners) receive are throttled. So the error log is a *notification*, never a volume signal — you cannot count it, rate it, or use its absence as evidence that nothing is wrong.

**`exports/*` metrics don't exist for folder- or organization-level sinks.** Aggregated sinks are a metric blind spot. Combine that with the once-a-day throttle and an aggregated sink can hit quota and **stop routing entirely** while producing one email a day and no metric whatsoever. That's the sharpest operational trap in this entire area.

**And the metrics lag.** They're sampled every 60 s and may not be visible for up to 420 s, so any alert needs a window comfortably wider than seven minutes or it will flap on delivery delay rather than on real errors. A ratio condition (`exports/error_count / exports/log_entry_count`) also beats a raw threshold on a high-volume sink, where a small constant error floor can mask a step change.

One more honest caveat: whether entries diverted to `export_errors` also increment `exports/error_count` is not documented either way. Measure it in your own project before you rely on it — which is a good general habit with export metrics.

### The other cloud-side stop: quota

Worth separating from schema mismatch because it presents completely differently. Export and destination quotas both apply, and the documented behaviour is blunt:

> If quota is exhausted, then a sink **stops routing** log entries to its destination.

For BigQuery destinations the binding constraint is the streaming insert quota, and exhaustion writes a `table_resource_exhausted` error code. (Logging describes this as a *per-table* streaming quota; BigQuery's published streaming limits are per project — 1 GB/s in the `us`/`eu` multi-regions, 300 MB/s elsewhere — so treat the per-table number as undocumented.) Two more shapes in the same family: entries whose timestamps fall outside a partitioned table's permitted time boundaries are rejected, and entries older than the retention period or **more than 24 hours in the future** are discarded by the Log Router before any sink sees them. Clock skew on a node is a log-delivery bug.

This produces a **project-shaped** total stop — as complete as the hung agent, but across every node at once. Same triage question ("is it in Logs Explorer?"), different answer path.

### Fixing and preventing it

Immediate remediation, in order of preference:

- **Fix the emitter.** The documented answer is to make the field type match the existing schema. The root cause is nearly always one application emitting a field as both scalar and object, or both string and number. Fixing it there fixes every consumer, not just BigQuery.
- **Recreate the table when you can't change the type** — which is the case for logs generated by Google Cloud services themselves. The documented workaround is to rename the table or change the sink's parameters so Logging recreates it. Mind the re-seeding hazard: because the *first* entry defines the schema, recreating the table just re-rolls the dice unless the first entry through is representative.
- **Backfill.** The diverted entries are sitting in `export_errors` with the full entry in `logEntry`, and the originals are still in the log bucket. Both are replayable into the fixed table.
- **Shrink the stream** if the trigger was volume-adjacent: tighten the inclusion filter, or sample deterministically with `sample(insertId, 0.25)`.
- **Stringify the volatile part.** If a payload genuinely carries user-controlled shapes, emit it as one JSON *string* field and parse at query time. One STRING column absorbs any shape; a tree of inferred columns cannot.

Structural prevention:

- **Don't derive a warehouse schema from unvalidated application output.** Auto-inference is a convenience feature that silently promotes every log-format change into a data-pipeline change.
- **Prefer a log bucket with analytics enabled plus a linked BigQuery dataset** over a classic sink. This is now Google's own recommendation, and it deletes the failure class rather than mitigating it: the schema is the fixed `LogEntry` shape, and `json_payload` is a native BigQuery **JSON** column — so a field that's a string in one entry and an object in another coexist happily, with no inference and no diversion. Nothing is copied, so there are no BigQuery ingestion or storage costs, only analysis costs when you query. The trade-offs are real and worth stating: the upgrade is **irreversible**, backfilling pre-upgrade entries can take days, a bucket can be linked to at most one dataset, you query through `JSON_VALUE(...)` and `CAST` instead of native typed columns, a linked dataset **can't** be a sink destination, and field-level access controls are **not honoured** through the linked dataset — that last one is a security decision, not a footnote.
- **Watch the table's shape, not just its size.** 10,000 columns per table (deleted columns still count), and Logging caps nesting at 13 levels against BigQuery's 15. A payload with unbounded key names — IDs used as map keys is the classic — grows columns until batches start failing wholesale. Cardinality in *keys* is a schema problem, not a volume problem.

## Triage: four questions, in this order

Every incident in this space resolves to "where did the bytes stop?" Ask in pipeline order and stop at the first `no`:

```
                    ┌─────────────────────────────────────────┐
 1. kubectl logs?   │ Does the app still produce output, and   │
                    │ is the runtime writing /var/log/pods?    │
                    └───────────────┬─────────────────────────┘
                            no ─────┴───── yes
                             │              │
              app / runtime  │              ▼
              problem        │   ┌─────────────────────────────────────┐
                             │   │ 2. agent input counters climbing?   │
                             │   └──────────────┬──────────────────────┘
                             │          no ─────┴───── yes
                             │           │              │
                             │  tail-side│              ▼
                             │  (perms,  │   ┌──────────────────────────────┐
                             │  offsets, │   │ 3. agent OUTPUT counters     │
                             │  hang)    │   │    climbing? errors? retries?│
                             │           │   └────────┬─────────────────────┘
                             │           │     no ────┴──── yes
                             │           │      │            │
                             │           │  HANG│(0 err)     ▼
                             │           │  or  │   ┌─────────────────────────┐
                             │           │  auth│   │ 4. visible in Logs      │
                             │           │  quota   │    Explorer?            │
                             │           │  (errors)└────────┬────────────────┘
                             │           │             no ───┴─── yes
                             │           │              │         │
                             │           │      ingest/ │         ▼
                             │           │      IAM/    │   SINK problem:
                             │           │      quota   │   filter, IAM,
                             │           │              │   or schema drop
                             ▼           ▼              ▼   (exports/error_count)
```

The mapping from observed shape to cause, which is where experience actually lives:

| What you observe | Almost certainly |
|---|---|
| One node, all pods, hard stop, no errors | Agent hang (Failure 1) |
| One node, all pods, gaps that resume | Agent restarts / OOM / rotation overrun |
| One node, partial gaps only during traffic bursts | Backpressure: input paused on `Mem_Buf_Limit`, rotated files purged after `rotate_wait` |
| All nodes, one service, partial | Payload/schema issue, or a sink filter |
| All nodes, everything, hard stop | Ingest-side: quota, revoked IAM, sink deleted |
| Present in Logs Explorer, absent in BigQuery | Sink: filter mismatch, permissions, or a schema mismatch diverting entries to `export_errors` (Failure 3) |
| Missing everywhere including Logs Explorer | Node half — never shipped |
| Only large log lines are wrong | CRI partial-line reassembly (`P` tags) |

---

## The one control that would have caught all three

Everything above is diagnosis. If you only get to build one thing, build a **synthetic log canary** measured end-to-end at the destination:

```
 DaemonSet: one tiny pod per node
   every 60s: emit a structured line
     {"canary":true,"node":"<NODE_NAME>","seq":<n>,"emitted_at":"<RFC3339>"}

 Then, in the warehouse (the actual destination, not an intermediate hop):

   SELECT node,
          TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), MAX(timestamp), MINUTE) AS lag_min
   FROM   `project.dataset.canary_log`
   WHERE  timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 2 HOUR)
   GROUP  BY node
   HAVING lag_min > 10          -- alert
```

Why this is worth more than any single component check:

- It traverses **every hop**: file write, rotation handling, tail, filter, buffer, output, ingest, router, sink, schema. A hang, a crash-loop, a dropped entry, a deleted sink, and an IAM regression all surface as lag.
- It is **per node**, so node-shaped failures can't hide inside a fleet aggregate. This is the property that distinguishes it from every dashboard that says "logs look fine."
- It has a **fixed, trivial schema**, so the canary itself is immune to the schema-mismatch class it's monitoring.
- It measures **freshness, not volume**, so it works on quiet nodes and doesn't need a traffic-dependent threshold.
- It lives **outside** the agent's pod, so it cannot be silenced by the same failure it's watching — which is precisely how Failure 2 defeated the self-reported metrics.

Add the two cheap component-level rules alongside it (`min by (node)` throughput → 0, and `exports/error_count > 0`) and you have the whole surface covered: the canary tells you *something* broke end-to-end, and the component rules tell you *which hop*.

---

## Interview Prep

### Q: Walk me through everything that happens between a `println` in a container and a row in BigQuery.

**A:** Seven hops, and it's worth naming the owner of each because that's how you triage later.

1. The app writes to fd 1/2. Those are pipes held by the container runtime, not files.
2. The runtime's CRI logger reads the pipe and writes `<RFC3339Nano> <stdout|stderr> <F|P> <message>` lines to `/var/log/pods/<ns>_<pod>_<uid>/<container>/<restartCount>.log`, with a symlink farm at `/var/log/containers/` carrying pod, namespace and container in the filename. A line longer than the runtime's read buffer is split into `P` fragments ending in an `F`.
3. The kubelet rotates those files — `containerLogMaxSize` (default `10Mi`), `containerLogMaxFiles` (default 5), checked every `containerLogMonitorInterval` (10s). Mechanically it's **rename plus CRI `ReopenContainerLog`**, not copytruncate, so a tailing agent keeps draining the old inode through its open fd. What actually destroys data is the deletion of files beyond the retained count.
4. A log-agent DaemonSet tails the symlinks. `in_tail` keeps byte offsets in a SQLite `db` so it resumes across restarts, rejoins `P` fragments, and a filter enriches records with Kubernetes metadata from the API server, cached per pod.
5. Records accumulate into chunks in a buffer — memory by default, `storage.type filesystem` for durability — and an output plugin batches them out, with retry and exponential backoff driven by the engine's scheduler. On a managed platform there may be an extra hop *inside the pod*: GKE's `fluentbit` container's only output is an `http` POST of msgpack to `127.0.0.1:2021`, and a second container in the same pod makes the gRPC `WriteLogEntries` calls.
6. Cloud Logging ingests, applies quota and IAM, and hands each entry to the **Log Router**, which evaluates every sink independently — `_Required`, `_Default`, and any user sinks.
7. A BigQuery sink maps the entry onto a table: envelope fields plus `jsonPayload` expanded into columns, typed from the first entry that introduced each field.

The two boundaries that matter for debugging are **step 4→5 on the node** — one process per node, so failures are node-shaped and total — and **step 6→7 in the cloud**, where one sink serves every node, so failures are field-shaped and partial.

### Q: Every pod on one node stopped shipping logs, but `kubectl logs` works and another agent on the same node is fine. How do you triage?

**A:** The shape already tells me a lot before I run anything.

*Node-shaped* means the fault is in something with a per-node singleton, which on this path is the agent process. A service-shaped failure would point at the application or the sink filter; a project-shaped one at ingest, quota, or IAM.

`kubectl logs` working proves the app and the runtime are fine and the files are on disk, so the left three hops are eliminated. That's the most commonly misread signal in log incidents — it says nothing about shipping.

The second agent working is the strongest clue, because it's a controlled experiment I didn't have to set up. Both agents share the *substrate*: the same files, same filesystem, same kernel, same node clock, same egress path. They share nothing else — separate tail offsets, separate buffers, separate destinations, separate binaries. So a substrate fault would break both. Only one broke, therefore the fault is internal to that agent (or to its specific destination/credentials), and I can stop investigating disk, permissions, and networking.

Then I bisect with counters: hit `:2020/api/v1/metrics/prometheus` twice thirty seconds apart. If input records are frozen the input side is stuck; if input climbs while output doesn't, it's the output or the engine. Zero errors alongside zero throughput rules out backpressure and auth failures, which are noisy. Check `/api/v1/storage` for chunk accumulation, `top -H` for a single thread at 100%, and `strace -c` to see whether it's spinning in userspace or blocked in a syscall. Capture a stack with `gdb -p` or `eu-stack` **before** restarting, because the restart is the end of the investigation.

### Q: Why didn't Kubernetes restart the hung agent, and what probe would you have written?

**A:** Kubernetes restarts a container on process exit or liveness-probe failure. A hung process does neither, and the kubelet's only knowledge is that the PID is alive.

Three gaps, and I'd walk through all three because interviewers usually expect only the first. Self-managed agent DaemonSets frequently ship with **no probes at all**. Managed ones often *do* have a probe — GKE's is an `httpGet /` on port 2020 — but that endpoint is served by the **monitoring thread**, which is a different thread from the engine, so it keeps answering throughout a hang. A probe that doesn't live on the thread doing the work can't observe the work stopping.

And the purpose-built health endpoint fails in both directions. `Health_Check` defaults to **off**, and when it's off the endpoint isn't registered — so a probe pointed at `/api/v1/health` gets a **404** and restarts a perfectly healthy agent, which is how people conclude the probe is broken and delete it. Turn it on and health is defined as an error-rate predicate: unhealthy only if output errors exceed `HC_Errors_Count` or failed retries exceed `HC_Retry_Failure_Count` within `HC_Period` (defaults 5, 5, 60s). A hang produces zero of both. It also fails open structurally, because the error-count window is filled by a timer *on the engine loop*: when the engine stalls the newest sample freezes, the window delta becomes zero, and it returns `200 ok` forever.

The correct probe tests **forward progress**: read a monotonic counter, compare with the previous reading, fail when it hasn't moved. `periodSeconds: 60` with `failureThreshold: 5` gives about five minutes of grace, which has to exceed the longest legitimate quiet period on that node — otherwise you restart the agent every night at 4am. On genuinely low-traffic nodes I'd make it conditional: fail only when input is advancing while output isn't, which is unambiguous.

Two caveats I'd raise unprompted. Restarting is only safe if buffering is on disk; with memory buffering the probe converts a stall into data loss, so `storage.type filesystem` is a prerequisite rather than an optimisation. And the probe is self-reported health from inside the failing pod, so I'd back it with an external `min by (node)` throughput alert plus an `absent_over_time()` no-data rule.

### Q: What's the difference between a hang that pins a core and a deadlock, and how do you tell from outside the process?

**A:** CPU is the tell, and it's binary enough to be diagnostic on its own. A **busy loop** executes continuously without progressing: on a single event loop that's exactly 100% of one core, flat, with RSS also flat because it isn't allocating. A **deadlock** or a blocking wait sits at ~0% CPU, parked in `futex`, `read`, or `write`.

Then I'd split the busy-loop class further, because two mechanisms with the same CPU shape need completely different fixes, and one number separates them.

The **syscall spin** is the well-known one. An output holds a TLS connection the peer has abandoned without a clean close. `SSL_read` calls `read`, which returns `EAGAIN`. The code re-arms the fd for readability — a no-op, it's already registered — and yields; but because epoll is **level-triggered** and the fd is still reported ready, the yield returns immediately and the code retries. Thousands of iterations a second, forever, with no error ever recorded. The default `net.io_timeout` of `0s` is what makes it unbounded, and a finite timeout turns it into an error.

The **userspace loop** is what the incident actually was, and it makes *no* system calls at all. A once-per-second walk over the agent's own metrics structures entered a circular linked list whose head had been detached from the ring, so `it != head` was true forever. Seven instructions, no calls, no syscalls, one core.

The discriminator is the **user/kernel CPU split**, and it takes three seconds with `cat`. Sample `/proc/<pid>/task/*/stat` twice and diff fields 14 and 15, which are `utime` and `stime` in 100 Hz ticks. The hung engine thread showed `+313 utime, +1 stime` over three seconds: 1.04 cores of pure userspace with 0.3% kernel time. A retry spin doing 8,500 `read()` calls a second would have shown a mountain of `stime`. That one ratio redirected the whole investigation.

From outside: `top -H -p <pid>` separates spin from block instantly. `timeout 5 strace -f -p <pid> -c` classifies it — tens of thousands of `EAGAIN` reads is the TLS spin, a tight zero-timeout `epoll_wait` is an event-loop spin over an fd nobody drains, **an empty table is the positive finding** for a userspace loop, and one blocked syscall at 0% CPU is a deadlock or a saturated internal channel. `ss -tinp` corroborates the spin case: `ESTABLISHED` with an empty receive queue and a `lastrcv` measured in days. For the userspace case, `perf top -p` names the hot symbol, or a `ptrace` sampler (`ATTACH`/`GETREGS`/`DETACH` in a loop) gives you an instruction-pointer histogram — 60 samples all landing in one function is a proof rather than a profile.

Both look identical from Kubernetes' point of view — running, no restarts — which is why a progress-based probe covers both while reachability and error-rate probes cover neither. And I'd flag a third shape I have *not* explained: in the same sweep, 23 of 101 broken nodes sat at exactly 0% CPU rather than one core. That's the deadlock signature, it's a different bug, and I'd rather say so than assume the mechanism I proved for the spinning ones.

### Q: The agent was segfaulting in its metrics-collection code, and a telemetry sidecar had 214 restarts. Was either the cause?

**A:** Neither, and both did damage anyway — which makes this a good question for showing how you separate correlation from cause.

Take the segfault first. The backtrace ended in the agent's own metrics-exporter path, walking cmetrics structures on a one-second timer that runs on the engine thread. That reads as "the metrics code is buggy," and it isn't. The metrics timer copies the entire metrics tree once a second, and copying a metric's label list starts by counting it, so it touches every metric structure in the process every second. It is therefore the first code to dereference any damage anywhere in that tree, regardless of who caused it. The general lesson is that **a periodic full-structure walk is a corruption detector** — the top of the stack names who tripped over the damage, not who caused it, so read the lines before the backtrace.

The lines before it were a token-parse failure and a failed oauth2 refresh, which points at the credential path, and there are known dangling-pointer bugs in the metadata token fetch. But I'd be careful how strongly I state that, and this is where I'd show a correction to my own analysis. I originally claimed the error-handling path had **scribbled the string `403` over the list nodes**, because a memory dump around the corrupted ring contained `403` and a logging API hostname, which looks exactly like an error handler spraying response data over neighbouring structures.

That was wrong twice over. The `403` is a **legitimate label value**: the plugin declares `requests_total{status, name}` and fills `status` from `snprintf` of the HTTP status code, so the first 403 response creates a real new metric with a real label node holding the string `403`. I had mistaken a feature for wreckage. And the hostname nearby was **heap-neighbour noise**, because the error path allocated several strings at about the same time and they landed in adjacent chunks. Adjacency in a dump is not causation.

What settled it was what the dump did *not* show. A scribble leaves misaligned values and non-canonical addresses; every pointer in that ring was well-formed, aligned, and mutually consistent with valid `prev` links. Well-formed structures encoding a wrong graph is the signature of **reuse**, not overwriting. So the mechanism is a use-after-free: a label node freed while still linked, then handed back by the allocator as a label node of the newly created 403 metric, leaving the original owner's head pointing into a ring that no longer contained it.

That correction matters because it changes the fix. An error-path overrun would be fixed with bounds checking in the error path; a use-after-free on a metrics structure is fixed with lifetime ownership, unlink before free, plus the upstream lifecycle fixes. Different patches, different files, different projects. It also means the crash and the *hang* were one defect rather than two, so the segfault rate is a leading indicator of hangs. It still argues for keeping telemetry inputs out of the process whose job is shipping logs, since the recurring crash reports in this codebase cluster in the metrics and events paths, none of which are on the log path.

Then the sidecar. The container is the restart unit in Kubernetes; those 214 restarts belonged to the telemetry container while the agent container's own `restartCount` was 0. So the first move is always `-o jsonpath` over `status.containerStatuses[*]` for per-container counts and `lastState.terminated`. Exit 139 is `128 + 11`, so SIGSEGV — a memory-safety bug, not a config or limits problem, which matters for ownership: a 137 (OOMKilled) is mine to fix with limits or buffer sizing, a 139 is upstream's and my levers are version pinning or not running the faulting input.

The first harm was anchoring: a loud, visible failure next to a silent, total one absorbs the investigation, and people concluded "we know what's wrong, the logging pod is crash-looping." The second harm was worse. That container published the agent's throughput metrics, so while it was dead the series were **absent, not zero** — and `rate(x) == 0` never fires on absent series. The instrument and the subject failed together, so the only alert that could have caught the hang was silenced by an unrelated bug in its own reporting path. Fixes: alert on no-data at the same severity as on zero, and put the authoritative check outside the pod being measured.

Worth adding that on a managed platform this is structurally worse — there's no user-visible per-node throughput metric for the logging agent at all, so unless you scrape the agent's own endpoint per node yourself, a hung managed agent is undetectable by construction.

### Q: Rows are missing in BigQuery but the same logs are visible in Logs Explorer. Where's the break?

**A:** Downstream of ingestion, in the sink. That question is the cleanest bisection point in the whole pipeline: presence in Logs Explorer proves the node half worked and the entry was ingested and routed, so the node, the agent, its credentials and ingest quota are all eliminated in one query.

Four sink-side causes, roughly by likelihood. The sink's **inclusion filter** doesn't match what you assumed, or one of its up-to-50 exclusion filters does. The **writer identity** lost `roles/bigquery.dataEditor` on the dataset — and note the identity is shared per project now (`service-<PROJECT_NUMBER>@gcp-sa-logging...`) for sinks created since May 2023, so one revoked grant can break several sinks at once. **Quota** exhaustion, which is documented to make a sink *stop routing* entirely and emits a `table_resource_exhausted` error code. Or — the interesting one — a **schema mismatch**.

The schema story is worth telling properly because most people get it wrong. The table's schema is set by the **first entry** the sink ever delivers. After that the sink can add columns for new fields, but it can never retype an existing column. So when a field that was a STRING starts arriving as an INTEGER or an array, that's a mismatch; the other trigger is a new field that would push the table past BigQuery's 10,000-column limit, where deleted columns still count.

And here's the part that changes the response: those entries are **not dropped**. The sink creates an `export_errors` table (`export_errors_YYYYMMDD` when date-sharded) in the same dataset and writes them there, with the full entry in a `logEntry` string column and BigQuery's own error text in `schemaErrorDetail`. It's a dead-letter table that nobody has ever queried, which is why the failure feels silent even though it's well instrumented. The blast-radius difference from a type change versus a column overflow also matters: a type change diverts only the conflicting entries, while exceeding the column limit diverts **the entire batch**, healthy entries included.

Detection, in order of fidelity: query `export_errors` (per entry, immediate, not throttled); alert on `logging.googleapis.com/exports/error_count` for the `logging_sink` resource, with a window wider than seven minutes because those metrics can lag that long; and treat the `sink_error` log entry and its Essential Contacts email as notifications only, since they're throttled to one per error group per day. Two blind spots to name unprompted: `exports/*` metrics don't exist at all for folder- or organization-level sinks, and whether error-table diversions increment `error_count` isn't documented, so I'd measure it rather than assume it.

The fix is to correct the emitter; if the log is generated by a Google service and you can't, the documented workaround is to rename the table or repoint the sink so Logging recreates it — while remembering that the first entry through the new table re-rolls the schema dice. Then backfill from `export_errors` or the log bucket, because unlike a node-side loss the data still exists in two places.

### Q: Design monitoring so that "logs silently stopped" can't happen again.

**A:** I'd build one end-to-end check and two component checks, in that order of priority.

The end-to-end check is a **synthetic canary DaemonSet**: one tiny pod per node emitting a structured line every 60 seconds with the node name and a timestamp, then a query at the *destination* — the BigQuery table, not an intermediate hop — alerting when `max(timestamp)` per node falls more than ten minutes behind now. It's per node so node-shaped failures can't hide in a fleet aggregate, it traverses every hop so it catches hangs, crash-loops, deleted sinks, IAM regressions and schema drops alike, it has a trivial fixed schema so it's immune to the mismatch class it monitors, and it measures freshness rather than volume so it works on quiet nodes. It also sits outside the agent's pod, which is the specific property that would have defeated the correlated-instrument failure.

The component checks tell you *which* hop, once the canary says something is wrong: `min by (node) (rate(output_records[10m])) == 0` paired with `absent_over_time(...)` at the same severity on the node side — and on a managed platform that means scraping the agent's `:2020` endpoint per node yourself, because no user-visible cloud metric carries a node dimension for it — and on the sink side both `exports/error_count > 0` per sink *and* a row-count check on the `export_errors` table. I'd build both because they fail in different directions — the metric doesn't exist for aggregated folder/org sinks, and if a sink breaks at the routing layer the `exports/*` series can go quiet rather than error.

Then the structural fixes so failures are loud instead of silent: a progress-based liveness probe on the agent, `storage.type filesystem` so restarts don't lose buffered chunks, memory limits set deliberately (an OOM kill is a *better* outcome than a hang — loud, self-recovering, and visible in `restartCount`), `workers` on network outputs so one wedged destination can't stall the node's inputs, and per-container restart alerting that routes exporter bugs and agent outages to different places.

### Q: Where can this pipeline lose data, and where can it duplicate?

**A:** Loss and duplication live in different places, and the guarantees change at the ingest boundary.

Loss on the node, which is the only *unrecoverable* kind. Kubelet rotation deletes files beyond `containerLogMaxFiles` — `10Mi × 5` per container by default, and a chatty container can cycle all five in seconds. Note rotation itself is rename-plus-reopen, so the agent can keep draining a rotated file through its open fd; it's the deletion that destroys data.

There's a subtler one on the same path that I'd raise because it explains partial gaps rather than clean cliffs. When an input hits `Mem_Buf_Limit` it pauses, but the pause is selective: the reader stops while the rotated-file **purge** collector keeps running. So once `rotated + rotate_wait` elapses, the file is dropped from the watch list and its offset row deleted, pending bytes and all. The agent even logs it — `purged rotated file while data ingestion is paused, consider increasing rotate_wait` — which is why filtering that warning out of your own log pipeline to reduce noise is a genuinely expensive mistake. Filesystem buffering avoids the whole thing, because the input writes chunks to disk instead of pausing.

Also: memory-buffered chunks die with the process on OOM, crash, or eviction. `Skip_Long_Lines` and botched `P`-fragment reassembly drop or mangle oversized records, and a line exceeding `Buffer_Max_Size` can get the *file* removed from the monitored list. And a hang is the worst case: unbounded loss for its whole duration with no replay source.

Loss before any sink sees the entry: the Log Router discards entries whose timestamps are older than the retention period or **more than 24 hours in the future**, so node clock skew is a log-delivery bug. Ingest quota rejections belong here too.

Loss at the sink, which is mostly recoverable. Schema mismatches divert entries to `export_errors` rather than deleting them. Quota exhaustion stops a sink outright, but the entries remain in whatever bucket also received them — which is the practical argument for keeping `_Default` retention instead of excluding everything into a sink and leaving yourself nothing to replay from. Sinks are also not retroactive, so anything that arrives while a sink is misconfigured is never backfilled by fixing it.

Duplication is the transport's design. The path is effectively at-least-once: a batch that succeeds server-side but whose acknowledgement is lost gets retried, and restarting from a stale `in_tail` offset re-reads the tail of a file. There's a quiet contributor to that second case: lock contention on the offset database can fail *without logging anything*, so offsets silently stop being persisted and the next restart re-reads a large backlog. Duplicates, not loss, and no warning either way. Cloud Logging deduplicates on `(timestamp, insertId)` **at query time** in Logs Explorer; BigQuery does not, so a sink table can legitimately contain duplicates. There's also a lesser-known source: after a Logging-side incident, backfill produces tables prefixed `backfill_` that may overlap with rows already delivered, and the recommended procedure is to merge them into the original table and delete them.

So: at-least-once in the happy path, lossy in specific enumerable failure modes, and duplicate-tolerant by necessity. The durable fix downstream is idempotent reads — deduplicate on `insertId` (or an application event ID) in the query or a materialised view — and the durable fix upstream is an end-to-end freshness canary, because no individual hop's self-report can tell you the chain held.

---

## Key takeaways

- **Blast-radius shape names the layer.** Node-shaped and total means a per-node singleton — the agent. Service- or field-shaped and partial means the payload or the sink. Project-wide means ingest, quota, or IAM. Read the shape before reading logs.
- **`kubectl logs` working proves nothing about shipping.** It reads node-local files through the kubelet. Every failure in this note leaves it healthy.
- **One event loop carries every input, filter and timer, so a single non-yielding coroutine is a node-wide outage.** Silence follows for a precise reason: the log *writer* is its own thread, but log *lines* are produced by the code that stalled. The channel is fine and has nothing to carry — don't read silence as "the logging is broken."
- **CPU flat at exactly one core, zero errors, and uptime climbing while record counters sit still is the hang fingerprint.** That last divergence is structural: uptime is computed in the HTTP thread on request, while counters come from a timer on the thread that stalled. Deadlocks sit near 0% CPU; backpressure shows retries and growing buffers.
- **Health-check endpoints usually measure error rate, not progress — and are often served by a different thread than the work.** `NOT erroring` is not `still working`, and "the HTTP server answers" is not either. Note the double trap: the purpose-built health endpoint is **off by default** (so a probe pointed at it 404s and restarts a healthy agent), and once enabled it fails *open* through a stall. Liveness must assert that a monotonic counter moved inside a bounded window.
- **A hang is worse than a crash.** Crashes are loud, self-recovering, and counted. Prefer designs that die: bound buffers, set memory limits, and add progress probes so stalls become restarts. And note memory limits won't save you here — hung agents sit far below their limit, so the OOM killer never fires.
- **Unbounded I/O waits are what turn a slow peer into a wedged process.** `net.io_timeout` defaults to *no timeout*, and a level-triggered event loop plus an `EAGAIN` retry path turns that into an infinite spin. Setting a finite I/O timeout converts an unrecoverable hang into a recoverable error — the cheapest fix in this note.
- **A periodic full-structure walk is a corruption detector.** When a crash lands in metrics-collection code, the top of the stack names whoever tripped over the damage, not whoever caused it. Read the log lines *before* the backtrace, and keep telemetry inputs out of the process whose job is shipping logs.
- **The crash and the hang here were one defect.** A metrics label node freed while still linked, then reallocated during a `403` burst: a stale pointer into unmapped memory faults (exit 139, self-healing), a stale pointer into a valid ring loops forever (one core, no restart, months of loss). So the segfault rate is a *leading indicator* of hangs, and the fix category is lifetime ownership rather than error-path hardening. The earlier "the error handling scribbled `403` over the list" story is retracted: the `403` was a legitimate label value and the neighbouring strings were heap-adjacency noise. **Adjacency in a memory dump is not causation.**
- **The container is the restart unit.** Pod-level restart counts hide which container failed, and a loud irrelevant crash-loop next to a silent total failure is a powerful anchoring trap. Always look per container, and translate exit codes (137 OOM, 139 SIGSEGV, 143 SIGTERM).
- **Never let the instrument share a failure domain with the subject.** When the metrics exporter died, throughput went *absent* rather than zero, and `== 0` rules don't fire on absent series. Alert on no-data at equal severity, and put the authoritative check outside the pod.
- **Two independent agents on one node is a free controlled experiment.** They share only the substrate. If one works, the substrate is fine and the fault is inside the other binary.
- **Auto-inferred warehouse schemas turn every log-format change into a pipeline change.** The **first entry** sets the table's schema; after that a BigQuery sink can add columns but never retype one. A string-to-number or scalar-to-object change is a mismatch, and so is any new field that would breach the 10,000-column limit (deleted columns still count).
- **Mismatched entries are diverted, not deleted** — into an `export_errors` table in the same dataset. Per entry for a type change, but *the whole batch* on column overflow. It's a dead-letter table nobody queries, which is what makes a well-instrumented failure feel silent.
- **Know how the sink's instrumentation lets you down.** `exports/error_count` is a zero-baseline, high-signal alert almost nobody enables — but it lags up to 420 s, and it **doesn't exist for folder/org-level sinks**. The `sink_error` log entry and its email are throttled to one per error group per **day**, so they're notifications, never volume signals.
- **Sink-side diversions are recoverable; node-side losses are not.** Diverted entries sit in `export_errors` *and* the log bucket. Bytes lost to rotation are gone forever, which is why detection latency on the node half is the number that actually matters — and why excluding everything from `_Default` into a single sink removes the copy you'd replay from.
- **Prefer read-through to copy-in where you can.** A log bucket with analytics enabled plus a linked BigQuery dataset exposes `json_payload` as a native JSON column, so conflicting shapes coexist and the whole mismatch class disappears — with no BigQuery ingestion or storage cost. Pay for it in query-time `JSON_VALUE`/`CAST`, an irreversible upgrade, and field-level ACLs not being honoured through the link.
- **One per-node, end-to-end freshness canary outranks every component check.** Same path, fixed schema, per-node granularity, outside the failure domain.

---

## Related Notes

- [[notes/K8s/when-gauge-sums-lie|When Gauge Sums Lie]] — the other half of "the monitoring is lying to you": interval metadata, ghost series at coarse rollups, and duplicate emitters from leader-elected exporters
- [[notes/K8s/vpa-eviction-loops|The VPA Eviction Loop]] — another failure that persists because the component that would fix it never applies its own output
- [[notes/K8s/kubernetes-autoscaling|Kubernetes Autoscaling]] — node churn, which sets how long a broken agent survives before a rotation accidentally repairs it
- [[notes/Linux/anatomy-of-a-userspace-hang|Anatomy of a Userspace Hang]] — the OS-level companion to this note: what `free()` actually does, why intrusive circular lists have no terminator but their own head, and the `ptrace` sampling and PIE symbolisation that proved the mechanism behind Failures 1 and 2
- [[notes/Linux/linux-fundamentals-architecture-and-debugging|Linux Fundamentals, Architecture & Debugging]] — `strace`, `perf`, `/proc`, thread state, and signals: the toolbox for deciding whether a process is spinning or blocked
- [[notes/K8s/interactive-containers-piping-and-ttys|Interactive Containers, Piping & TTYs]] — what stdout/stderr actually are inside a container, and how the runtime captures them
- [[notes/AuthNZ/oauth-oidc-and-workload-identity|OAuth, OIDC & Workload Identity Federation]] — how the agent authenticates to the logging API, and the credential expiry class of shipping failure
