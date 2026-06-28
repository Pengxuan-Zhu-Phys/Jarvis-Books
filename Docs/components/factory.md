# Component — TaskFactory (`jarvishep2/factory.py`)

**Role**: Worker-process manager + Redis initializer + read-only status snapshot provider +
monitor center. It does **not** execute tasks, hold calculators, or own Sample objects.
**Status**: **D1.1 partially implemented** on `jarvis2` — lifecycle (spawn/stop) + a
fixed-frequency polling monitor are live; `op_count`-gated snapshot (D5.1), watchdog/respawn
(D6.1), and `get_run_metrics` (D5.2) are not yet implemented. Sections below mark each gap.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §6;
discussion `factory_design.md` (primary), Blueprint §6 (independent monitor).
**Reuses V1**: replaces the singleton thread-pool `WorkerFactory` (`jarvishep/factory.py`),
keeping its run-metrics shape where the contract requires.

> **Doc-vs-code note.** This page follows the **current `jarvishep2/factory.py` implementation**.
> Where the original design (`op_count` monitor, watchdog) is not yet built, the text says so
> and points at the owning WP rather than describing it as live.

---

## 1. Responsibilities

1. **Worker lifecycle** *(implemented)*: start/stop N `Worker` processes, pass picklable init
   config, signal graceful shutdown. *Respawn of dead Workers is deferred to D6.1.*
2. **Redis init** *(implemented)*: create the control-process client (`init_redis`) before
   Workers start.
3. **Read-only monitor** *(D1.1 baseline implemented)*: a background daemon thread keeps an
   in-memory `_snapshot` fresh by **fixed-frequency polling** (default ~2 Hz) — each tick reads
   the full `RedisQueue.snapshot_raw()`. `get_monitor_snapshot()` is an in-memory `deepcopy`.
   The **`op_count`-gated** change-detection optimization and the 60 Hz target are **D5.1**, not
   yet built (the raw snapshot already exposes `op_counts`, but the updater does not gate on
   them).
4. **Watchdog** *(D6.1 — not implemented)*: detect stale Worker heartbeats; sweep held slots;
   re-queue in-flight Samples; respawn.

Read/write separation: during a normal run, **Workers and the Sampler are the only writers**;
the Factory only reads. This keeps it off the hot path entirely.

---

## 2. Class structure

```python
class TaskFactory:
    _instance = None                 # process-local singleton
    _lock = threading.Lock()

    def __init__(self, redis_config: dict | None = None):
        self.redis_config = dict(redis_config or {})
        self.redis: RedisQueue | None = None
        self.workers: list[Worker] = []
        self._snapshot: dict = {}
        self._snapshot_lock = threading.RLock()
        self._updater_thread: threading.Thread | None = None
        self._running = False
        self._logger = get_jarvis_logger("factory")
```

Construction is via the **`TaskFactory.get_instance(redis_config)`** classmethod (process-local
singleton; `reset_instance()` is a test helper).

> **Deviations from the original design** (current code is ground truth):
> - `last_op_counts` — the `op_count`-gating bookkeeping field — is **not present**; it lands
>   with the D5.1 gated collector.
> - `_watchdog_thread` — **not present**; the watchdog is D6.1.
> - `_logger` (`get_jarvis_logger("factory")`) is added for lifecycle/monitor logging.

---

## 3. Member functions

**Implemented (D1.1):**

