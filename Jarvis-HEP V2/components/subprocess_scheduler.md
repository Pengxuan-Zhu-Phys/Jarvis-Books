# Component — Subprocess Scheduler (`jarvishep2/async_subprocess.py`)

**Role**: the asyncio-based external-process runner. Each **Worker owns one instance** and uses it
to run a Sample's calculator commands with bounded concurrency, streaming I/O, and timeout
→terminate→kill fallback.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `async_subprocess.py` 679 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4;
[worker.md](worker.md), [calculator.md](calculator.md).
**Reuses V1**: none by import (reimplemented; depends on [`log_kv`](logger.md)).

---

## 1. Classes defined

### 1.1 `SubprocessRuntimeConfig` — `@dataclass`
Scheduler tuning, normalized in `__post_init__`.

| Field (default) | Meaning |
|-----------------|---------|
| `max_concurrency` (8) | worker tasks == concurrency cap |
| `max_pending` (256) | bounded pending queue (backpressure) |
| `queue_put_timeout_sec` (30) | enqueue timeout |
| `per_task_timeout_sec` (None) | default per-job timeout |
| `terminate_grace_sec` (5) | SIGTERM→SIGKILL grace |
| `chunk_size` (65536) | stream drain chunk |
| `log_policy` (`logger`) | `logger`/`file`/`quiet`/`tee-limited` |
| `progress_interval_sec` (5) | progress log cadence |
| `diagnostics_enabled` (False) / `diagnostics_interval_sec` (10) | status-file diagnostics |

### 1.2 `SubprocessJob` — `@dataclass`
One command. Fields: `cmd` (str|list), `cwd`, `env`, `shell` (True), `timeout_sec`, `log_dir`,
`log_policy`, `task_id` (auto uuid), `stream_logger`, `meta`.

Calculator jobs may also carry sample-command logging metadata in `meta`:

| Key | Meaning |
|-----|---------|
| `stage` | `install` / `initialize` / `execution`; used in command header and summary labels. |
| `command_index` | Per-calculator command counter; rendered as `[stage#00001]`. |
| `command_log_to_stream` | Route command header/output/summary to `stream_logger` (the sample logger) instead of only the scheduler logger. |
| `emit_command_summary` | Emit a raw `Command Summary -> [...]` line after process drain and before job completion. |

### 1.3 `SubprocessResult` — `@dataclass`
Outcome: `task_id`, `returncode`, `started_at`/`finished_at`, `duration_sec`, `timed_out`,
`stdout_path`/`stderr_path`, `stdout_bytes`/`stderr_bytes`, `cmd_display`. Property `ok` =
`returncode==0 and not timed_out`.

### 1.4 `SubprocessExecutionError(RuntimeError)`
Carries an optional `result`; raised on spawn failure (with an fd-pressure hint on errno 23/24).

### 1.5 `AsyncSubprocessScheduler`
Single-loop bounded scheduler (fixed worker count, bounded queue, streaming drain).

**Attributes:** `config`, `logger`, `status_path`; loop/thread plumbing `_loop`, `_thread`,
`_ready`, `_stopped`, `_shutdown_lock`, `_queue`, `_semaphore`, `_workers`, `_diagnostics_task`,
`_progress_task`, `_shutdown_started`; counters `_submitted`, `_completed`, `_failed`, `_running`,
`_peak_running`, `_timed_out`.

| Method | Behavior |
|--------|----------|
| `start()` | spawn the loop thread; wait until ready (10 s). |
| `snapshot() -> dict` | submitted/completed/failed/running/peak_running/timed_out/pending + best-effort `fd_count`/`rss_mb`. |
| `submit(job) -> Future` | thread-safe enqueue onto the loop; returns a `concurrent.futures.Future`; rejects while shutting down; surfaces queue-full as `TimeoutError`. |
| `arun(job) -> SubprocessResult` (`async`) | await a submitted job. |
| `run(job, timeout=None) -> SubprocessResult` | sync convenience (used by `CalculatorModule._run_command_sync`). |
| `shutdown(wait=True, timeout=30)` | drain + stop the loop; idempotent (lock-guarded). |
| `_thread_main()` | create loop, queue, semaphore, N worker tasks + progress/diagnostics tasks; `run_forever`; cancel pending + close on exit. |
| `_async_shutdown()` | `queue.join()`, poison workers, cancel progress/diagnostics, final status snapshot. |
| `_worker(worker_id)` (`async`) | pull jobs, run under the semaphore, set the future result/exception, update counters. |
| `_run_job(job)` (`async`) | spawn shell/exec subprocess (`start_new_session`), stream-drain stdout/stderr per `log_policy`, enforce timeout, write per-task `meta.json` when `log_dir` set, return `SubprocessResult`. |
| `_terminate_with_fallback(process)` (`async`) | `killpg` SIGTERM then SIGKILL. |
| `_progress_loop()` / `_diagnostics_loop()` (`async`) | periodic progress log / status-file append. |
| `_write_status_snapshot(reason)` | append a JSON status line to `status_path`. |

---

## 2. Module-level helpers

`_utc_now_iso`, `_safe_json`, `_emit_log_line`, `_normalize_cmd_for_shell`/`_for_exec`,
`_best_effort_fd_count`, `_best_effort_rss_mb`.

---

## 3. Per-Worker usage

Each Worker builds one scheduler in `_init_runtime` sized by the widest calculator layer
(`max_concurrency = max(subprocess_max_concurrency, layer_width)`) and runs it for the Worker's
lifetime. Calculators submit `SubprocessJob`s via `scheduler.run(...)`. Global calculator
concurrency is capped **separately** by the Redis free-pool ([redis_queue.md](redis_queue.md)); this
scheduler governs within-Worker command execution.

---

## 4. Concurrency / lifecycle / failure semantics

- One loop thread per Worker process; started lazily (spawn-safe — built in the child).
- Streaming drain prevents pipe deadlocks; timeout → terminate-then-kill; `ok` reflects rc +
  timeout.
- Bounded pending queue gives producer backpressure (`max_pending`).
- `snapshot()` is cheap (feeds the monitor / `peak_running`); shutdown drains in-flight then stops
  the loop — no orphan processes (`start_new_session` + `killpg`).

### 4.1 Sample command log ordering

For calculator commands, the scheduler must preserve this per-job emission order:

1. formatted command header (`Run execution command -> ... Screen output ->`);
2. raw stdout/stderr as the process produces it;
3. raw `Command Summary -> [stage#00001]`.

The command summary is emitted by the scheduler worker before setting the job result. This is the
ordering guard that prevents workflow-level logs from appearing between the external process output
and its summary.

The scheduler does not decide whether sample logs appear on the terminal. It emits to
`stream_logger`; `SampleLogger` decides whether to mirror to console based on the sample's
check-modules-only `sample_console` / `console=` flag. Full contract: [logger.md](logger.md)
§3–§4 (API + ownership, file vs console, block shape, ordering, meta keys).

---

## 5. Interfaces / collaborators

- **Worker** ([worker.md](worker.md)) owns one scheduler.
- **CalculatorModule** ([calculator.md](calculator.md)) submits jobs (env from
  [env_setup.md](env_setup.md)).
- **SampleLogger** ([logger.md](logger.md)) is the command-log sink for sample-local calculator
  command blocks.
- **log_kv** ([logger.md](logger.md)) formats progress/diagnostics blocks.

---

## 6. Tests

`tests/test_subprocess_scheduler.py` (6): concurrency bound, timeout→kill, env injection,
snapshot counters, per-Worker isolation, spawn safety.
