# Component — Worker (`jarvishep2/worker.py`)

**Role**: the long-lived process that does all real work. Pulls one Sample at a time from
Redis, runs its workflow (same-layer calculators concurrent), and hands the result to the
Archiver. Holds calculator instances and Opera functions for its whole lifetime.
**Status**: **D1.1 implemented** (opera + likelihood MVP on `jarvis2`); **D1.2 partially
implemented** (calculator steps in-process, local `pack_id`); D2.1+ (free-pool, layer
concurrency, clone_shadow) not yet built.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4 (Locked
decision); discussions `worker_design.md` (primary), `Jarvis_HEP_High_Concurrency_Design_Blueprint.md`.
**Reuses V1**: `CalculatorModule` (`jarvishep/Module/calculator.py`), `AsyncSubprocessScheduler`
(`jarvishep/async_subprocess.py`), `Sample.materialize`, `OperasModule`, `LogLikelihood`.

---

## 1. Responsibilities

1. **Process lifetime** (one `multiprocessing.Process`, spawn ctx, persists for the whole run).
2. **Initialize once**: Redis client, global `UMapper`, one `CalculatorModule` per type
   (templates pre-loaded), Opera function cache, a **per-Worker `AsyncSubprocessScheduler`**.
3. **Main loop**: `blpop` one task → rebuild Sample → `bind_params` → `materialize` → run the
   `execution_plan` layer by layer (same layer concurrent) → compute LogL → stage → submit
   result → close. Then next task.
4. **Calculator concurrency**: acquire/release slots via the Redis free-pool so a heavy
   calculator's global concurrency is capped across all Workers; preserve `pack_id`.
5. **Isolation**: `clone_shadow` calculators get a per-Sample physical dir; safe tools run via
   symlink / `registered_executables`.
6. **Heartbeat + graceful shutdown + failure handling** (failed sample still leaves a log).

It must **not**: propose samples, allocate buckets (it receives a resolved `save_dir`/bucket
policy), or archive final products (it stages and hands off).

---

## 2. Class structure

```python
class Worker(multiprocessing.Process):
    def __init__(
        self,
        worker_id: int,
        redis: RedisQueue | Mapping[str, Any],   # live queue or picklable config
        worker_config: Mapping[str, Any],        # mapper, operas, calculators, sample_config
    ):
        super().__init__(name=f"HEP2-Worker-{worker_id}", daemon=False)
        self.redis_config = RedisQueue.extract_connection_config(redis)
        self.worker_config = dict(worker_config)
        # NO live Redis client, calculators, or scheduler here

    # ---- built inside run() (child process) ----
    _redis:       RedisQueue | None
    _mapper:      Any
    _operas:      dict[str, Any]
    _calculators: dict[str, CalculatorModule]
    _likelihood:  LogLikelihoodEvaluator | None
    _is_running:  bool
```

**Spawn rule**: `__init__` stores only picklable config; all live objects (Redis client,
calculators, scheduler, asyncio loop) are built in `run()` in the child process.

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `run` | `() -> None` | Process entry: `_init_*` then `_main_loop`. Installs signal handlers. |
| `_init_redis` | `() -> None` | Connect `RedisQueue`; register to `hep:worker:status:<id>`. |
| `_init_mapper` | `() -> None` | Build the global `UMapper` (`u → x`) from config. |
| `_init_calculators` | `() -> None` | One `CalculatorModule` per type; `preload_templates()`; attach the per-Worker scheduler. |
| `_preload_opera_funcs` | `() -> None` | `importlib` each opera once → `opera_funcs` cache. |
| `_register_calc_pools` | `() -> None` | Seed `calc:free:<name>` with each calculator's `make_paraller`. |
| `_main_loop` | `() -> None` | `while is_running:` `pull_task(timeout=5)`; `process_task`; heartbeat. |
| `process_task` | `(task: dict) -> None` | The core pipeline (see §4). |
| `_run_layer` | `(steps: list[ExecutionStep], sample) -> None` | Dispatch all steps in a layer concurrently via the scheduler; await the layer. |
| `_run_calculator_step` | `async (step, sample) -> None` | `acquire_calc` → `clone_shadow?` → `CalculatorModule.execute(sample.info)` → `release_calc` (in `finally`). |
| `_run_opera_step` | `(step, sample) -> None` | Call the cached opera func on `sample.info`. |
| `_run_likelihood` | `(sample) -> None` | Compute LogL (reuse `LogLikelihood`). |
| `_stage_and_submit` | `(sample) -> None` | `mv work_dir → staging/<uuid>`; `submit_result(sample.to_info_dict())`. |
| `_heartbeat` | `() -> None` | `redis.heartbeat(worker_id, status, current_sample, ts, pid)`. |
| `stop` | `() -> None` | Flip `is_running`; drain current task; close scheduler/logger; release held slots. |

---

## 4. process_task pipeline

```python
def process_task(self, task):
    sample = Sample.from_task_dict(task)
    top = get_jarvis_logger("worker").bind(worker_id=self.worker_id, sample_uuid=sample.uuid)
    try:
        sample.bind_params(self.mapper)
        sample.materialize(worker_id=self.worker_id)     # dirs + per-Sample logger
        for layer in group_by_layer(sample.execution_plan):
            self._run_layer(layer, sample)               # same-layer concurrent
        self._run_likelihood(sample)
        sample.status = "Completed"
    except Exception as e:
        sample.status = "Failed"
        sample.materialize_failure_artifacts(error=e)    # invariant #9
        top.error("sample failed; see sample log")
    finally:
        self._stage_and_submit(sample)   # submit_result bumps hep:sample:op_count once
        sample.close()
```