| Method | Signature | Behavior |
|--------|-----------|----------|
| `get_instance` | `(redis_config=None) -> TaskFactory` | Classmethod; return/lazily create the process-local singleton; merges `redis_config` if given. |
| `reset_instance` | `() -> None` | Classmethod; drop the singleton (test helper). |
| `init_redis` | `(*, client=None) -> RedisQueue` | Build the control-process `RedisQueue` from `redis_config` and `connect()` (optional injected `client` for tests). |
| `start_workers` | `(n: int, **worker_kwargs) -> list[Worker]` | Spawn `n` `Worker(worker_id, self.redis, config)` via the spawn ctx; refuses if Redis is uninitialized or live Workers already exist; deep-copies `worker_kwargs` per Worker; tracks them in `self.workers`. See §3a for what crosses the spawn boundary. |
| `stop_all_workers` | `(*, graceful=True, join_timeout=30.0) -> None` | `graceful`: `SIGTERM` (via `request_worker_shutdown`) + join; then force-`terminate()` + short join for any survivors; clears the list. |
| `request_worker_shutdown` | `() -> None` | Send `SIGTERM` to each live Worker pid (finish current Sample, then exit). |
| `get_monitor_snapshot` | `() -> dict` | In-memory `deepcopy(self._snapshot)` under the lock; no Redis call. |
| `start_monitor` | `(*, update_hz=2.0) -> None` | Launch the background daemon updater thread (idempotent if already alive). Fixed-frequency polling (see §4). |
| `_collect_latest_status` | `() -> dict` | Build one snapshot: local Worker rows + `RedisQueue.snapshot_raw()` (queue lengths, `sample_stats`, `calculator_status`, `op_counts`). **Not** op_count-gated yet. |
| `_worker_status` | `() -> list[dict]` | Per-Worker `{worker_id, pid, alive, name}` rows. |
| `shutdown` | `(*, wait=True) -> None` | Stop the monitor thread, `request_worker_shutdown`, `stop_all_workers`, **close the Redis client**, log a completion line. |

**Designed but not yet implemented** (do not call — tracked WPs):

| Method | WP | Status |
|--------|----|--------|
| `start_watchdog` / `_respawn_worker` | D6.1 | not present — heartbeat-staleness + respawn + in-flight re-queue. |
| `get_run_metrics` | D5.2 | not present — run_summary projection. |
| `op_count`-gated `_collect_latest_status` | D5.1 | current collector polls the full snapshot every tick. |

---

## 3a. What the Factory passes to a Worker (spawn boundary)

`start_workers` constructs `Worker(worker_id, self.redis, config)` — i.e. it hands the Worker
the **live control-process `RedisQueue`**. But the Worker's `__init__` immediately calls
`RedisQueue.extract_connection_config(redis)`, storing **only the picklable `redis_config`**
(connection settings). A live `RedisQueue` therefore **never crosses the spawn boundary**: under
the spawn context only `redis_config` + the picklable `worker_config` travel to the child, which
opens its **own** Redis client in `Worker.run()`.

**Design choice / rationale.** A connected Redis client (sockets, connection pool) is not
picklable and must not be shared across a process boundary — mirroring invariant #8 for loggers.
Accepting the live queue at the call site is a convenience (the control process and the Workers
share one source of connection settings); normalizing it down to `redis_config` is what makes the
Worker spawn-safe. Each Worker owning its client also keeps the Factory off the Workers' hot path.

## 3b. Recommended initialization order

`init_redis()` → `start_workers(n, ...)` → `start_monitor()`.

- `init_redis()` **must** run first: `start_workers` raises `RuntimeError` if `self.redis is None`,
  and the monitor reads `self.redis.snapshot_raw()`.
- `start_workers` refuses a second call while live Workers exist (no accidental double-spawn).
- `start_monitor` can technically run before Workers, but ordering it last means the first
  snapshot already contains Worker rows.

This is exactly the order `Jarvis2Core.init_factory` uses (see [core.md](core.md) §2).

---

## 4. Monitor snapshot (current: fixed-frequency polling)

```python
def _collect_latest_status(self) -> dict:
    snap = {
        "timestamp": time.time(),
        "workers": self._worker_status(),
        "workers_alive": sum(w.is_alive() for w in self.workers),
        "workers_total": len(self.workers),
    }
    if self.redis is None:
        return snap
    raw = self.redis.snapshot_raw()                 # full read every tick (no op_count gate yet)
    snap["task_queue_length"]    = raw.get("task_queue_length", 0)
    snap["archive_queue_length"] = raw.get("archive_queue_length", 0)
    snap["sample_stats"]         = raw.get("sample_stats", {})
    snap["calculator_status"]    = raw.get("calculator_status", {})
    snap["op_counts"]            = raw.get("op_counts", {})
    return snap
```

- **Current behavior (D1.1 baseline).** `start_monitor(update_hz=2.0)` runs a daemon loop that
  calls `_collect_latest_status()` every `1/update_hz` seconds (default ~2 Hz) and replaces
  `self._snapshot` under the lock. Each tick does a **full** `snapshot_raw()` (queue `LLEN`s +
  two `HGETALL`s + per-kind `op_count` GETs); `op_counts` is reported but **not** used to skip
  fetches. Updater exceptions are logged and the loop continues (Redis loss never crashes the
  control process).
