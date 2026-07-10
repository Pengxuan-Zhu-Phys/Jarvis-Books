# Component — TaskFactory (`jarvishep2/factory.py`)

**Role**: Worker-process manager + Redis initializer + read-only monitor snapshot provider +
watchdog. It does **not** execute tasks, hold calculators, or own Sample objects.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `factory.py` 471 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §6; `factory_design.md`.
**Reuses V1**: none by import — replaces the singleton thread-pool `WorkerFactory`.

---

## 1. Class defined — `TaskFactory`

Process-local singleton (class vars `_instance`, `_lock`) that spawns and monitors Workers.

**Attributes** (from `__init__`): `redis_config`, `redis`, `workers: list[Worker]`, snapshot
state (`_snapshot`, `_snapshot_lock` RLock, `_last_op_counts`, `_updater_thread`, `_running`),
run metrics (`_run_started_at`, `_peak_workers_alive`), watchdog state (`_watchdog_thread`,
`_watchdog_running`, `_watchdog_stale_sec`, `_watchdog_poll_interval_sec`, `_max_sample_retries`),
recovery state (`_worker_spawn_template`, `_redis_connection_config`, `_recovery_lock`,
`_last_recovered_pid`, `_respawn_count`), `_logger`.

**Member functions:**

| Method | Behavior |
|--------|----------|
| `get_instance(redis_config=None)` (`@classmethod`) | return/create the singleton; merge redis_config. |
| `reset_instance()` (`@classmethod`) | drop the singleton (test helper). |
| `init_redis(*, client=None)` | build + connect the control-process `RedisQueue`. |
| `_alive_workers()` | live Workers. |
| `start_workers(n, **worker_kwargs)` | register calc pools, deep-copy config per Worker, spawn `n` `Worker`s with picklable connection settings; refuse if Redis uninitialized or Workers already alive; record spawn template + connection config for respawn. |
| `get_run_metrics()` | project submitted/ok/failed + worker counts for run_summary. |
| `stop_all_workers(*, graceful=True, join_timeout=30)` | SIGTERM + join, then force-terminate survivors. |
| `request_worker_shutdown()` | SIGTERM every live Worker pid. |
| `get_monitor_snapshot()` | in-memory deepcopy of the latest snapshot (no Redis on the read path). |
| `_worker_status()` | per-Worker `{worker_id,pid,alive,name}` rows. |
| `_fetch_workers_redis()` | heartbeat hashes for tracked Workers. |
| `_carry_forward_section(...)` / `_subsystem_refresh_needed(...)` | op_count-gated incremental fetch helpers. |
| `_collect_latest_status()` | build one snapshot: local rows + op_count-gated Redis sections + queue lengths. |
| `_start_snapshot_updater(*, update_hz=120)` / `start_monitor(*, update_hz=120)` | background ~120 Hz updater thread. |
| `start_watchdog(*, enabled=True, stale_sec=30, poll_interval_sec=1, max_sample_retries=3)` | launch the D6.1 watchdog thread. |
| `_heartbeat_timestamp(heartbeat)` (`@staticmethod`) | parse last_heartbeat/ts. |
| `_worker_heartbeat(worker_id)` | fetch one heartbeat hash. |
| `_inspect_workers()` | per Worker: process-exit or stale-while-busy → `_handle_worker_failure`. |
| `_force_stop_worker(worker)` | SIGKILL + join. |
| `_requeue_in_flight_task(heartbeat)` | re-queue the decoded in-flight task with a retry counter; submit a Failed result when retries exhausted. |
| `_handle_worker_failure(worker, *, reason)` | sweep held calc slots, re-queue task, kill + respawn a replacement Worker (dedup by dead pid). |
| `shutdown(*, wait=True)` | stop watchdog + monitor threads, signal + join Workers, close Redis, reset all state. |

---

## 2. Spawn boundary

`start_workers` passes only `redis.connection_config()` (settings) + a deep-copied picklable
`worker_config`. A live Redis client **never** crosses the spawn boundary — each Worker opens its
own client in `Worker.run()`. The respawn path reuses `_redis_connection_config` +
`_worker_spawn_template`.

---

## 3. Monitor snapshot (op_count-gated)

The updater always reads the four `op_count` keys + both queue lengths each tick; the per-subsystem
`HGETALL`s (worker heartbeats / calculator status / sample stats) refresh **only** when their
`op_count` advanced (or on bootstrap). `get_monitor_snapshot()` is a pure in-memory deepcopy
(safe to poll). Updater exceptions are logged; Redis loss never crashes the control process.

---

## 4. Watchdog (D6.1)

`_inspect_workers` runs every `poll_interval_sec`: a dead process or a `busy`/`starting` Worker
whose heartbeat is older than `stale_sec` triggers `_handle_worker_failure` →
`sweep_held_calc_slots` + `_requeue_in_flight_task` (bounded by `max_sample_retries`) + respawn.

---

## 5. Interfaces / collaborators

- **Jarvis2Core.init_factory** ([core.md](core.md)) obtains the singleton, reuses the core Redis
  client, `start_workers` + `start_monitor` + `start_watchdog`.
- **Worker** ([worker.md](worker.md)) receives the picklable config; all task/result flow is via
  Redis (Factory does not relay).
- **RedisQueue** ([redis_queue.md](redis_queue.md)) read-only monitor reads + slot sweep.
- **run_summary** ([monitor.md](monitor.md)) consumes `get_run_metrics()`.

---

## 6. Drift from design

As-built the watchdog/respawn (designed as "deferred") **is implemented**; the snapshot updater
default is 120 Hz; `get_run_metrics` and the recovery bookkeeping (`_worker_spawn_template`,
`_last_recovered_pid`, `_respawn_count`) are present.

---

## 7. Tests

`tests/test_worker_pool.py` (6), `test_monitor_snapshot.py` (5), `test_worker_failure.py` (2),
`test_distributed_resume.py` (8): lifecycle, op_count-gated snapshot, watchdog respawn + slot
sweep + in-flight re-queue, graceful shutdown.
