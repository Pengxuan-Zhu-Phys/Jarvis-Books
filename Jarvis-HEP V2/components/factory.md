# Component — TaskFactory (`jarvishep2/factory.py`)

**Role**: Worker-process manager + Redis initializer + read-only monitor snapshot provider +
watchdog. It does **not** execute tasks, hold calculators, or own Sample objects.
**Status**: **As-built** @ `jarvis2` (D9.4) — composition of private `_MonitorLoop` /
`_Watchdog`; explicit construction preferred. ~676 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §6;
[`../DESIGN_PRINCIPLES_REVIEW_2.0.md`](../DESIGN_PRINCIPLES_REVIEW_2.0.md) §3.4 / D9.4.
**Reuses V1**: none by import — replaces the singleton thread-pool `WorkerFactory`.

---

## 1. Classes

### 1.1 `TaskFactory` (public facade)

Owns Worker lifecycle and composes two private helpers. Prefer
`TaskFactory(redis_config)` held on `Jarvis2Core.factory`. `get_instance` /
`reset_instance` remain as a **deprecated** process-local shell for older tests.

**Attributes:** `redis_config`, `redis`, `workers`, run metrics
(`_run_started_at`, `_peak_workers_alive`), recovery
(`_worker_spawn_template`, `_redis_connection_config`, `_recovery_lock`,
`_last_recovered_pid`, `_respawn_count`), `_monitor: _MonitorLoop`,
`_watchdog: _Watchdog`, `_logger`.

Compatibility property aliases expose pre-D9.4 private field names
(`_snapshot`, `_updater_thread`, `_running`, `_watchdog_thread`, …) so
existing unit tests keep working without a mass rewrite.

| Method | Behavior |
|--------|----------|
| `get_instance` / `reset_instance` | deprecated singleton shell |
| `init_redis` | control-process `RedisQueue` |
| `start_workers` | register calc pools, spawn `n` Workers (spawn context) |
| `get_run_metrics` | submitted/ok/failed + worker counts; untracked gauges = `None` |
| `stop_all_workers` / `request_worker_shutdown` | SIGTERM then force |
| `get_monitor_snapshot` | in-memory deepcopy via `_monitor` (no Redis) |
| `start_monitor` / `start_watchdog` | start collaborator threads |
| `_handle_worker_failure` | kill → sweep slots → requeue → respawn (on factory so duck-typed stubs work) |
| `shutdown` | stop watchdog + monitor, join Workers, close Redis |

### 1.2 `_MonitorLoop` (private)

Op_count-gated snapshot collector + ~120 Hz background updater (D5.1).

- `collect_latest_status()` — local worker rows + gated Redis sections
- `start(update_hz)` / `stop()` / `get_snapshot()` / `clear()`

### 1.3 `_Watchdog` (private)

Worker liveness loop (D6.1): process-exit or stale-while-busy →
`factory._handle_worker_failure`. Also owns `force_stop_worker`,
`requeue_in_flight_task`, `kill_orphan_process_groups`.

---

## 2. Spawn boundary

`start_workers` passes only `redis.connection_config()` + a deep-copied picklable
`worker_config`. A live Redis client **never** crosses the spawn boundary.

---

## 3. Monitor snapshot (op_count-gated)

Updater always reads the four `op_count` keys + queue lengths; per-subsystem
`HGETALL`s refresh only when the matching `op_count` advanced (or on bootstrap).

---

## 4. Watchdog (D6.1)

`_inspect_workers` every `poll_interval_sec`: dead process or `busy`/`starting`
Worker with heartbeat older than `stale_sec` → recovery (dedup by dead pid).

---

## 5. Interfaces / collaborators

- **Jarvis2Core.init_factory** constructs `TaskFactory(redis_config)`, reuses core Redis,
  `start_workers` + `start_monitor` + `start_watchdog`.
- **client.dispatch_monitor** builds an explicit `TaskFactory` (no singleton).
- **SnapshotReader / run_summary** consume `get_monitor_snapshot` / `get_run_metrics`.

---

## 6. Tests

- `tests/test_monitor_snapshot.py` — explicit `TaskFactory()` (no singleton);
  collaborator composition; op_count gating; honest `None` metrics.
- Worker pool / failure suites exercise recovery via factory facades.
