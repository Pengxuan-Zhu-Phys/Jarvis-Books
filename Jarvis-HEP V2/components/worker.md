# Component — Worker (`jarvishep2/worker.py`)

**Role**: the long-lived process that does all real work. Pulls one Sample at a time from Redis,
runs its execution plan (same-layer calculators concurrent), materializes SAMPLE artifacts under
a Redis-allocated bucket, submits results to the Archiver queue, and marks the bucket sample
finished.
**Status**: **As-built** @ `jarvis2` **`64d7486`**.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4.
**Reuses V1**: none by import — uses V2 `CalculatorModule`, `OperasModule`,
`AsyncSubprocessScheduler`, `LogLikelihoodEvaluator`.

---

## 1. Class defined — `Worker(Process)`

`Process` is the **spawn-context** process (`mp_context.get_spawn_context().Process`,
invariant #10). `__init__` stores **only picklable config**; every live object is built in `run()`
inside the child. OS title: `Jarvis2-Worker-<id>[:<scan>]` via `setproctitle`.

**Attributes** (set in `__init__`, populated in `run()`):

| Attribute | Built where | Meaning |
|-----------|-------------|---------|
| `worker_id: int` | init | numeric id |
| `redis_config: dict` | init | connection settings only |
| `worker_config: dict` | init | mapper / operas / calculators / sample_config / sample_directory |
| `_redis` | `_init_redis` | child-process Redis client |
| `_mapper` / `_operas` / `_calculators` / `_likelihood` / `_scheduler` / `_command_parser` | `_init_runtime` | per-worker runtime |
| `_delete_method`, `_staging_dir`, `_handoff_to_staging` | `_init_runtime` | default **handoff false** (`Cleanup.strategy=direct`) |
| `_sample_buckets_enabled` | `_init_runtime` | from `sample_directory.enabled` (default true) |
| `_held_calc_packs`, heartbeat fields | runtime | watchdog + exclusive PackID |

**Member functions (high level):**

| Method | Behavior |
|--------|----------|
| `run()` | set process title, logging, signals, `_init_redis` / `_init_runtime`, main loop |
| `process_task(task)` | allocate bucket → rebuild Sample → workflow → submit + finish bucket |
| `_run_layer` / calculator / opera / likelihood steps | same-layer calc concurrency |
| `_handoff_sample_to_staging` | **only if** `_handoff_to_staging` (optional legacy path) |
| `_stage_and_submit` | `redis.submit_result(to_info_dict())` (+ optional feedback) |
| `_force_release_all_held_packs` | always release PackIDs before handoff failures |

---

## 2. `process_task` pipeline (as-built)

```
pull task
 → allocate_sample_bucket()  # Redis: SAMPLE/<bucket>/ + active++
 → Sample.from_task_dict + set_config(sample_config with _bucket_parent)
 → start()                   # "ready for submittion" into Sample_running.log
 → bind_params / opera_params
 → materialize(worker_id)    # creates SAMPLE/<bucket>/<uuid>/ + Sample_running.log
 → for layer in execution_plan: _run_layer
 → status Completed | Failed
 finally:
   force_release held PackIDs
   sample.close()
   optional _handoff_sample_to_staging   # off by default
   submit_result
   finish_sample_bucket(bucket_id)      # active--; pack only after Archiver archives
   cleanup_transient_paths
```

**Work directory vs SAMPLE path:**

- Calculator **`path: …/@PackID`** → runtime cwd under pack slot (e.g. EggBox `001/`).
- **`@Sdir`** → sample `save_dir` when I/O is sample-scoped.
- SAMPLE tree is the **artifact / archive** root, not necessarily calculator cwd.

---

## 3. Concurrency, isolation, failure semantics

- **Calculator slots:** exclusive stable PackIDs (`001…N`); `force_release` on every sample.
- **Staging:** default **off**. Enable with `Cleanup.strategy: mv_to_staging`.
- **Buckets:** Worker never packs tar; only finishes Redis lifecycle counters.
- **Graceful stop:** SIGTERM/SIGINT flip `_is_running`; finish in-flight sample; `finally` closes Redis.
- **Crash:** Factory watchdog sweeps held packs + requeues.

---

## 4. Interfaces / collaborators

- **RedisQueue**: pull_task, allocate/finish bucket, acquire/release_calc, submit_result, heartbeat.
- **Factory**: spawn, heartbeat, watchdog.
- **Sample**: materialize / to_info_dict / close / Sample_running.log.
- **Archiver** ([datarecorder.md](datarecorder.md)): consumes archive queue; packs buckets later.
- **Portal / Operas / Likelihood / AsyncSubprocessScheduler**: Layer-1 execution.

---

## 5. Drift from early design sketches

- Default handoff is **direct**, not mandatory staging.
- Process title + scan-name suffix.
- Explicit PackID release before archive handoff.
- SAMPLE path is **bucketed** under Redis-owned ids.

---

## 6. Tests

`test_worker_mvp.py`, `test_worker_calculator.py`, `test_worker_pool.py`, `test_layer_concurrency.py`,
`test_worker_failure.py`, `test_clone_shadow.py`, plus `test_sample_bucket.py` for allocate/finish.
