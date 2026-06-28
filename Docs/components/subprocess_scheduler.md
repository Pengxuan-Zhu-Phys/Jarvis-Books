# Component — Subprocess Scheduler (`jarvishep2/async_subprocess.py`)

**Role**: the asyncio-based external-process runner. In V2 each **Worker owns one instance** and
uses it to run a Sample's same-layer calculators concurrently (the in-Sample parallelism that is
the slow-regime speedup).
**Status**: design — plan WP-D2.2 (per-Worker), reused from V1.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4; discussion
`Jarvis-HEP_Discussion_Summary_2026-06-21.md` §7; [worker.md](worker.md), [calculator.md](calculator.md).
**Reuses V1**: `AsyncSubprocessScheduler` + `SubprocessJob`/`SubprocessResult`/`SubprocessRuntimeConfig`
(`jarvishep/async_subprocess.py`) — **reused as-is**, only the ownership changes (per-Worker, not
one global).

---

## 1. Responsibilities

1. Run external commands as async subprocesses with **timeout**, env injection
   (`SubprocessJob(env=…)`), stdout/stderr draining, and graceful terminate→kill fallback.
2. Bound concurrency by a configured max (within one Worker).
3. Expose a cheap **snapshot** (running/peak/pending, fd/rss) for monitoring.
4. **V2**: one scheduler **per Worker** (its own event loop), so each Worker's same-layer
   calculators run concurrently while the Worker still handles one Sample at a time.

---

## 2. Structure (unchanged from V1)

```python
@dataclass
class SubprocessJob:      command; cwd; env; timeout; stage; ...
@dataclass
class SubprocessResult:   returncode; stdout; stderr; duration; def ok() -> bool
class AsyncSubprocessScheduler:
    def __init__(self, max_concurrency, ...): ...
    def start(self): ...
    def submit(self, job) -> concurrent.futures.Future: ...
    async def arun(self, job) -> SubprocessResult: ...
    def run(self, job, timeout=None) -> SubprocessResult: ...
    def snapshot(self) -> dict: ...
    def shutdown(self, wait=True, timeout=30.0): ...
```

---

## 3. Member functions (key)

| Method | Signature | Behavior |
|--------|-----------|----------|
| `submit` | `(job) -> Future` | Schedule a subprocess on the Worker's loop; returns a future (used to fan out a layer). |
| `arun` | `async (job) -> SubprocessResult` | Await one subprocess (the layer-concurrency primitive). |
| `run` | `(job, timeout) -> SubprocessResult` | Sync convenience (single calculator). |
| `snapshot` | `() -> dict` | `running`/`peak_running`/`pending`/`fd_count`/`rss_mb` — fed to the monitor and run_summary. |
| `shutdown` | `(wait, timeout) -> None` | Drain + stop the loop thread; terminate stragglers. |

---

## 4. Per-Worker usage (the V2 change)

```
V1: one global scheduler shared by the thread pool.
V2: Worker._init: self.scheduler = AsyncSubprocessScheduler(max_concurrency=layer_width)
    Worker._run_layer(steps):
        futs = [self.scheduler.submit(job_for(step)) for step in steps]   # same layer ⇒ concurrent
        wait(futs)                                                        # await the layer
```

Each Worker has its own loop; there is **no** global scheduler contended across Workers. Global
calculator concurrency is capped separately by the Redis free-pool ([redis_queue.md](redis_queue.md)),
so the per-Worker scheduler only governs *within-Sample* fan-out.

---

## 5. Concurrency / lifecycle / failure semantics

- One loop thread per Worker process; started in `run()` (spawn-safe — built in the child).
- **Timeout** → terminate-then-kill fallback; the step's `SubprocessResult.ok()` is False; the
  Worker marks the Sample Failed (with a log) and releases the slot.
- `snapshot()` is cheap and lock-light — safe to poll for the monitor's `peak_running` (feeds the
  `parallelism_efficiency` metric).
- Shutdown drains in-flight subprocesses then stops the loop; no orphan processes.
- `io_manager.py`'s blocking-work executor role (V1) is **absorbed** here — calculator-path I/O
  runs through the scheduler/loop; opera-only scans never start a scheduler.

---

## 6. Interfaces

- **Worker** → owns one scheduler; `submit`/`arun` to fan out a layer.
- **CalculatorModule** → `run_command` issues `SubprocessJob`s to it (with env from
  [env_setup.md](env_setup.md)).
- **Monitor / run_summary** → `snapshot()` for `peak_running` / efficiency.

---

## 7. Tests (`tests/test_subprocess_scheduler.py`)

Unit:
1. **Concurrency** — N independent jobs submitted together finish in ≈max(durations), not the sum
   (timed), bounded by `max_concurrency`.
2. **Timeout** — a hung job is terminated→killed; `ok()` False; no orphan.
3. **Env injection** — `SubprocessJob(env=…)` is visible to the child (pairs with env_setup).
4. **Snapshot** — `running`/`peak_running`/`pending` track an in-flight batch correctly.
5. **Per-Worker isolation** — two schedulers (two Workers) don't share state; shutdown of one
   doesn't affect the other.
6. **Spawn-safe** — built inside `run()`; no live loop pickled.

Verification logic: test 1 proves the in-Sample fan-out (the slow-regime lever); test 4 backs the
`parallelism_efficiency` metric.

---

## 8. Open questions

- Whether async operas (`call_mode: acall`) share this loop or get their own (default: share;
  revisit on contention).
