---
title: "Summary: Alive But Not Shipping"
---

> **Full notes:** [[notes/K8s/node-log-pipeline-silent-failures|Alive But Not Shipping: Node-Level Log Pipelines and Their Silent Failure Modes -->]]

## Key Concepts

### The symptom
- BigQuery missing rows for **every pod on one node**, hard stop at one timestamp, hours long.
- Pods `Running`, `kubectl logs` **fresh**, node `Ready`, agent container `restartCount: 0`, and a **second log agent on the same node still shipping**.
- Three independent faults: a hung agent engine, segfaults in the metrics path (noise + blinded monitoring), and a sink schema mismatch diverting entries to a dead-letter table.

### The pipeline (know every hop and its owner)
- app fd 1/2 → containerd CRI logger → `/var/log/pods/...` (+ `/var/log/containers` symlinks) → kubelet rotation → agent `in_tail` + sqlite offset db → filters → buffer → output → Cloud Logging ingest → **Log Router** → sinks (`_Required` 400d, `_Default` 30d, user) → BigQuery.
- **Node half:** one process per node ⇒ failures are **node-shaped and total**.
- **Cloud half:** one sink per project ⇒ **project-shaped** — usually partial (specific fields/entries), occasionally total (quota exhaustion stops a sink outright).
- Log Router evaluates **every sink independently** ⇒ "in Logs Explorer but not in BigQuery" is a precise localisation, not a contradiction. It buffers transient destination trouble but **not** config errors, and sinks are **never retroactive**.
- **Managed agents add a hop inside the pod:** GKE's `fluentbit` container's only output is `http` → `127.0.0.1:2021`; a second container makes the gRPC `WriteLogEntries` calls; a third (`fluentbit-metrics-collector`) scrapes `:2020`. The DaemonSet runs in addon-manager `Reconcile` mode (your edits get reverted) ⇒ which is *why* teams run a second self-managed agent, and why the "two agents" clue exists at all.

