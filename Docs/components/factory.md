# Component — TaskFactory (`jarvishep2/factory.py`)

**Role**: Worker-process manager + Redis initializer + read-only status snapshot provider +
monitor center. It does **not** execute tasks, hold calculators, or own Sample objects.
**Status**: **D1.1 + D5 + D6.1 implemented** on `jarvis2` — lifecycle (spawn/stop) +
`op_count`-gated monitor snapshot (~120 Hz updater, 60 Hz-safe `get_monitor_snapshot`) +
`get_run_metrics` (D5.2) and watchdog/respawn (D6.1) are live.
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
3. **Read-only monitor** *(implemented)*: a background daemon thread (`_start_snapshot_updater`,
   default ~120 Hz) keeps an in-memory `_snapshot` fresh via **`op_count`-gated** incremental
   fetches — idle subsystems cost one `GET` per kind plus always-cheap `LLEN` queue reads;
   `get_monitor_snapshot()` is an in-memory `deepcopy` (60 Hz-safe, no Redis on the read path).
4. **Watchdog** *(D6.1 — implemented)*: detect stale/dead Worker heartbeats; sweep held calc
   slots; re-queue in-flight Samples (bounded retries); respawn a replacement Worker.

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
        self._last_op_counts: dict[str, int] = {}
        self._updater_thread: threading.Thread | None = None
        self._running = False
        self._run_started_at: float | None = None
        self._peak_workers_alive = 0
        self._logger = get_jarvis_logger("factory")
```

Construction is via the **`TaskFactory.get_instance(redis_config)`** classmethod (process-local
singleton; `reset_instance()` is a test helper).

> **Deviations from the original design** (current code is ground truth):
> - `_watchdog_thread` — **not present**; the watchdog is D6.1.
> - `_logger` (`get_jarvis_logger("factory")`) is added for lifecycle/monitor logging.
> - `worker_heartbeats` — Redis heartbeat hashes keyed by Worker id; refreshed only when
>   `hep:worker:op_count` advances (local `workers` rows are always refreshed from `is_alive()`).

---

## 3. Member functions

**Implemented (D1.1):**

| Method | Signature | Behavior |
|--------|-----------|----------|
| `get_instance` | `(redis_config=None) -> TaskFactory` | Classmethod; return/lazily create the process-local singleton; merges `redis_config` if given. |
| `reset_instance` | `() -> None` | Classmethod; drop the singleton (test helper). |
| `init_redis` | `(*, client=None) -> RedisQueue` | Build the control-process `RedisQueue` from `redis_config` and `connect()` (optional injected `client` for tests). |
| `start_workers` | `(n: int, **worker_kwargs) -> list[Worker]` | Spawn `n` `Worker(worker_id, redis.connection_config(), config)` via the spawn ctx; refuses if Redis is uninitialized or live Workers already exist; deep-copies `worker_kwargs` per Worker; tracks them in `self.workers`. See §3a for what crosses the spawn boundary. |
| `stop_all_workers` | `(*, graceful=True, join_timeout=30.0) -> None` | `graceful`: `SIGTERM` (via `request_worker_shutdown`) + join; then force-`terminate()` + short join for any survivors; clears the list. |
| `request_worker_shutdown` | `() -> None` | Send `SIGTERM` to each live Worker pid (finish current Sample, then exit). |
| `get_monitor_snapshot` | `() -> dict` | In-memory `deepcopy(self._snapshot)` under the lock; no Redis call. |
| `get_run_metrics` | `() -> dict` | Project `submitted`/`ok`/`failed` + worker counts for `run_summary` (D5.2). |
| `start_monitor` | `(*, update_hz=120.0) -> None` | Launch `_start_snapshot_updater` (idempotent if already alive). |
| `_start_snapshot_updater` | `(*, update_hz=120.0) -> None` | Background daemon loop at ~100–120 Hz; replaces `_snapshot` under the lock. |
| `_collect_latest_status` | `() -> dict` | Build one snapshot: local Worker rows + `op_count`-gated Redis sections (see §4). |
| `_worker_status` | `() -> list[dict]` | Per-Worker `{worker_id, pid, alive, name}` rows. |
| `_fetch_workers_redis` | `() -> dict` | Redis heartbeat hashes for tracked Workers (gated on `worker` op_count). |
| `shutdown` | `(*, wait=True) -> None` | Stop the monitor thread, `request_worker_shutdown`, `stop_all_workers`, **close the Redis client**, log a completion line. |

**Designed but not yet implemented** (do not call — tracked WPs):

| Method | WP | Status |
|--------|----|--------|
| `start_watchdog` / `_handle_worker_failure` | D6.1 | implemented — heartbeat-staleness + respawn + in-flight re-queue. |


---

## 3a. What the Factory passes to a Worker (spawn boundary)

`start_workers` constructs `Worker(worker_id, self.redis.connection_config(), config)` — i.e. it
passes **only picklable connection settings** derived from the control-process queue. The Worker's
`__init__` also accepts a live `RedisQueue` for convenience (tests), normalizing via
`RedisQueue.extract_connection_config(redis)`. A live Redis client **never crosses the spawn
boundary**: under the spawn context only `redis_config` + the picklable `worker_config` travel to
the child, which opens its **own** Redis client in `Worker.run()`.

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

## 4. Monitor snapshot (`op_count`-gated incremental fetch)

```python
def _collect_latest_status(self) -> dict:
    snap = {
        "timestamp": time.time(),
        "workers": self._worker_status(),          # always local — no Redis
        "workers_alive": ...,
        "workers_total": len(self.workers),
    }
    op_counts = self.redis.get_all_op_counts()     # 4× GET (one pipeline)
    lengths = self.redis.get_queue_lengths()       # 2× LLEN (always refreshed)
    snap["task_queue_length"] = lengths["task_queue_length"]
    snap["archive_queue_length"] = lengths["archive_queue_length"]
    snap["op_counts"] = op_counts

    # Per subsystem: fetch only when op_count advanced (or section missing on bootstrap)
    if worker_op_count advanced:
        snap["worker_heartbeats"] = self.redis.fetch_worker_status(...)
    else:
        snap["worker_heartbeats"] = carry_forward(prev)
    # same pattern for calculator_status (HGETALL) and sample_stats (HGETALL)

    self._last_op_counts = dict(op_counts)
    return snap