`_run_layer` uses the per-Worker `AsyncSubprocessScheduler` to run the layer's calculators
concurrently; opera/likelihood steps run inline (microseconds). One Sample is in flight at a
time — concurrency is **within** the Sample only (invariant #6).

> **`bind_params` call timing (current code).** In the D1.1 implementation `process_task` calls
> `sample.set_config(...)` and `sample.start()` first, then `sample.bind_params(self._mapper)`
> **before** `materialize`, and only when a mapper was built (`if self._mapper is not None`). The
> mapper itself is constructed once at startup in `_init_runtime` (`build_mapper`). `bind_params`
> is currently a reserved stub (UMapper lands later — see [sample.md](sample.md)), but the call
> site is fixed here so it does not move when the real `u → x` mapping arrives.

---

## 5. Concurrency, isolation, failure semantics

- **Calculator slots**: `acquire_calc(name)` blocks on `calc:free:<name>`; the slot count =
  `make_paraller`, capping global concurrency across all Workers. `pack_id` minted per acquire,
  released in `finally` (even on exception/timeout) — no slot leak.
- **clone_shadow** (D2.3): non-thread-safe tools get `decode_shadow_path`/`decode_shadow_commands`
  (reuse V1) to run in a per-Sample physical copy; safe tools symlink in.
- **Timeouts**: calculator `timeout` honored by the scheduler; on timeout the step fails, the
  slot is released, the Sample is marked Failed with a log.
- **Crash**: if the Worker process dies mid-Sample, the Factory watchdog (see [factory.md](factory.md))
  detects a stale heartbeat, sweeps the Worker's held slots, re-queues the in-flight Sample,
  and respawns. The Worker itself only guarantees slot release on *handled* exceptions.
- **Graceful stop (current code)**: `run()` installs handlers for both `SIGTERM` and `SIGINT`
  (`signal.signal(...)`); `_handle_signal` flips `self._is_running = False` only. The
  `_main_loop` checks the flag at the top of each iteration, so a signal received mid-Sample lets
  the current `process_task` finish, then the loop exits; the `finally` in `run()` sends a
  `stopped` heartbeat and closes the Redis client. The Factory's `request_worker_shutdown` /
  `stop_all_workers` drive this by sending `SIGTERM` (see [factory.md](factory.md) §5).

---

## 6. Interfaces

- **Redis** (`RedisQueue`): `pull_task`, `acquire_calc`/`release_calc`, `submit_result`,
  `heartbeat`, `incr_op`.
- **Factory**: spawns the Worker with picklable config; reads its heartbeat; respawns on death.
- **Sample**: `from_task_dict`, `bind_params`, `materialize`, `to_info_dict`, `close`.
- **CalculatorModule**: `preload_templates`, `execute(sample_info)`, shadow helpers (V1).
- **AsyncSubprocessScheduler**: `submit(SubprocessJob)` for layer concurrency.

---

## 7. Tests

`tests/test_worker_mvp.py` (D1.1, opera-only, `fakeredis` + real spawn):
1. **End-to-end** — one Worker drains N opera tasks; DATABASE records **set-equal** to the
   captured V1 golden output.
2. **Spawn safety** — Worker pickles/launches under `spawn`; Factory passes
   `connection_config()` only.
3. **Graceful stop** — `SIGTERM` and `SIGINT` finish the in-flight Sample then exit; no orphan.
4. **op_count semantics** — one `sample` op_count increment per completed Sample (via
   `submit_result` only).
5. **Factory read-only** — `_collect_latest_status` issues zero Redis writes on idle ticks.

`tests/test_worker_calculator.py` (D1.2):
4. **Calculator parity** — 1 Worker on `tests/parity_project` (`--check-modules` scale) →
   DATABASE + SAMPLE trees equal to the V1 golden.
5. **Token resolution** — `@Sdir`/`@PackID` resolve under the per-Sample save dir.

`tests/test_worker_pool.py` / `test_layer_concurrency.py` (D2):
6. **Free-pool cap** — a heavy calculator never exceeds `make_paraller` across 2+ Workers
   (assert via `CALC_STATUS`); slots always released (no leak after forced step failure).
7. **Layer concurrency** — an independent 2-calculator layer finishes in ≈max(t1,t2), not
   t1+t2 (timed); output identical to serial.
8. **clone_shadow** — concurrent samples on a shadow calculator get isolated dirs (no
   cross-talk).

Verification logic: tests 1/4 are the parity gates; tests 6/7 prove the slow-regime speedup
mechanism (global cap + in-Sample concurrency) without breaking parity.

---

## 8. Open questions

- Worker loop fully synchronous across layers vs. a thin asyncio wrapper (design §13.2 —
  proposed: async only inside a layer).
- Nuisance-parameter optimization loop inside the Worker (`worker_design.md` §7) — interface
  TBD; reserve an `ExecutionStep` type `nuisance_optimize`.
- Whether calculator instances re-install on config change mid-run (default: install is
  idempotent, keyed by slot).