- **Planned (D5.1).** Gate per-subsystem fetches on a monotonic `op_count` so idle subsystems
  cost one `GET`/tick, raise the updater frequency, and make `get_monitor_snapshot()` 60 Hz-safe.
  `op_count` is a global version number the Factory only **compares**, never writes.

---

## 5. Concurrency / lifecycle / failure semantics

- The Factory is a **process-local singleton** (via `get_instance`); the monitor is a daemon
  thread; `get_monitor_snapshot` is lock-guarded and never blocks on Redis (pure memory read).
- **Factory ↔ Worker collaboration & lifecycle.** The Factory **owns the Worker processes'
  lifecycle but not their work**: it spawns them (`start_workers`), hands each a picklable
  config + connection settings, and later signals them to stop. Once spawned, a Worker pulls and
  executes Samples on its own (see [worker.md](worker.md)); the Factory never relays tasks or
  results — those flow Sampler → Redis → Worker → Redis → control process directly. The Factory's
  only ongoing interaction with live Workers is **observational** (`is_alive()` / heartbeats read
  into the snapshot) plus a one-shot `SIGTERM` at shutdown. Typical lifecycle:
  `get_instance → init_redis → start_workers → start_monitor → … run … → shutdown`.
- **Graceful shutdown** *(implemented)*: `shutdown()` stops the monitor thread, `SIGTERM`s every
  live Worker (`request_worker_shutdown`), joins them (force-`terminate()` after the grace
  period), then **closes the Redis client** and drops the handle. A Worker that receives
  `SIGTERM` finishes its current Sample before exiting (signal handler flips `_is_running`).
- **Worker death** *(D6.1 — not implemented)*: there is currently **no watchdog/respawn**. A
  Worker that dies mid-Sample is simply gone; recovery (stale-heartbeat detection, slot sweep,
  in-flight re-queue, respawn) lands in D6.1.
- **Redis loss**: the monitor updater logs the exception and continues the loop; it never crashes
  the control process. Workers/Sampler surface the hard error on their own path.
- **Independent monitor process** *(design goal)*: the snapshot is reconstructable purely from
  Redis, so `Jarvis2 --monitor --pid N` can attach from another process/host reading the same
  keys (Blueprint §6) — the in-memory snapshot is an optimization, not the only source. (The
  `--monitor` reader itself is D5.2.)

---

## 6. Interfaces

- **`Jarvis2Core.init_factory`** (see [core.md](core.md)) creates/obtains the singleton, reuses
  the core's Redis client (or `init_redis()`), then `start_workers(Runtime.workers)` +
  `start_monitor(update_hz=2.0)`. *(No `start_watchdog` yet — D6.1.)*
- **Sampler** writes tasks to Redis directly (Factory does not relay).
- **Worker** receives the picklable config + connection settings from `start_workers`; everything
  else flows through Redis (§3a, §5).
- **Monitor / dashboard** reads `get_monitor_snapshot()` (or Redis directly).
- **run_summary** *(D5.2)* will read `get_run_metrics()` at shutdown — not implemented yet.

---

## 7. Tests

**Current (D1.1):** lifecycle + polling-snapshot coverage under `fakeredis` + real spawn —
`start_workers(n)` spawns live processes; `start_monitor` populates `_snapshot`;
`get_monitor_snapshot()` is a pure in-memory read; `shutdown` joins all Workers (no orphan) and
closes Redis.

**Planned with their WPs:**
1. **op_count gating** (D5.1) — mutate one subsystem → only that subsystem is re-fetched next
   tick; idle ticks do one `GET` per kind + one `LLEN`.
2. **Snapshot latency** (D5.1) — `get_monitor_snapshot()` is pure memory; 60 Hz loop over it
   does not touch the server.
3. **Watchdog respawn** (D6.1) — SIGKILL a Worker → watchdog re-queues its Sample, sweeps its
   slots, spawns a replacement; final UUID set has no missing/duplicate.
4. **Read-only invariant** (D5.1) — during a normal run the Factory issues zero Redis *writes*.

Verification logic: tests 1/2/4 prove the Factory stays off the hot path (the failure mode of
the retired throughput-core monitor); test 3 proves crash safety.

---

## 8. Open questions

- Auto-scale Worker count from `task_queue_length` (future; off by default).
- Snapshot via Redis pub/sub push vs. polling (polling + op_count is the planned baseline).
