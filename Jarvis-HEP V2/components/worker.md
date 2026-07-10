# Component — Worker (`jarvishep2/worker.py`)

**Role**: the long-lived process that does all real work. Pulls one Sample at a time from Redis,
runs its execution plan (same-layer calculators concurrent), stages products, and hands the result
to the Archiver. Holds calculator instances and Opera functions for its whole lifetime.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `worker.py` 393 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4; discussions
`worker_design.md`.
**Reuses V1**: none by import — uses the V2 `CalculatorModule`, `OperasModule`,
`AsyncSubprocessScheduler`, `LogLikelihoodEvaluator`.

---

## 1. Class defined — `Worker(Process)`

`Process` is the **spawn-context** process (`mp_context.get_spawn_context().Process`,
invariant #10). `__init__` stores **only picklable config**; every live object is built in `run()`
inside the child.

**Attributes** (set in `__init__`, populated in `run()`):

| Attribute | Built where | Meaning |
|-----------|-------------|---------|
| `worker_id: int` | init | numeric id |
| `redis_config: dict` | init | `RedisQueue.extract_connection_config(redis)` (settings only) |
| `worker_config: dict` | init | mapper / operas / calculators / sample_config / command_parser blueprint |
| `_redis: RedisQueue\|None` | `_init_redis` | child-process Redis client |
| `_mapper` | `_init_runtime` | `build_mapper(worker_config["mapper"])` |
| `_operas: dict` | `_init_runtime` | `preload_operas(...)` |
| `_calculators: dict[str,CalculatorModule]` | `_init_runtime` | `CalculatorModule.from_config_list(...)` |
| `_likelihood: LogLikelihoodEvaluator\|None` | `_init_runtime` | likelihood expressions |
| `_scheduler: AsyncSubprocessScheduler\|None` | `_init_runtime` | per-Worker subprocess scheduler |
| `_command_parser: CommandParser\|None` | `_init_runtime` | Phase-2 token resolver |
| `_delete_method`, `_staging_dir`, `_handoff_to_staging` | `_init_runtime` | cleanup/staging policy |
| `_observables_lock: threading.Lock` | `_init_runtime` | guards concurrent calc merges |
| `_is_running`, `_current_sample_uuid`, `_current_task`, `_held_calc_packs` | init/runtime | loop + heartbeat + watchdog state |

**Member functions:**

| Method | Behavior |
|--------|----------|
| `_handle_signal(signum, frame)` | flip `_is_running=False` (SIGTERM/SIGINT). |
| `_init_redis()` | build child Redis client + first idle heartbeat. |
| `_init_runtime()` | build mapper/operas/calculators/likelihood/scheduler/command-parser; resolve staging+delete policy; attach scheduler+parser+`env_setup` to each calculator. |
| `_heartbeat(status)` | `redis.heartbeat(...)` with pid, current_sample, ts, held_calc_packs (JSON), encoded current_task. |
| `_run_calculator_step(step_name, sample)` | `acquire_calc` (timeout) → record held pack → `prepare_runtime` → `execute(runtime_prepared=True)` → merge observables under lock → `release_calc` + drop pack in `finally`. |
| `_merge_calculator_observables(...)` | update `sample.observables` + `info["pack_ids"]`/`pack_id`. |
| `_force_serial_layers()` | rollback switch (`worker_config["force_serial_layers"]`). |
| `_run_calculator_steps(calc_steps, sample)` | fan out >1 same-layer calculators on a `ThreadPoolExecutor`; else serial. |
| `_run_layer(layer, sample)` | run calculators concurrently, then opera/likelihood steps inline. |
| `_run_opera_step(step_name, sample)` | call cached `OperasModule.execute`; merge observables. |
| `_run_likelihood(sample)` | `LogLikelihoodEvaluator.calculate(info)` → `sample._likelihood`. |
| `_collect_cleanup_paths` / `_cleanup_transient_paths` | gather + delete transient paths via `delete_paths`. |
| `_handoff_sample_to_staging(sample)` | metadata-only move of `save_dir` → staging (`stage_sample_dir`); record `staging_path`/`product_list`. |
| `_stage_and_submit(sample)` | `redis.submit_result(sample.to_info_dict())`. |
| `process_task(task)` | **core pipeline** (see §2). |
| `_main_loop()` | `while _is_running:` pull → process → heartbeat (idle/busy). |
| `run()` | process entry: `setup_jarvis_logging(role="worker")`, install signal handlers, `_init_redis`/`_init_runtime`, run loop, then drain scheduler + close Redis in `finally`. |

---

## 2. `process_task` pipeline

```
Sample.from_task_dict(task) → set_config(sample_config) → start()
 → bind_params(mapper)  (or copy opera_params)
 → materialize(worker_id)
 → for layer in group_by_layer(execution_plan): _run_layer(layer, sample)
 → status = Completed
 except: status = Failed; materialize_failure_artifacts(info, error); log
 finally: _handoff_sample_to_staging → _stage_and_submit → _cleanup_transient_paths
          → close(); clear current task / held packs
```

One Sample in flight at a time; concurrency is **within** a Sample (same-layer calculators on the
thread pool). An optional `JARVIS2_WORKER_PROFILE` env var appends per-layer timing JSON.

---

## 3. Concurrency, isolation, failure semantics

- **Calculator slots:** `acquire_calc(name)` blocks on `calc:free:<name>`; pack_id held in
  `_held_calc_packs`, released in `finally` (no leak). Same-layer calculators run on a
  `ThreadPoolExecutor`; `force_serial_layers` disables that.
- **Isolation:** `clone_shadow` calculators install per-pack physical dirs; safe tools symlink in
  (see [calculator.md](calculator.md)).
- **Staging:** materialized work dirs are moved to staging (fast, metadata-only) before submit, so
  the Archiver finalizes from a stable location ([datarecorder.md](datarecorder.md)).
- **Graceful stop:** SIGTERM/SIGINT flip `_is_running`; the loop finishes the in-flight Sample,
  then `run()`'s `finally` sends a `stopped` heartbeat and closes Redis.
- **Crash:** a dead Worker is detected by the Factory watchdog via stale heartbeat; held slots are
  swept and the in-flight task re-queued ([factory.md](factory.md)).

---

## 4. Interfaces / collaborators

- **RedisQueue** ([redis_queue.md](redis_queue.md)): pull_task, acquire/release_calc,
  submit_result, heartbeat.
- **Factory** ([factory.md](factory.md)): spawns with picklable config, reads heartbeat, respawns.
- **Sample** ([sample.md](sample.md)): from_task_dict / bind_params / materialize / to_info_dict /
  close.
- **CalculatorModule** ([calculator.md](calculator.md)), **OperasModule** ([operas.md](operas.md)),
  **LogLikelihoodEvaluator** ([likelihood.md](likelihood.md)),
  **AsyncSubprocessScheduler** ([subprocess_scheduler.md](subprocess_scheduler.md)),
  **archive_handoff** ([datarecorder.md](datarecorder.md)).

---

## 5. Drift from design

Init is one `_init_runtime` method (not the designed `_init_mapper`/`_init_calculators`/
`_preload_opera_funcs`/`_register_calc_pools`); there is **no `stop()`** (graceful stop is signal
+ `finally`). Layer concurrency uses a **`ThreadPoolExecutor`**, not the async scheduler, for
fan-out. New as-built: staging handoff, transient cleanup, held-pack heartbeat fields, the
`_observables_lock`, and the worker-profile hook.

---

## 6. Tests

`tests/test_worker_mvp.py` (17), `test_worker_calculator.py` (8), `test_worker_pool.py` (6),
`test_layer_concurrency.py` (5), `test_worker_failure.py` (2), `test_clone_shadow.py` (5):
end-to-end opera/calculator parity, spawn safety, graceful stop, free-pool cap, layer concurrency
timing, clone_shadow isolation, failure path.