```

- **Behavior.** `_start_snapshot_updater(update_hz=120.0)` runs a daemon loop that calls
  `_collect_latest_status()` every `1/update_hz` seconds and replaces `self._snapshot` under the
  lock. Each tick always reads all four `op_count` keys and both queue lengths; subsystem
  `HGETALL`s run **only** when the corresponding `op_count` increased since the last tick (or on
  first bootstrap when the section is absent). Idle ticks therefore avoid heavy hash reads.
  `get_monitor_snapshot()` is a pure in-memory `deepcopy` — safe at 60 Hz from `--monitor`.
  Updater exceptions are logged and the loop continues (Redis loss never crashes the control
  process).
- **`op_count` contract.** A monotonic global version number per subsystem; the Factory only
  **compares**, never writes.

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
- **Worker death** *(D6.1 — implemented)*: the watchdog detects process exit or stale
  `last_heartbeat` while `busy`/`starting`, sweeps `held_calc_packs`, re-queues the in-flight
  task (up to `Runtime.Watchdog.max_sample_retries`), and respawns a replacement Worker.
- **Redis loss**: the monitor updater logs the exception and continues the loop; it never crashes
  the control process. Workers/Sampler surface the hard error on their own path.
- **Independent monitor process** *(design goal)*: the snapshot is reconstructable purely from
  Redis, so `Jarvis2 --monitor --pid N` can attach from another process/host reading the same
  keys (Blueprint §6) — the in-memory snapshot is an optimization, not the only source. (The
  `--monitor` reader is D5.2 — see [monitor.md](monitor.md).)

---

## 6. Interfaces

- **`Jarvis2Core.init_factory`** (see [core.md](core.md)) creates/obtains the singleton, reuses
  the core's Redis client (or `init_redis()`), then `start_workers(Runtime.workers)` +
  `start_monitor(update_hz=120.0)` + `start_watchdog()` from `Runtime.Watchdog`.
- **Sampler** writes tasks to Redis directly (Factory does not relay).
- **Worker** receives the picklable config + connection settings from `start_workers`; everything
  else flows through Redis (§3a, §5).
- **Monitor / dashboard** reads `get_monitor_snapshot()` (or Redis directly).
- **run_summary** *(D5.2)* reads `get_run_metrics()` via `Jarvis2Core.write_run_summary()` at
  shutdown (optional `shutdown(write_run_summary=True)`).

---

## 7. Tests

**Current (D1.1 + D5.1):** lifecycle + `op_count`-gated snapshot coverage under `fakeredis` + real
spawn — `start_workers(n)` spawns live processes; `start_monitor` populates `_snapshot`;
`get_monitor_snapshot()` is a pure in-memory read; `get_run_metrics()` projects sample stats;
`shutdown` joins all Workers (no orphan), stops the monitor thread, and closes Redis.

**Covered in D1.1 tests (`tests/test_worker_mvp.py`) + D5.1 (`tests/test_monitor_snapshot.py`):**
1. **op_count gating** — idle ticks skip `HGETALL`; subsystem refresh follows `op_count` bump.
2. **Snapshot latency** — `get_monitor_snapshot()` is pure memory; 60 Hz loop over it does not
   touch the server.
3. **Read-only invariant** — `_collect_latest_status` issues zero Redis writes on idle ticks.
4. **Graceful shutdown** — monitor thread stops; snapshot cleared; Redis closed.

**Planned with their WPs:**
5. **Watchdog respawn** (D6.1) — SIGKILL a Worker → watchdog re-queues its Sample, sweeps its
   slots, spawns a replacement; final UUID set has no missing/duplicate.

Verification logic: tests 1/2/4 prove the Factory stays off the hot path (the failure mode of
the retired throughput-core monitor); test 3 proves crash safety.

---

## 8. Open questions

- Auto-scale Worker count from `task_queue_length` (future; off by default).
- Snapshot via Redis pub/sub push vs. polling (polling + op_count is the planned baseline).
