# Component — Core orchestrator (`jarvishep2/core.py`, `jarvishep2/client.py`)

**Role**: the `Jarvis2` run orchestrator. Wires config → Redis → CommandParser → Sampler →
Factory/Workers → Archiver → checkpoint, drives the scan, and tears everything down.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `core.py` 675 lines + `client.py` 133 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §0.1, §2.
**Reuses V1**: none by import.

> **As-built drift:** the design's 15-step `init_argparser → … → run` sequence is condensed into
> `bootstrap_distributed_runtime()` + `run()`. Supported task types are **check-modules** and
> **stateless distributed samplers** (Bridson/Random/Grid/CSV); `--benchmark`/`--convert`/`--plot`
> are **not** implemented (see [benchmark.md](benchmark.md)).

---

## 1. Class defined — `Jarvis2Core`

**Attributes** (from `__init__`): `config`, `runtime` (`get_runtime_block`), `info`, `redis`,
`factory`, `archiver`, `sampler`, `command_parser`, `_resume_checkpoint_payload`, `_resume_policy`
(`auto`/`resume`/`fresh`), `_logger`.

**Member functions:**

| Method | Behavior |
|--------|----------|
| `init_logger()` | `setup_jarvis_logging(role="core")`. |
| `load_task_yaml(path)` | load + normalize task YAML; refresh `runtime`/`info`. |
| `_populate_info_from_config()` | derive task_root / task_result_dir / scan_name / run_id. |
| `prepare_resume(*, resume=False, fresh=False)` | preload checkpoint state before sampler bring-up. |
| `init_sampler_from_config()` | pick sampler via `Distributor.set_method`, set config + execution-plan template, register on the core. |
| `bootstrap_distributed_runtime()` | logger → command_parser → redis → sampler → factory → archiver → checkpoint heartbeat (requires `Runtime.mode == redis`). |
| `_load_check_module_rows` / `_build_check_module_samples` | build fixed-point smoke samples from a CSV. |
| `run_distributed_scan(*, timeout=3600)` | drive a stateless sampler: repropose (resume) + `run_distributed`/`submit_all_remaining`, then wait for archived results. |
| `run_check_modules(*, timeout=120, verify_golden=None)` | submit smoke samples, wait, optionally verify golden. |
| `_verify_check_modules_golden` / `_sample_tree_file_sets` / `_normalize_check_module_records` | DATABASE + SAMPLE parity checks. |
| `run(*, resume=False, check_modules=False, verify_golden=None, write_run_summary=True)` | bootstrap → dispatch (check-modules / stateless / error) → `shutdown` in `finally`. |
| `check_modules(*, verify_golden=None)` | CLI entry for `--check-modules`. |
| `is_redis_runtime()` | `Runtime.mode == "redis"`. |
| `checkpoint_file()` | resolve the checkpoint path from task_root/scan_name/sampler. |
| `prompt_resume_from_checkpoint(...)` (`@staticmethod`) | interactive resume prompt (timeout-guarded). |
| `_preload_resume_checkpoint(*, resume=False, fresh=False)` | decide resume vs fresh; load + validate the checkpoint payload. |
| `apply_resume_checkpoint(worker_config=None)` | drain stale tasks, import sampler state, set repropose hint. |
| `save_runtime_checkpoint(*, reason="")` | build + atomically save the checkpoint payload (sampler state + archiver persistence). |
| `init_redis(*, client=None)` | ensure managed `Jarvis-Redis:<scan>` if port free, else connect existing; build control client. |
| `init_command_parser()` | Phase-1 `build_command_parser`. |
| `_command_parser_payload` / `_apply_command_parser_to_worker_config` | make the parser picklable + apply Phase-1 to calculator configs. |
| `build_worker_config(**overrides)` | picklable Worker blueprint with Phase-1 resolution applied. |
| `init_factory(worker_config=None)` | obtain the factory, reuse core Redis, `start_workers(Runtime.workers)` + monitor + watchdog (no-op unless redis runtime). |
| `init_archiver(db_path=None)` | start Archiver; **default `mode=process`** (`ArchiverProcess`); thread mode still supported. |
| `_restore_archiver_persistence` / `_archiver_records_written` | resume acked-uuids / read record counter. |
| `set_sampler(sampler)` | attach sampler, wire Redis, import resume state if resuming. |
| `start_runtime_checkpoint()` | enable the checkpoint heartbeat (save callback). |
| `submit_samples(samples)` | push a group via the sampler. |
| `wait_for_results(expected, …)` | poll Archiver `records_written`; progress shows ok/running/archive_q/archived; stall error if workers done but rows frozen. |
| `_init_sample_buckets` / `_finalize_sample_buckets` | Redis SAMPLE bucket meta; seal open bucket at end-of-run. |
| `_install_control_signal_handlers` | SIGINT/SIGTERM → clean shutdown; SIGTSTP/Ctrl+Z ignored. |
| `get_monitor_snapshot()` / `monitor_once()` | factory snapshot / formatted monitor view. |
| `write_run_summary(output_dir=None)` | build + write the run summary. |
| `shutdown(*, wait=True, write_run_summary=False)` | idempotent: stop sampler checkpoint, Archiver, factory/Workers, control lock, **always stop managed Redis**. |

---

## 2. `client.py` — CLI

| Function | Behavior |
|----------|----------|
| `build_parser()` | argparse: `task_yaml` (positional) + `--monitor`/`--check-modules`/`--resume`/`--pid`/`--redis-host`/`--redis-port`/`--redis-db`. |
| `_redis_cli_overrides` / `_apply_redis_overrides` | fold non-default Redis CLI flags into `core.config["Runtime"]["redis"]`. |
| `run_monitor(*, factory=None, redis=None)` | read + print one monitor view (1 if no active scan). |
| `dispatch_monitor(args)` | attach via the in-process factory or a fresh Redis client. |
| `dispatch_run(args)` | load YAML, apply Redis overrides, `check_modules()` or `run(resume=…)`. |
| `dispatch(args)` / `main(argv=None)` | route monitor vs run; entry point (`Jarvis2` console script). |

See [cli.md](cli.md) for the full CLI surface.

---

## 3. Shutdown ordering

`run()` installs signal handlers, then always calls `shutdown` in `finally` (including
KeyboardInterrupt from Ctrl+C).

Order inside `shutdown` (idempotent):

1. sampler `shutdown_checkpointing` (skip heavy finalize on interrupt)
2. optional `_finalize_sample_buckets` (seal open bucket) when not interrupting
3. stop Archiver (process join timeout shorter on interrupt)
4. optional `run_summary`
5. release Redis control lock
6. `factory.shutdown` (Workers)
7. **always** stop managed `Jarvis-Redis:<scan>` if this process started it (`atexit` backup)

Prefer **Ctrl+C**, not Ctrl+Z (suspend freezes Workers and leaves Redis half-alive).

---

## 4. Interfaces / collaborators

Ties together **RedisQueue**, **CommandParser**, **Distributor/Sampler**, **TaskFactory** +
**Worker**(s), **Archiver**, **runtime_checkpoint**, **dashboard/run_summary** — i.e. nearly every
other component.

---

## 5. Tests

`tests/test_core_run_distributed.py` (2), `test_distributed_acceptance.py` (6),
`test_distributed_resume.py` (8), `test_cli.py` (9): end-to-end opera/calculator runs, resume,
shutdown ordering, CLI dispatch + Redis overrides + monitor attach.