### On-node log files
- Real path `/var/log/pods/<ns>_<pod>_<uid>/<container>/<restartCount>.log` — **not always `0.log`**; symlink `/var/log/containers/<pod>_<ns>_<container>-<id>.log`, truncated to 251 chars + `.log`.
- CRI format: `<RFC3339Nano> <stdout|stderr> <F|P> <message>`. `P` = partial fragment (line exceeded the runtime's read buffer) ⇒ needs `multiline.parser cri`, or big JSON arrives truncated and then breaks sink schemas.
- Kubelet rotation: `containerLogMaxSize` **10Mi**, `containerLogMaxFiles` **5**, `containerLogMonitorInterval` **10s**, `containerLogMaxWorkers` **1**.
- Mechanism is **rename + CRI `ReopenContainerLog`**, *not* copytruncate ⇒ the agent keeps draining the old inode through its open fd. **Deletion** beyond `maxFiles` is what destroys data. `10Mi × 5` per container, cycled in *seconds* by a chatty pod. **No replay** ⇒ a 4-hour stall is unrecoverable.
- **The subtle loss — paused input + purge:** on `Mem_Buf_Limit` the input pauses the **reader**, but **not** the scan/purge collectors. Once `rotated + rotate_wait <= now`, purge drops the file and deletes its offset row — pending bytes included. Warning line: `purged rotated file while data ingestion is paused, consider increasing rotate_wait`.
  - ⇒ `rotate_wait` (default **5s**) must exceed your worst backpressure pause.
  - ⇒ `storage.type filesystem` avoids it entirely (the input writes chunks to disk instead of pausing, unless `storage.pause_on_chunks_overlimit on`). **Memory-only buffering is what turns backpressure into loss.**
  - ⇒ **Never exclude `mem buf overlimit` warnings from your log pipeline** to reduce noise; that line is the backpressure signal.
- **`kubectl logs` proves the app + runtime work. It proves nothing about shipping.** And `kubectl logs -f` doesn't read rotated files, so it can look quiet when it isn't.

### Fluent Bit concurrency model
- **Not single-threaded** — but only one thread advances the pipeline:
  - process main thread (blocks on shutdown), **engine thread** (event loop: input collectors, **all filters**, retry scheduler, **1-second metrics timer**), log thread, HTTP monitoring thread (`:2020`), output worker threads (one per `workers`).
- ⇒ One non-yielding coroutine on the engine loop stalls the **whole node** (cooperative scheduling, no preemption).
- ⇒ `:2020` keeps answering while the engine is dead, and answers *plausibly*:
  - `fluentbit_uptime` / `/api/v1/uptime` = computed at request time in the HTTP thread ⇒ **keeps climbing**.
  - Plugin counters = the **last snapshot the engine pushed**, via a timer *on the stalled loop* ⇒ **frozen**.
  - ⇒ "uptime rising, record counters flat" is structural, not coincidence — the cheapest remote stall signal.
- Silence is *not* a dead logger (that's its own thread). Log **lines** are produced by the code that stalled; the channel is fine and has nothing to carry.
- Defaults to remember: `flush` **1s**; retry backoff `scheduler.base` **5** / `scheduler.cap` **2000** (Nth retry waits a random value in `[base, min(base·2^N, cap)]`); **`Retry_Limit` = 1** ⇒ one retry, then dropped into `output_dropped_records_total`.
- Move network I/O off the main loop: output `workers N`, input `threaded: on`.

### Failure 1 — hung engine
- Fingerprint: **CPU flat at ~1.00 core**, RSS flat and far below the limit (so **no OOM rescue**), `uptime` climbing, input/output counters **frozen**, `errors_total` and `retries_failed_total` **0**, no logs since T, `restartCount` 0.
- Flat-at-one-core = **busy loop**. Deadlock = **~0% CPU**. Backpressure = **retries + growing buffers**.
- **Three reasons nothing restarted it:**
  1. Self-managed DaemonSets often have **no probes at all**.
  2. Managed probes exist but test **reachability on the wrong thread** — GKE uses `httpGet / :2020`, served by the monitoring thread, which stays true throughout a hang.
  3. `/api/v1/health` fails in **both** directions: `Health_Check` defaults **Off** ⇒ the endpoint isn't registered ⇒ a probe gets **404** and restarts a healthy agent; enabled, it's an error-rate predicate (`errors > HC_Errors_Count`=5 **OR** `retry_failures > HC_Retry_Failure_Count`=5 within `HC_Period`=60s) that a hang trivially passes. It also fails **open** structurally: the window is filled by the engine-loop timer, so a stall freezes the newest sample, the delta → 0, and it returns `200 ok` forever. An empty sample list also reads healthy.
- Upstream now *guarantees* the monitoring server stays up while the engine is blocked (there's an integration test asserting it) — correct for monitoring, fatal for liveness.
- ⇒ **Liveness must test forward progress**, not reachability or error rate.

### What actually wedges the engine
- **The level-triggered TLS retry spin (dominant class):** a peer abandons a connection without a clean close (socket `ESTABLISHED`, empty rx queue) → `SSL_read` → `read` → **`EAGAIN`** → re-arm the fd (**no-op**, already registered) → `flb_coro_yield()` returns **immediately** because epoll is **level-triggered** and the fd still reports ready → `goto retry_read` → **~8,500 iterations/s, forever**.
  - Reported evidence: ~17,000 `read()` in 2s all `EAGAIN`; `output_proc_records_total` frozen; `output_errors_total` **0**; socket `lastrcv` ≈ 3.6 **days**; a process stuck 3 days with zero log lines.
  - **Fix available on every version: `net.io_timeout` defaults to `0s` (no timeout).** Set it finite ⇒ the spin becomes a recoverable error. Highest-value config change in the note.
- Other classes: a stale connection node left in the event loop's ready queue after teardown (`epoll_wait` spins, socket `CLOSE_WAIT`); internal channel saturation (blocked in `write()`, **0% CPU**, HTTP still answers); threaded input + processors registered on the wrong event loop (100% CPU immediately); long-line/multiline reassembly (hours at 100%, then all inputs stop, sometimes OOM); fs-buffer chunk corruption or counter underflow ("no available chunk" forever); reload and shutdown spins.
- **Not hangs, but identical on a dashboard:** permanent credential failure (one token error, then 401/403 on every flush forever, never self-recovers — but `errors_total` **climbs**, so an error-rate check *would* catch it); silent offset-persistence failure (`SQLITE_BUSY`/`LOCKED` fails without logging ⇒ **duplicates** after the next restart, not loss).

### Failure 2 — segfaults in the metrics path (and the decoy)
- The crash backtrace ends in the agent's **own** metrics exporter walking cmetrics structures on the **1-second engine timer** (`cfl_list_size → copy_label_values → cmt_cat → flb_me_get_cmetrics → collect_metrics → flb_me_fd_event → flb_engine_start`).
- The lines *before* it name the real cause: `unable to parse token body` / `cannot retrieve oauth2 token` — heap corruption in the **credential-refresh** path (malformed headers to the metadata endpoint).
- ⇒ **A periodic full-structure walk is a corruption detector.** The top of the stack names whoever tripped over the damage, not whoever caused it. **Read the log lines before the backtrace.**
- **Telemetry code is the crashiest part of a log agent and has nothing to do with logs** (Kubernetes-events input use-after-free on realloc, OTLP decoder NULL metric names, node/process exporter pointer bugs). Don't run metrics/events inputs in the instance whose job is shipping logs.
- **A poison chunk survives the restart** — a crash while decoding buffered data replays on startup ⇒ a real crash loop with an external cause. See `storage.delete_irrecoverable_chunks`.
- Crashes **restart** the container: loud, counted, self-healing. Hangs don't: silent, uncounted, permanent. On a fleet you get both from adjacent causes, and attention goes to the one that's already fixing itself.
- **The decoy:** the container is the **restart unit**. Pod-level restart counts hide which container died ⇒ anchoring trap. Exit codes: **137** SIGKILL (OOM / probe kill — usually *your* fix), **139** SIGSEGV (memory bug — usually *upstream's*), **143** SIGTERM, **1/2** app-level.
- **Correlated instrument failure:** the telemetry container published the agent's throughput metrics, so during its crash-loop those series were **ABSENT, not zero** — and `rate(...) == 0` **never fires on absent series**. The only alert that could have caught the hang was silenced by a bug in its own reporting path.
- **Worse on managed platforms:** there is *no user-visible per-node metric* for the GKE logging agent (`logging.googleapis.com/log_entry_count` has no node label; `kubernetes.io/node/logs/input_bytes` is input-side only; the agent's real counters land in Google-internal metric names). Unless you scrape `:2020` per node yourself, **a hung managed agent is undetectable by construction.**

### Failure 3 — BigQuery sink schema mismatch
- **The first log entry the sink delivers defines the table schema.** After that it can **add** columns but **never retype** one (BigQuery has no in-place retype path).
- Two documented mismatch triggers: (1) a later entry **changes an existing field's type**; (2) a new field would breach the **10,000-column** limit — and **deleted columns still count** toward it.
- **Entries are NOT dropped — they're diverted** to an **`export_errors`** table (`export_errors_YYYYMMDD` if date-sharded) in the same dataset. Columns: `logEntry` (whole entry as a string), **`schemaErrorDetail`** (BigQuery's own error text), `sink`, `logName`, `timestamp`, `severity`, `insertId`, `resourceType`, `trace`. A dead-letter table nobody queries ⇒ a well-instrumented failure that *feels* silent.
- **Batching asymmetry:** type change ⇒ **only the conflicting entries** divert; column limit exceeded ⇒ **the entire batch** diverts, healthy entries included.
- Field-name rewriting is **lossy**: non-alphanumerics → `_` (`foo%%` → `foo__`), leading `_` stripped, names capped at **128** chars, user-supplied names **lowercased** (`MESSAGE` → `message`), `LogEntry`-native names preserved. So `foo%%`/`foo__` and `MESSAGE`/`message` collide into one column — and the docs don't say who wins.
- **`@type` trap:** a payload `@type` renames the whole top-level column (`jsonPayload` → `jsonpayload_abc_xyz`). A service that starts emitting `@type` mid-stream forks the table; your queries keep hitting a column that stopped receiving data.
- **Detection, best first:** (1) query `export_errors` — per entry, immediate, **not throttled**; (2) alert `logging.googleapis.com/exports/error_count` (DELTA, resource `logging_sink`; companions `exports/log_entry_count`, `exports/byte_count`); (3) the `sink_error` log entry + Essential Contacts email.
- **Three ways detection lets you down:** `sink_error` entries and emails are throttled to **one per error group per day** (notification, never a volume signal); `exports/*` metrics **don't exist for folder/org-level sinks**; metrics are sampled 60s and can lag **420s** ⇒ alert windows must exceed ~7 min. Whether diversions increment `error_count` is **undocumented** — measure it.
- **Other cloud-side stop — quota:** "if quota is exhausted, then a sink **stops routing**". BigQuery sinks are bound by the streaming insert quota; exhaustion emits **`table_resource_exhausted`**. Project-shaped and total (looks like the node failure, wrong scope). Also: the Log Router discards entries older than retention or **>24h in the future** ⇒ node clock skew is a delivery bug.
- **The router is not a replay buffer.** It buffers transient destination trouble but explicitly **not configuration errors**, and sinks are **never retroactive** — fixing a sink backfills nothing.
- **Fix:** correct the emitter; if it's a Google-generated log you can't change, rename the table / repoint the sink so Logging recreates it (**re-seeding hazard:** the first entry through re-rolls the schema). Backfill from `export_errors` or the bucket. Tighten the filter or `sample(insertId, 0.25)`. Stringify volatile payloads — one STRING absorbs any shape.
- **Structural fix:** log bucket + analytics enabled + **linked BigQuery dataset** (now Google's own recommendation). Schema is the fixed `LogEntry`; **`json_payload` is a native JSON column** ⇒ string-in-one-entry and object-in-another coexist, no inference, no diversion, **no BQ ingestion or storage cost** (analysis cost only). Costs: query via `JSON_VALUE()` + `CAST`, upgrade is **irreversible**, pre-upgrade backfill takes **days**, one dataset per bucket, can't be a sink destination, and **field-level ACLs are not honoured** through the link.

### The one control that catches all three
- **Per-node synthetic canary DaemonSet**: one line per node per 60s, alert on **freshness at the destination** (`MAX(timestamp)` per node > 10 min behind).
- Wins because it is: end-to-end (every hop), **per node** (node-shaped faults can't hide in fleet aggregates), fixed trivial schema (immune to Failure 3), freshness-not-volume (works on quiet nodes), and **outside the agent's pod** (immune to Failure 2).
- Pair with two component rules to localise: `min by (node) rate(output_records[10m]) == 0` + `absent_over_time(...)`, and `exports/error_count > 0`.

## Quick Reference

```text
FAILURE FINGERPRINTS
                      restarts  CPU        counters   errors    uptime
  healthy                 0     spiky      climbing     0       climbing
  HUNG ENGINE             0     1.00 flat  FROZEN       0       climbing  <-- silent
  deadlock / chan full    0     ~0%        FROZEN       0       climbing
  backpressure            0     spiky      slow       >0/retry  climbing
  creds dead (no hang)    0     spiky      in>0 out=0 CLIMBING  climbing
  OOM kill              +1 each spiky      gaps         0       resets
  segfault              +many   n/a        ABSENT       -       resets

  uptime is computed in the HTTP thread; plugin counters come from a timer ON
  THE ENGINE LOOP.  "uptime rising + counters flat" IS the stall signature.

BLAST-RADIUS SHAPE -> LAYER
  one node, all pods, hard stop, no errors  -> agent hang
  one node, all pods, gaps that resume      -> restarts / OOM / rotation overrun
  one node, partial gaps during bursts      -> paused input + rotated-file purge
  all nodes, one service, partial           -> payload type change or sink filter
  all nodes, everything, hard stop          -> ingest quota / IAM / sink deleted
  in Logs Explorer, NOT in BigQuery         -> sink: filter, IAM, quota, or
                                               schema mismatch -> export_errors
  missing everywhere incl. Logs Explorer    -> node half, never shipped
  only large lines wrong                    -> CRI 'P' fragment reassembly

TRIAGE ORDER (stop at first NO)
  1. kubectl logs fresh?          no -> app / runtime
  2. input counters climbing?     no -> tail side (perms, offsets, hang)
  3. output counters climbing?    no -> hang (0 errors) | auth/quota (errors)
  4. visible in Logs Explorer?    no -> ingest / IAM / quota
                                 yes -> SINK: filter, perms, quota, schema

STRACE CLASSIFICATION (the decisive test)
  ~10k+ read/recvfrom per sec, ALL EAGAIN -> level-triggered TLS spin (commonest)
  tight epoll_wait, timeout 0             -> ready-queue spin, fd never drained
  NO syscalls at all, 100% CPU            -> pure userspace loop (parser/filter)
  blocked write/read/futex, ~0% CPU       -> deadlock or saturated channel
  nanosleep + epoll_wait, slow rhythm     -> healthy idle

EXIT CODES   137 = 128+9  SIGKILL (OOM, probe kill)   -> usually YOUR fix
             139 = 128+11 SIGSEGV (memory bug)        -> usually UPSTREAM's
             143 = 128+15 SIGTERM (grace exceeded)
```

**Diagnose a live hang (capture BEFORE restarting)**
```bash
# per-container restarts + last exit -- never trust the pod-level number
kubectl get pod -n kube-system "$POD" -o jsonpath='{range .status.containerStatuses[*]}{.name}{" r="}{.restartCount}{" "}{.lastState.terminated.reason}{" exit="}{.lastState.terminated.exitCode}{"\n"}{end}'
kubectl logs -n kube-system "$POD" -c "$C" --previous | tail -50   # cause is BEFORE the backtrace

# stall or genuinely no input? run twice, 30s apart -- identical = stall
kubectl exec -n kube-system "$POD" -c "$AGENT" -- \
  curl -s localhost:2020/api/v1/metrics/prometheus | grep -E 'records_total|errors_total|retries|uptime'

# buffer draining? chunks piling with zero output = engine not draining
kubectl exec -n kube-system "$POD" -c "$AGENT" -- curl -s localhost:2020/api/v1/storage
#   (requires `storage.metrics on` in [SERVICE])

# which thread burns the core?         one at ~100%, rest idle = busy loop
kubectl exec -n kube-system "$POD" -c "$AGENT" -- top -H -b -n1 -p 1
# no `top` in the image? per-thread CPU from procfs (fields 14/15, sample twice):
for t in /proc/1/task/*; do awk '{print $1, $14, $15}' "$t/stat"; done

# userspace loop or blocked syscall?   THE decisive test
timeout 5 strace -f -p "$PID" -c
ss -tinp | grep -A1 :443       # ESTABLISHED + rx_queue 0 + lastrcv in DAYS = spin
#   no syscalls at all      -> pure userspace infinite loop
#   tight epoll_wait, tmo 0 -> event-loop spin (fd never drained/unregistered)
#   blocked read/futex, 0%  -> deadlock, NOT this class
gdb -p "$PID" -batch -ex "thread apply all bt" | head -60   # or eu-stack -p
perf top -p "$PID"                                          # names the plugin

# sink side
gcloud logging sinks describe "$SINK"          # filter, destination, writer identity
gcloud logging read 'logName:"logging.googleapis.com%2Fsink_error" AND resource.type="logging_sink"' \
  --freshness=7d --format='table(timestamp, labels.sink_id, labels.error_code, labels.error_detail)'
# highest-fidelity, not throttled:
#   SELECT timestamp, logName, schemaErrorDetail, insertId
#   FROM `project.dataset.export_errors` ORDER BY timestamp DESC
```

**Progress-based liveness (the fix for Failure 1)**
```yaml
livenessProbe:
  exec:
    command: ["/bin/sh","-c","CUR=$(curl -sf --max-time 5 localhost:2020/api/v1/metrics/prometheus | awk '/^fluentbit_output_proc_records_total/ {s+=$2} END {printf \"%.0f\", s+0}'); [ -n \"$CUR\" ] || exit 1; PREV=$(cat /tmp/last 2>/dev/null || echo -1); echo $CUR > /tmp/last; [ \"$PREV\" = -1 ] && exit 0; [ \"$CUR\" -gt \"$PREV\" ]"]
  periodSeconds: 60
  failureThreshold: 5      # must exceed the longest LEGITIMATE quiet period
  timeoutSeconds: 10
```
- Sum across outputs; treat an unreachable endpoint as unhealthy.
- **Prerequisite:** `storage.type filesystem`, else the restart converts a stall into loss.
- On quiet nodes, fail only when **input advances while output doesn't** (unambiguous).
- Do **not** point it at `/api/v1/health`: 404 when `Health_Check Off`, fails open when on.

**Rules of thumb**
- Read blast-radius **shape** before reading logs; it names the layer.
- Two independent agents on one node = a free controlled experiment. They share only the substrate (files, fs, kernel, egress). One working ⇒ fault is inside the other binary.
- **Set a finite `net.io_timeout` on every network output.** Unbounded I/O waits plus a level-triggered loop turn a dead peer into an infinite spin instead of a recoverable error.
- **A hang is worse than a crash.** Bound buffers, set memory limits, add progress probes — make components die instead of stall. (Memory limits alone won't help: hung agents sit far below them, so the OOM killer never fires.)
- On a crash inside metrics or serialisation code, read the log lines **before** the backtrace.
- Liveness = forward progress. Reachability and error-rate probes detect neither hangs nor deadlocks — especially when served by a different thread than the work.
- Capture stacks before restarting; the restart ends the investigation.
- Alert `min by (node)`, never `sum` — one node of 1,200 at zero moves the fleet total by 0.08%.
- Sink diversions are recoverable (`export_errors` **and** the bucket still have them); rotation losses are not. Detection latency on the node half is the number that matters.
- Don't derive a warehouse schema from unvalidated application output.

**Anti-patterns**
- Concluding "logs are fine" from `kubectl logs` (or from `-f`, which skips rotated files).
- Using `/api/v1/health` (or a TCP/HTTP reachability check) as the agent's liveness probe.
- Reading pod-level `restartCount` and stopping there.
- Writing `rate(x) == 0` with no companion `absent_over_time(x)` rule.
- Monitoring an agent exclusively via metrics that the agent's own pod publishes.
- Assuming a managed logging agent is monitored — no user-visible per-node throughput metric exists for it.
- Running metrics/events inputs in the same instance that ships logs.
- **Excluding `mem buf overlimit` warnings from the log pipeline** to cut noise; that's the backpressure signal, and its absence hides the rotate-purge loss.
- Restarting the hung process before capturing `top -H`, `strace`, `ss`, and a backtrace.
- Adding a progress liveness probe while still buffering in memory.
- Letting applications emit a field as both scalar and object, then blaming BigQuery.
- Never querying `export_errors` — the dead-letter table the sink creates for you.
- Alerting on `exports/error_count` with a window under ~7 min (metric visibility lags up to 420s), or relying on it at all for a folder/org-level sink.
- Excluding everything from `_Default` into a sink, removing the only copy you could backfill from.
