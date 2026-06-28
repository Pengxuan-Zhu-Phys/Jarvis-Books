# Component — TaskFactory (`jarvishep2/factory.py`)

**Role**: Worker-process manager + Redis initializer + read-only status snapshot provider +
monitor center. It does **not** execute tasks, hold calculators, or own Sample objects.
**Status**: design — plan WP-D1.1 (spawn/manage) → D5.1 (op_count snapshot) → D6.1 (watchdog).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §6;
discussion `factory_design.md` (primary), Blueprint §6 (independent monitor).
**Reuses V1**: replaces the singleton thread-pool `WorkerFactory` (`jarvishep/factory.py`),
keeping its run-metrics shape where the contract requires.

---

## 1. Responsibilities

1. **Worker lifecycle**: start/stop N `Worker` processes, pass picklable init config, respawn
   dead ones (D6.1).
2. **Redis init**: create the client + ensure keys exist before Workers/Sampler start.
3. **Read-only monitor**: a background thread keeps an in-memory `_snapshot` fresh via the
   **`op_count`** change-detection mechanism; `get_monitor_snapshot()` is a sub-millisecond
   in-memory `deepcopy`, safe at **60 Hz**.
4. **Watchdog** (D6.1): detect stale Worker heartbeats; sweep held slots; re-queue in-flight
   Samples; respawn.

Read/write separation: during a normal run, **Workers and the Sampler are the only writers**;
the Factory only reads. This keeps it off the hot path entirely.

---

## 2. Class structure

```python
class TaskFactory:
    _instance = None                 # process-local singleton
    _lock = threading.Lock()

    def __init__(self, redis_config: dict | None = None):
        self.redis_config = redis_config or {}
        self.redis: RedisQueue | None = None
        self.workers: list[Worker] = []
        self._snapshot: dict = {}
        self._snapshot_lock = threading.RLock()
        self.last_op_counts = {"worker": 0, "calculator": 0, "sample": 0, "task": 0}
        self._updater_thread: threading.Thread | None = None
        self._watchdog_thread: threading.Thread | None = None
        self._running = False
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `init_redis` | `() -> None` | Build `RedisQueue`; ensure keys; reset nothing (op_counts persist within a run). |
| `start_workers` | `(n: int, **worker_kwargs) -> None` | Spawn `n` `Worker` processes (spawn ctx) with picklable config; track in `self.workers`. |
| `stop_all_workers` | `(graceful=True) -> None` | `SIGTERM` then join; force-terminate after a grace period. |
| `start_monitor` | `(update_hz=120) -> None` | Launch the background snapshot updater thread. |
| `get_monitor_snapshot` | `() -> dict` | In-memory `deepcopy(self._snapshot)` under the lock; target < 0.5 ms; 60 Hz-safe. |
| `_collect_latest_status` | `() -> dict` | The `op_count`-gated collector (see §4). |
| `start_watchdog` | `(interval=1.0, timeout=10.0) -> None` | Heartbeat-staleness detection + respawn (D6.1). |
| `_respawn_worker` | `(worker_id) -> None` | Sweep held slots (`BUSY→free`), re-queue its in-flight Sample (bounded retries), spawn a replacement. |
| `get_run_metrics` | `() -> dict` | Project `hep:sample:stats` + timing into the frozen run-summary shape (D5.2). |
| `shutdown` | `(wait=True) -> None` | Stop monitor + watchdog, stop workers, close Redis, log a shutdown summary. |

---

## 4. op_count-driven snapshot (the key optimization)

```python
def _collect_latest_status(self) -> dict:
    snap = {"timestamp": time.time()}
    for kind, fetch in (("worker", self._fetch_workers),
                        ("calculator", self._fetch_calculators),
                        ("sample", self._fetch_samples)):
        cur = self.redis.get_op_count(kind)
        if cur > self.last_op_counts[kind]:          # only fetch on change
            snap[kind] = fetch()
            self.last_op_counts[kind] = cur
        else:
            snap[kind] = self._snapshot.get(kind)    # carry forward
    snap["task_queue_length"] = self.redis.r.llen(TASK_QUEUE)   # cheap, always
    return snap
```

- The updater thread runs at ~100–200 Hz, giving the 60 Hz monitor headroom.
- **Idle subsystems cost one `GET`/tick** — no `HGETALL` storms when nothing changes.
- `op_count` is a monotonic global version number; the Factory only **compares**, never writes.

---

## 5. Concurrency / lifecycle / failure semantics

- The Factory is a **process-local singleton** per the control process; the monitor and
  watchdog are daemon threads; `get_monitor_snapshot` is lock-guarded and never blocks on
  Redis (pure memory read).
- **Worker death**: watchdog notices a heartbeat older than `timeout`, calls `_respawn_worker`;
  in-flight Sample re-queue is bounded (avoid poison-task loops).
- **Redis loss**: the updater logs and backs off; it never crashes the control process; the
  Sampler/Workers surface the hard error.
- **Independent monitor process**: because the snapshot is reconstructable purely from Redis,
  `Jarvis2 --monitor --pid N` can attach from another process/host reading the same keys
  (Blueprint §6) — the Factory's in-memory snapshot is an optimization, not the only source.

---

## 6. Interfaces

- **core.init_factory** creates it, `init_redis`, `start_workers`, `start_monitor`,
  `start_watchdog`.
- **Sampler** writes tasks to Redis directly (Factory does not relay).
- **Monitor / dashboard** reads `get_monitor_snapshot()` (or Redis directly).
- **run_summary** reads `get_run_metrics()` at shutdown.

---

## 7. Tests (`tests/test_monitor_snapshot.py`, `test_factory_lifecycle.py`, `fakeredis`)

Unit:
1. **op_count gating** — mutate one subsystem → only that subsystem is re-fetched next tick
   (assert call counts on a spy client); idle ticks do one `GET` per kind + one `LLEN`.
2. **Snapshot latency** — `get_monitor_snapshot()` is pure memory (no Redis call); 60 Hz loop
   over it does not touch the server.
3. **Advancing counters** — an external writer bumping stats is reflected within one tick.
4. **Lifecycle** — `start_workers(2)` spawns 2 live processes; `shutdown` joins all, no orphan.
5. **Watchdog respawn** (D6.1) — SIGKILL a Worker → watchdog re-queues its Sample, sweeps its
   slots, spawns a replacement; final UUID set has no missing/duplicate.
6. **Read-only invariant** — during a normal run the Factory issues zero Redis *writes*
   (assert on a spy client).

Verification logic: tests 1/2/6 prove the Factory stays off the hot path (the failure mode of
the retired throughput-core monitor); test 5 proves crash safety.

---

## 8. Open questions

- Auto-scale Worker count from `task_queue_length` (future; off by default).
- Snapshot via Redis pub/sub push vs. polling (polling + op_count is the baseline).
