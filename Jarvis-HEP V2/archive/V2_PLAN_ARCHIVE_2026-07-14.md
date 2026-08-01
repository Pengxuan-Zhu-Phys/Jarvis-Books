# V2 Distributed Plan — Completed-WP Archive (extracted 2026-07-14)

Verbatim ledger rows and WP details for **completed** work packages, moved out of
[`V2_DISTRIBUTED_PLAN.md`](../V2_DISTRIBUTED_PLAN.md) to keep the active plan small.
Statuses here are frozen history; the live plan only tracks open work.

## Archived Progress Ledger rows (status: done)

| WP   | Title | Milestone | Depends on | Status | Date | Notes |
| ---- | ----- | --------- | ---------- | ------ | ---- | ----- |
| —    | M0/M1 (benchmark, lazy materialization, buffered logger, single executor, parity gate) | (V1)      | —                | **done — now part of frozen V1 1.7.4** | 2026-06    | Committed on the V1 line; **not** a V2 WP. V2 reuses the lazy-`Sample`/buffered-logger *ideas*, reimplemented on the V2 branch.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| D0.1 | New `Sample` model (uuid/u_coords/task_dict/info_dict)                                 | D0        | —                | **done**                               | 2026-06-29 | Shipped via D1–D7; `jarvishep2/sample.py` + `tests/test_sample_taskdict.py`. Lazy materialize, `to_task_dict`/`from_task_dict`, failure replay, `bind_params` placeholder.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| D0.2 | Redis schema + queue wrapper + serialization + `[distributed]` extra                   | D0        | —                | **done**                               | 2026-06-29 | `jarvishep2/redis_queue.py` (`calc:free:{name}`, atomic `HINCRBY` acquire/release, payload validation) + `tests/test_redis_queue.py`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| D0.3 | std-`logging` two-layer module (drop loguru)                                           | D0        | —                | **done**                               | 2026-06-29 | `jarvishep2/logging/` + `tests/test_logging_layers.py`, `tests/test_log_kv.py`. Queue-backed non-blocking sinks, `key=value` contract.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| D0.4 | D0 review fixes + test backfill (gate D0)                                              | D0        | D0.1, D0.2, D0.3 | **done**                               | 2026-06-29 | Redis namespace/race fixes + D0 unit tests green (`test_redis_queue`, `test_sample_taskdict`, `test_logging_layers`). Verified 2026-06-29.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| D0.5 | D0 wrap-up: defensive hardening + spawn-pickling + integration test + polish           | D0        | D0.4             | **done**                               | 2026-06-29 | `tests/test_d0_integration.py` (round-trip + `SpawnPicklingTests`), spawn pickling in `test_worker_mvp.py`. **D0 closed.**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| D1.1 | TaskFactory skeleton + Worker process (opera-only MVP)                                 | D1        | D0.4             | **done**                               | 2026-06-28 | All six §WP-D1.1 acceptance gates green under `fakeredis` + spawn (`tests/test_worker_mvp.py`, 17 tests). **Shipped:** `TaskFactory` lifecycle + `op_count`-gated monitor (~120 Hz), explicit `connection_config()` spawn boundary, `shutdown` (monitor stop, snapshot clear, Redis close), `Worker` opera+likelihood pipeline, single `sample` op_count via `submit_result`, SIGTERM+SIGINT graceful stop, Factory read-only monitor path, captured-V1 golden DATABASE parity. **Out of scope (later WPs):** watchdog (D6.1), dashboard/`get_run_metrics` (D5.2). Docs: [`factory.md`](components/factory.md), [`worker.md`](components/worker.md).                         |
| D1.2 | Calculator in Worker (`preload_templates`, `execute`) + single-Worker parity           | D1        | D1.1             | **done**                               | 2026-06-28 | `jarvishep2/Module/calculator.py` + `worker.py` calculator layer on `jarvis2`. **Shipped:** held `CalculatorModule` per type (`from_config_list` + `preload_templates`), `_run_calculator_step` via local `mint_pack_id` (D2.1 upgrades to Redis free-pool), `@Sdir`/`@PackID` token resolution, `build_execution_plan` (calculator → likelihood), parity vs `tests/fixtures/parity_m1/` + `tests/parity_project` (10-point check_modules scan), `tests/test_worker_calculator.py` (6 tests). **Out of scope:** cross-Worker free-pool, layer concurrency (D2).                                                                                                              |
| D2.1 | Multi-Worker pool + held calculators + Redis free-pool + `pack_id`                     | D2        | D1.2             | **done**                               | 2026-06-28 | `jarvishep2/factory.py` (`start_workers`, `register_calculator_pools`), `jarvishep2/worker.py` (`acquire_calc`/`release_calc`), `jarvishep2/calculator_pools.py`, `tests/test_worker_pool.py` (3 tests). **Shipped:** N Workers via spawn, Redis `calc:free:<name>` slot cap across Workers, fresh `pack_id` per acquire, `hep:calculator:status` free/busy updates, 2-Worker EggBox DATABASE parity, busy-count never exceeds configured slots. **Out of scope:** layer concurrency (D2.2), clone_shadow (D2.3).                                                                                                                                                            |
| D2.2 | Layer-internal calculator concurrency (per-Worker `AsyncSubprocessScheduler`)          | D2        | D2.1             | **done**                               | 2026-06-28 | `jarvishep2/async_subprocess.py`, `jarvishep2/worker.py` (`_run_layer`, `force_serial_layers`), `jarvishep2/workflow.py` (`resolve_module_layers`), `jarvishep2/Module/calculator.py` (`attach_scheduler`). **Shipped:** per-Worker scheduler, same-layer calculator fan-out, `required_modules` layer derivation, timed parallel-vs-serial test + observables parity (`tests/test_layer_concurrency.py`), scheduler unit tests (`tests/test_subprocess_scheduler.py`), workflow plan tests (`tests/test_workflow_execution_plan.py`). Rollback: `force_serial_layers: true`. **Out of scope:** cross-Sample concurrency, explicit YAML layer override, clone_shadow (D2.3). |
| D2.3 | `clone_shadow` isolation + LibDeps symlink path                                        | D2        | D2.1             | **done**                               | 2026-06-29 | `jarvishep2/Module/calculator.py` (`decode_shadow_*`, `prepare_runtime`, `ensure_shadow_installed`), `jarvishep2/library.py` (`link_into_sample`), `jarvishep2/worker.py` (runtime prep before execute), `tests/test_clone_shadow.py` (7 tests). **Shipped:** per-pack physical shadow install, safe-tool symlink into `@Sdir`, concurrent isolation (`isolation_count=1`), EggBox parity preserved. **Out of scope:** `registered_executables` (D3.1), full LibDeps boot install.                                                                                                                                                                                           |
| D3.1 | `registered_executables` + two-phase `CommandParser`                                   | D3        | D1.2             | **done**                               | 2026-06-29 | `jarvishep2/command_parser.py`, `base.py`, `worker_config.py`, `core.init_command_parser`, Worker+Calculator Phase 2, `tests/test_command_parser.py` (8 tests). **Shipped:** Phase-1 static tokens + `registered_executables` (`direct_path`/`symlink`); Phase-2 `@SampleID/@Sdir/@PackID`; EggBox parity via `eggboxlk`. **Out of scope:** `env_setup` (D3.2). |
| D3.2 | `env_setup` capture-from-source + cache                                                | D3        | D3.1             | **done**                               | 2026-06-29 | `jarvishep2/env_setup.py` (`EnvCapture`), `Module/calculator.py` (`bind_env`, `SubprocessJob env=`), `worker.py` (bind at `_init_runtime`), `tests/test_env_setup.py` (9 tests). **Shipped:** `source && env` capture, per-Worker cache, merge order, subprocess visibility, rollback when `env_setup` absent. **Out of scope:** general shell backend. |
| D3.3 | `Runtime.FileOperation.delete_method`                                                  | D3        | D3.2             | **done**                               | 2026-06-29 | `jarvishep2/file_ops.py` (`delete_path`, `delete_paths`, `normalize_delete_method`), `runtime_config.py` (`FileOperation` parse, `get_delete_method`), `worker.py` (`_cleanup_transient_paths` on `cleanup_paths`/`staging_paths`), `archiver.py` (`cleanup_staging`), `core.init_archiver` wiring, `tests/test_file_ops.py` (15 tests). **Shipped:** `shutil` default + `rm` backend; bad method raises; Worker + Archiver call sites. **Out of scope:** rsync/tar backends. |
| D4.1 | Worker staging mv + Archiver process skeleton + handoff                                | D4        | D2.1             | **done**                               | 2026-06-29 | `jarvishep2/archive_handoff.py`, `worker.py` (`_handoff_sample_to_staging`), `archiver.py` (`ArchiveProcessor`, `ArchiverProcess`), `runtime_config.py` (Cleanup/Archiver parse), `core.init_archiver`, `tests/test_archiver_handoff.py` (5 tests). **Shipped:** `mv_to_staging` default, staging→SAMPLE handoff, thread+process archiver modes, EggBox parity preserved. **Out of scope:** batching (D4.2). |
| D4.2 | Archiver batch persistence (HDF5/CSV/SAMPLE) + output parity gate                      | D4        | D4.1             | **done**                               | 2026-06-29 | `archiver.py` (`batch_size`, `flush_interval_sec`, `flush_batch`), `tests/test_archiver_parity.py` (2 tests). **Shipped:** batch-size-3 EggBox DATABASE+SAMPLE parity, interval flush for partial batches. **Out of scope:** SLHA/JSON/xSLHA/CSV conversion (Jarvis-Portal I/O backend per V1 — see `io_system.md`); idempotent restart gate (D6.2). |
| D5.1 | `op_count` writers + TaskFactory background snapshot                                   | D5        | D2.1             | **done**                               | 2026-06-29 | `jarvishep2/factory.py` (`_collect_latest_status`, `get_monitor_snapshot`, `get_run_metrics`), `tests/test_monitor_snapshot.py` (5 tests) + `tests/test_worker_mvp.py` snapshot gates. **Shipped:** op_count-gated HGETALL refresh (~120 Hz updater), in-memory deepcopy read path, read-only collect invariant, run-metrics projection from `hep:sample:stats` + queue lengths. **Out of scope:** Textual dashboard UI (D5.2 optional extra). |
| D5.2 | `--monitor` dashboard + run_summary from Redis                                         | D5        | D5.1             | **done**                               | 2026-06-29 | `jarvishep2/dashboard.py`, `jarvishep2/client.py`, `jarvishep2/monitoring/run_summary.py`, `jarvishep2/core.py` (`monitor_once`, `write_run_summary`), `tests/test_dashboard_reader.py` (6 tests). **Shipped:** `SnapshotReader` + `MonitorView`, `Jarvis2 --monitor` one-shot CLI, frozen `run_summary.{json,csv,txt}` schema, no-scan clear exit. **Out of scope:** Textual TUI, Prometheus/web exporter. |
| D6.1 | Heartbeats + dead-Worker respawn + in-flight re-queue                                  | D6        | D2.1             | **done**                               | 2026-06-29 | `jarvishep2/factory.py` (`start_watchdog`, `_handle_worker_failure`), `jarvishep2/worker.py` (`last_heartbeat`, `current_task`, `held_calc_packs`), `jarvishep2/redis_queue.py` (sweep/decode helpers), `runtime_config.py` (`Runtime.Watchdog`), `tests/test_worker_failure.py` (2 tests). **Shipped:** stale/process-exit detection, calc slot sweep, bounded in-flight re-queue, Worker respawn. **Out of scope:** checkpoint/RNG (D6.2). |
| D6.2 | RNG spawning + distributed checkpoint/resume                                           | D6        | D6.1             | **done**                               | 2026-06-29 | `jarvishep2/Sampling/runtime_checkpoint.py`, `jarvishep2/Sampling/seeded_sampler.py`, `jarvishep2/core.py` (`_preload_resume_checkpoint`, `apply_resume_checkpoint`, `save_runtime_checkpoint`, `start_runtime_checkpoint`), `jarvishep2/archiver.py` (idempotent SAMPLE skip + DATABASE write), `jarvishep2/redis_queue.py` (`drain_task_queue`), `tests/test_distributed_resume.py` (6 tests). **Shipped:** `jarvis-hep.v2-distributed` payload, master `SeedSequence`, resume drain + re-propose, V1/throughput-core refusal, 30 s heartbeat UX frozen, worker-count-independent trajectory. **Out of scope:** Redis RDB/AOF durability (§13.5). |
| D7.1 | Slow-regime acceptance: scaling + archive latency + chaos + parity                     | D7        | D4.2, D5.2, D6.2 | **done**                               | 2026-06-29 | `tests/test_distributed_acceptance.py` (6 gates), `tests/benchmark_project/bin/acceptance_slow.py`, `docs/benchmarks/d7_1_acceptance.json`. **Shipped:** worker scaling (~linear 1→2), bounded staging backlog, SIGKILL chaos + parity, EggBox DATABASE/SAMPLE/CSV golden parity, monitor ≥60 Hz read path. **Out of scope:** 256-Worker hardware gate (scaled to host `cpu_count`). |
| D9.1 | Token resolution single-owner + delete sync/async twins + `CommandParser.to_picklable` | D9        | —                | **done**                               | 2026-07-10 | CommandParser sole token owner + `to_picklable`; calculator sync-only (async/no-scheduler twins removed). |
| D9.2 | `Distributor` registry + IO format registry (unknown type errors)                     | D9        | —                | **done**                               | 2026-07-10 | Portal IO bridge + Distributor register table. |
| D9.3 | `CalculatorModule` split (Spec / RuntimePreparer / thin orchestrator)                 | D9        | D9.1, D9.2       | **done**                               | 2026-07-10 | `calculator_spec.py` + `runtime_preparer.py` + orchestrator. |
| D9.5 | `Jarvis2Core` cleanup: golden verification → tests/, check_modules column inference   | D9        | D8.2             | **done**                               | 2026-07-10 | Golden verify + CSV column inference in `testing/check_modules.py`. |
| D9.7 | `FixedSetSampler` template base + absorb `stateless_batch` helpers                    | D9        | —                | **done**                               | 2026-07-10 | Bridson/Random/Grid share FixedSetSampler; batch flush on base. |
| D9.8 | Smells sweep (import cycle, Archiver `from_config`, dead imports, ABC)               | D9        | —                | **done**                               | 2026-07-10 | `parse_registered_executables` on command_parser; Archiver `from_config`/`has_pending`; ABC barrier. |
| D10.1 | `hep:feedback` channel: `publish_feedback`/`pull_feedback`, worker flag, drain-on-resume | D10       | —                | **done**                               | 2026-07-11 | `FEEDBACK_QUEUE`, worker `publish_feedback` opt-in (auto-on for AdaptiveLevelSet). |
| D10.2 | Sampler core (d=2 flagship): generation loop, `NeighborGraph` + `DelaunayGraph`, crossing detection, deterministic edge-refinement budget; `core.run_adaptive` dispatch for `stateless=False` | D10       | D10.1            | **done**                               | 2026-07-11 | `adaptive_level_set.py`; `core.run_adaptive_scan`; registered `stateless=False`. |
| D11.4a | Eggbox Bridson + Operas V1-YAML vertical slice                                         | D11       | —                | **done**                               | 2026-07-13 | Unmodified `Example_Bridson_Operas.yaml`: missing Runtime → Redis, `Pi` expression parity, task-root env override; real Redis run 10,034/10,034, 0 failed. Follow-up caches Operas inputs once per Worker and selection once per expression/variable set; second smoke run 10,026/10,026, 0 failed. Does not close D11.4 or other samplers. |
| D11.4b | Shared object-oriented expression runtime                                              | D11       | D11.4a           | **done**                               | 2026-07-13 | `ExpressionContext` + immutable `CompiledExpression`; one `sympify`/`lambdify` implementation for Operas, Likelihood, Calculator/Portal, Selection, and AdaptiveLevelSet. YAML unchanged; domain error/coercion behavior retained. Full suite 266 passed + 1 skipped; real V1 Eggbox rerun 10,061/10,061, 0 failed. |
| D11.4c | V1 lightweight Expression Core migration                                               | D11       | D11.4b           | **done**                               | 2026-07-13 | All 38 V1 built-in function names + `Pi/E/Inf` migrated into `inner_func`; all-name tests, vector Gaussians, released GMFit expression, and direct V1 numerical audit. Full suite 270 passed + 1 skipped; real Eggbox rerun 10,049/10,049, 0 failed. Dynamic extensions subsequently completed in D11.4d. |
| D11.4d | Jarvis-Operas dynamic expression-function discovery                                   | D11       | D11.4c           | **done**                               | 2026-07-13 | Demand-gated per-process entry-point/persisted registry discovery; qualified names available everywhere; V1 `Operas.Modules` YAML shape unchanged; expressions and module operators both use direct NumPy-callable hot paths. Full suite 280 passed + 1 skipped; dedicated spawn-Worker discovery test passed. D11.4 still retains strict call/signature/logger and `list/info` UI work. |
| D12.7 | Slot-safety hardening: calculator process groups + in-execute heartbeat                 | D12       | —                | **done**                               | 2026-07-14 | Landed with the stable-PackID slot commit (same review): (a) scheduler registers active child PIDs (children already ran with `start_new_session=True`); Worker heartbeat carries `active_subprocess_pids`; `_handle_worker_failure` reaps orphan process groups (`_kill_orphan_process_groups`, session-leader guard against pid recycling) after force-stop and before slot sweep; (b) Worker periodic heartbeat thread (`heartbeat_interval_sec`, default 5 s) keeps long `module.execute()` alive past `Watchdog.stale_sec`; heartbeat snapshots now lock-guarded. Also atomic `release_calc` HDEL guard, kill-before-sweep ordering, corrupted-pool raise. Tests: scheduler PID tracking, setsid orphan reap + non-leader skip, stop→killpg→sweep ordering, 3 s calculator vs 1 s stale threshold survives with 0 respawns. Idempotent-installation convention documented in YAML_REFERENCE §9.4. |
| D9.4 | `TaskFactory` de-singleton + internal MonitorLoop/Watchdog split + honest metrics     | D9        | —                | **done**                               | 2026-07-16 | Core owns explicit factory; `get_instance` deprecated shell kept. Private `_MonitorLoop` + `_Watchdog` composition; recovery stays on factory facades (duck-typed unit stubs). Honest `None` metrics already; monitor tests use explicit `TaskFactory()`. Compat property aliases for pre-D9.4 private fields. |
| D9.6 | `Sample` single source of truth (`info` becomes projection)                           | D9        | D9.1, D9.3       | **done**                               | 2026-07-16 | Dual-truth collapse: `_mirror_fields_to_info` shares `params`/`observables` object identity with `info`; Worker uses `set_status`/`adopt_params`/`record_failure`/`record_handoff`/`pull_dual_truth_from_info` (no bare dual-key writes). `format_summary` stays on Sample (Occam). `sample→runtime_config` unlink deferred with F11. |
| D10.3 | Finalize + outputs (d=2): polyline chaining, `levelset.json`, PLOT overlay hook       | D10       | D10.2            | **done**                               | 2026-07-15 | `levelset.json` + polylines_u/x (chaining); stock **jplot** YAML under `images/<scan>_levelset_jplot.yaml` (scatter + level-set `plot` layers); samples CSV export; optional auto-render when JarvisPLOT installed; manual `Jarvis2 plot …`. Tests: `test_flowchart_and_plot_scene.py`. |
| D10.4 | Determinism + checkpoint/resume + core test suite (§9.1–9.8); YAML_REFERENCE §6.9 entry | D10       | D10.2            | **done**                               | 2026-07-16 | §9.1–9.8 covered in `tests/test_adaptive_level_set.py` (Hausdorff circle, precision mono, efficiency ≤30% dense Bridson, export/import + resume-continues, failure region, kNN/Delaunay cloud Hausdorff, feedback units). `run_adaptive` resume-safe (no gen-0 reset when `_points` present). YAML_REFERENCE §6.9 + Key Index already landed (verified). Fixtures: `ellipse_s`, `circle_r2_region_fail`. |
| D10.5 | Dimension extension d=3–5: `KNNGraph` proximity mode, Sobol gen-0 (d=5; Bridson grid is ~344 MB at d=5 — C10), d=3 crossing cloud, slice projections, §9.9–9.10 tests | D10       | D10.2            | **done**                               | 2026-07-16 | Core paths already landed; fixtures `sphere_r` / `hypersphere_r`; tests: dim defaults (Delaunay/KNN + factor), d=3 sphere cloud, d=4 knn+slices+shell error + near-band vs uniform, d=5 Sobol gen-0 power-of-two. Note: design’s strict 5× near/far density not used as CI hard gate (volume samples remain after gen-0; proximity cloud accuracy is the binding §9.10 check). |
| D11.1 | Truthful run result + machine-readable failure contract (`RunOutcome`)                 | D11       | —                | **done**                               | 2026-07-14 | `jarvishep2/run_outcome.py`; `core.run`/`check_modules` return `RunOutcome`; CLI exit uses completed/failed (all-fail & partial → 1; interrupt → 130); Worker stamps `error`/`error_type`/`failed_module` on Failed. Tests: `test_run_outcome.py`, CLI all-fail regression. |
| D11.2 | One-intent CLI + compatibility aliases + Redis override fix                            | D11       | D11.1            | **done**                               | 2026-07-14 | Subcommands `run/check/monitor/plot/portal/operas`; `normalize_argv` rewrites bare YAML; legacy flags kept; mode conflicts → exit 2; `--pid` rejected; thin `portal formats` / `operas list|info`. Tests: `test_cli.py`. Redis host override still open under D12.4. |
| D11.3 | Portal HEP format exposure + discovery + real SLHA fixture                             | D11       | D11.2            | **done**                               | 2026-07-15 | Portal **1.4.0**: `jarvis_portal.v1` / `jarvis_portal.v2` separate interface packages (no shared factories); adapters shared. V2 surface exposes SLHA+xSLHA; V1 surface unchanged (5 base formats). HEP-v2 uses `jarvis_portal.v2`; fixture `tests/test_io_slha.py`. `Jarvis2 portal formats` lists SLHA when Portal≥1.4. |
| D11.4 | Operas strict validation + signature/logger/Functions parity + discovery               | D11       | D11.2            | **done**                               | 2026-07-14 | `normalize_call_mode` (`call`/`acall` only); `filter_operator_kwargs` (signature filter + friendly missing-param; `**kwargs` passthrough); logger/`observables` only if accepted; CLI `operas list|info` from D11.2. Tests in `test_operas_bridge.py`. |
| D11.5 | Scan-driven PLOT scene + workflow graph + AdaptiveLevelSet overlay                      | D11       | D10.3, D11.2     | **done** (MVP)                         | 2026-07-14 | `flowchart.py` plan→`flowchart.json` + optional PNG; `plot_scene.py` scatter from HDF5 + levelset overlay YAML; core emits after successful run; `--skip-draw-flowchart`. Full V1 IO-port graph / auto-render of scatter still optional. Tests: `test_flowchart_and_plot_scene.py`. |
| D12.0 | Jarvis-Operas → core dependency + expression-scan scope fix                             | D12       | —                | **done**                               | 2026-07-14 | (a) `Jarvis-Operas>=1.3.0` is a core dep in `pyproject.toml`; `[operas]` extra kept as deprecated alias. README/INSTALL updated. (b) `expression_uses_operas_function` only treats bare strings as formula text at the call site or under `expression`/`selection`/`target_expression` keys — calculator `cmd`/install strings no longer force discovery. Tests: `test_operas_functions.py` gate cases. Argument-resolution layer remains D11.4. |
| D12.1 | Calculator V1-YAML parity (string commands, `${source}/${path}`, module `selection`)    | D12       | D12.0, D12.2 (accept); D11.3 (SLHA complete) | **done** (JSON path)          | 2026-07-14 | Committed `15f8ef4` + later runtime fixes. Review §4.0/§4.1. String cmds, `${source}/${path}`, module `selection`, `make_paraller` pools. Live Eggbox Bridson calculator path OK. SLHA complete still needs D11.3. |
| D12.2 | Core logging V1 visual parity (formatter, banner, file layout)                          | D12       | D12.0            | **done**                               | 2026-07-14 | `481fa97`. V1-style formatter/logo/scan log path. Tests: `test_logging_layers.py`. |
| D12.3 | Workflow flowchart export + JarvisPLOT rendering                                        | D12       | D11.5            | **done**                               | 2026-07-15 | V1-parity `jarvisplot.scene/v1` semantic graph from task YAML (Parameters/vars/file ports/Dump free-symbols/bridges/selection); stock `jarvisplot.render_flowchart` — no V2 PLOT adapter. Paths `<scan>/images/flowchart.{json,png}`; `--skip-draw-flowchart`. Tests: `test_flowchart_and_plot_scene.py`, `test_workflow_execution_plan.py`. |
| D12.4 | `EnvReqs.V2` grouped settings (workers/factory/redis override)                          | D12       | —                | **done**                               | 2026-07-15 | Whitelist: workers, batch_size, sample_directory, cleanup, archiver, **redis**, **factory**, **worker**. Optional `redis.{host,port,db}` overlays INTERNAL_REDIS_CONFIG; `factory.{monitor_hz,watchdog}`; `worker.{force_serial_layers,sample_artifacts}` (scalar `worker: N` still aliases workers). Legacy EnvReqs.Runtime defaults strip unknown keys. Tests: `test_task_config_compat.py`. |
| D12.5 | `Jarvis2 project` subcommands (create/pack/browse/fetch/info)                            | D12       | D11.2            | **done**                               | 2026-07-15 | Ported `project_scaffold` / `project_packager` / `official_project_library` + `project_template/` (V2 EnvReqs.V2 quickstarts). CLI: `Jarvis2 project create|pack|browse|fetch|info`. Catalog packaged fallback + schema_version guard. Tests: `test_project_tools.py`. |
| D12.6 | Jarvis-Examples-owned official catalog (schema-versioned remote index)                  | D12       | D12.5            | **done**                               | 2026-07-15 | Catalog = GitHub JSON only (no PyPI): `Jarvis-Examples/catalog/official_project_library.json`. V2 default index URL → Examples raw; offline user cache `~/.jarvis/cache/` + packaged snapshot. **Access:** `public` vs `restricted` + `requires_key`; list/browse table shows Key column; fetch decrypts OpenSSL AES-256-CBC via system openssl (`--key` / `JARVIS_PROJECT_FETCH_KEY`). Tests: `test_project_tools.py`. |
| D12.8 | SAMPLE buckets + direct handoff + process Archiver + process titles                     | D12       | D12.1            | **done**                               | 2026-07-14 | Commit **`64d7486`**. Defaults: `Cleanup.strategy=direct` (staging optional), `Archiver.mode=process` + `pack_buckets=true`, `Scan/EnvReqs.V2.sample_directory` limit=200. Redis bucket meta (`active`/`completed`/`archived`/`sealed`); **pack only when `archived>=assigned`** (fixes early-tar prune race). Managed `Jarvis-Redis:<scan>` + `setproctitle` (`Jarvis2*`). Tests: `test_sample_bucket.py`, `test_redis_server.py`. Live Eggbox Bridson ~8k samples drains cleanly. |
| D13.1 | `FeedbackSampler` base extracted from ALS + porting guide                              | D13       | —                | **done**                               | 2026-07-16 | `jarvishep2/Sampling/feedback_sampler.py` + `components/feedback_sampler.md`. ALS subclasses `FeedbackSampler`; suite `test_adaptive_level_set.py` + `test_feedback_sampler.py` green. |
| D13.2 | `Source/MCMC` chain runtime port + `MCMC`/`AM`/`DRAM` methods                          | D13       | D13.1            | **done**                               | 2026-07-16 | Engines under `jarvishep2/Sampling/Source/MCMC/`; `MCMCSampler` on FeedbackSampler; methods MCMC/AMMCMC/AM/DRAM registered; worker-count trajectory test green; `tests/test_mcmc_sampler.py`. Full V1 golden DRAM parity is progressive (diag + e2e path live). |
| D13.3 | Ensemble family: stretch / DE / parallel tempering                                      | D13       | D13.2            | **done**                               | 2026-07-17 | `EnsembleChain`/`DEMCMCChain` engines; methods EnsembleMCMC/Ensemble/DEMCMC/PTMCMC/PT/PTEnsemble; half-ensemble barriers + PT exchange; `tests/test_ensemble_samplers.py`. V1 golden moments / wall-clock gate progressive under D13.6. |
| D13.4 | `nuisance_optimize` Worker step + pass-condition                                        | D13       | —                | **done**                               | 2026-07-17 | `Module/nuisance.py` + `profile1d.py`; Worker step + plan/flowchart; Profile1D golden-section; pass-condition gate; `tests/test_nuisance_optimize.py`. Full HinoLLP physics golden progressive under D13.6. |
| D13.5 | Dynesty bridge (`RedisEvaluationPool`) + checkpoint wrap                                | D13       | D13.1            | **done**                               | 2026-07-17 | Vendored dynesty 3.0.0 + UUID channel; `RedisEvaluationPool`; `DynestySampler` registered. Accept progressive: full DynamicNestedSampler batch uuid + native resume + V1 logZ golden under D13.6. |
| D13.5b | MultiNest (static NestedSampler, V1 name) + multinest_result.csv / jplot              | D13       | D13.5            | **done**                               | 2026-07-17 | `MultiNestSampler` subclasses Dynesty path; always static; `DATABASE/multinest_result.csv` + `dynesty_runplot` jplot. Not Fortran MultiNest. Tests: `test_multinest_sampler.py`. |
| D13.6 | D13 acceptance + docs closure (YAML_REFERENCE §6 methods, samplers_catalog, DATABASE chain columns) | D13 | D13.2–D13.5 | **done** | 2026-07-20 | Diagnostics export + acceptance tests committed (`f10889b`); DATABASE `sampler_summary.json` / `chain_history.csv` live. |
| D13.7 | D13 review fixes: fail-loud pool dispatch, feedback drop logging, nuisance re-run default, MCMC ESS | D13 | D13.6 | **done** | 2026-07-20 | (a) `RedisEvaluationPool.map` fail-loud on ambiguous dispatch; (b) unmatched `hep:feedback` warning; (c) `re_run_physics` default **true** (V1 NAttempt full-pipeline parity) + YAML_REFERENCE §6.13; (d) per-chain `ess_logl` / `ess_logl_mean` in MCMC summary. |
| D13.8 | Configurable flat feedback return (`{uuid, logL}` default; −∞ for unusable; flat extra fields) | D13 | D13.7 | **done** | 2026-07-20 | Implemented: `feedback_return.py`, Worker projection, Redis flat wire, likelihood `LogL=-inf` fallback, consumers (pool/MCMC/ALS). Design: [`DESIGN_FEEDBACK_RETURN_2.0.md`](DESIGN_FEEDBACK_RETURN_2.0.md). |
| D13.9 | Task YAML validation gate (early fail; Method/Variables/Bounds/Archiver; `Jarvis2 validate`) | D13 | D13.8 | **done** | 2026-07-21 | Pure `task_validation` + `contracts/*`; run/check path validates before Redis; `--strict` / `--json`; dead-key warnings. Design: [`DESIGN_YAML_VALIDATION_2.0.md`](DESIGN_YAML_VALIDATION_2.0.md). (Design doc originally labeled “D14”; plan **D14** remains cluster execution.) |
| D13.10 | Nested UX freeze + AdaptiveBridson + check-modules inspect layout | D13 | D13.9 | **done** | 2026-07-21 | (a) vendored dynesty **3.1.0**; (b) Method=engine — Dynesty always Dynamic, MultiNest always static; `Bounds.dynamic` rejected (`JV2-BND-012`); (c) Sampling Simple/Full templates narrowed; (d) **AdaptiveBridson** renames AdaptiveLevelSet (no alias); (e) `Jarvis2 check`: workers=1, `SAMPLE/test/<uuid>/` flat, pack off; CSV-or-N-samples smoke. |
| D17.1–D17.5 | Strict task-card validation (vocabulary, zones, numeric, error UX, corpus gate) | D17 | — | **done** | 2026-07-31 | Commit `69085f4`. Verified by re-measurement (design §7): **example corpus 21→55 passing**, the remaining 10 are `JV2-MTH-003` for methods V2 genuinely lacks (DNN/RLTPMCMC), not schema gaps. `Calculators.path` ×35 / `deps_source` / `Operas.Modules[].selection` now declared — schema no longer contradicts `worker_config.py`'s "Tolerate it". `x-jarvis-zone` in 25 files; dynesty `run_nested`/`sampler` pass through; **Portal is now the I/O authority** (simulated Portal adding `HepMC` → accepted; unknown-to-Portal → `JV2-SCH-002` listing Portal's real formats). AdaptiveBridson typo → error + "Did you mean 'initial_radius'?". Numeric union: `1e-5`/`1.0e-5`/`'0.08'` pass, `'abc'`/`'many'` reject. `run` and `validate` both exit **2** with **zero side effects** (no `outputs/`, no `logs/`). `tests/test_task_schema_corpus.py` added; 19 schema tests green. |
| D17.6 | Strict-validation polish: plural typos, actionable numeric errors, hermetic corpus gate, summary-table hardening | D17 | D17.1–D17.5 | **done** | 2026-07-31 | Commit `6e308a4`. The CJK-width work was deliberately superseded by D17.7's ASCII-escaped diagnostics. Plural `additionalProperties` messages now extract every key and run a separate “did you mean?” match; numeric unions report `expected a number (e.g. 0.05 or 1.0e-5)` instead of generic `anyOf`; the Examples corpus tests skip cleanly when the sibling checkout is absent; the summary helper accepts an empty list; and long Allowed-keys hints show the first 8 plus a remainder count. |
| D17.7 | ASCII-only task cards (`JV2-ENC-001`) | D17 | — | **done** | 2026-07-31 | Commit `6e308a4`. Before schema validation, all parsed YAML keys and string values are checked for every non-ASCII code point. One error per offending location reports path and one-based positions; the hint directs Chinese explanation into a fully supported `#` comment. Encoding failures exit 2 before runtime bootstrap and therefore create no `outputs/`. Diagnostic fields escape non-ASCII as `\\uXXXX`; human summaries use ASCII borders/text, preventing terminal-width drift. Regression coverage: Chinese key, `Scan.name`, description, Cyrillic homoglyph, Chinese comment, zero-side-effect `run`, 65 Examples + 7 templates + 6 parity YAML encoding corpus. Focused validation suite: 128 passed. Full suite was started but a test-owned `Jarvis2:scan` child stalled; it was terminated without touching user runtime processes. |
| D17.8 | Encoding-diagnostic UX: raw user card + one-pass schema report | D17 | D17.7 | **done** | 2026-07-31 | Commit `0e7ab4d`. `load_task_yaml` preserves the original parsed card before V1/default normalisation and runtime metadata injection. `JV2-ENC-001` therefore reports only user-authored paths (a Chinese `Scan.name` no longer produces derived `scan_name` / `task_result_dir` errors). Encoding no longer short-circuits validation: only duplicate `additionalProperties` entries for exact non-ASCII keys are suppressed; ordinary typos in the same block still receive `did you mean`. Docs record both guarantees. Focused validation suite: 129 passed; compile and whitespace checks pass. |
| D17.9 | Close the task-card root zone | D17 | — | **done** | 2026-08-01 | The root is now `closed` and declares only `Scan`, `Sampling`, `Calculators`, `Operas`, `EnvReqs`, and `LibDeps`. `Calculater`, `Opera`, `EnvReq`, and `Scam` fail before outputs are made, with a did-you-mean hint. Removed V1 root blocks give exact migration guidance: Likelihood → `Sampling.LogLikelihood`; `Mapper`, `project_name`, and `Utils` are rejected (`Utils` tells users to remove it). `LibDeps` is its own closed schema. Focused schema/corpus suite green. |
| D18.1 | LibDeps token completeness | D18 | — | **done** | 2026-08-01 | `${LibDeps:path}`, `${LibDeps:make_paraller}`, and `@{ROOT path}` resolve in the control parser and survive its picklable worker form. ROOT is read from preflight; an absent configuration fails naming `EnvReqs.CERN_ROOT`. Optional `CERN_ROOT.path` is also made available without a version check. |
| D18.2 | Declare `LibDeps` in the task-card schema | D18 | — | **done** | 2026-08-01 | `core/libdeps.json` declares `path`, `make_paraller`, closed Modules/installations, and registered executables; V1 `installed` is accepted and intentionally ignored. This is the schema prerequisite used to close D17.9's root. |
| D18.3 | LibDeps install engine | D18 | D18.1, D18.2 | **done** | 2026-08-01 | `LibraryInstaller` runs after environment preflight and before Redis/Workers. It preserves V1's sequential standalone `cd` handling for plain-string commands, resolves dependency layers, caps each layer at `make_paraller`, writes `logs/<scan>/library-<name>.log`, and throws `LibraryInstallError` before Redis on failure. |
| D18.4 | LibDeps install control + skip CLI | D18 | D18.3 | **done** | 2026-08-01 | Reuses D13.11's `jarvis_install.json` / `.jarvis_install_stamp.json` filenames and schema. Fingerprints provide reuse; `reinstall: true` advances the global epoch and rebuilds non-interactively. `run`/`check --skip-library-installation` warn and require existing module paths. |
| D18.5 | LibDeps docs + skill | D18 | D18.3 | **done** | 2026-08-01 | Updated YAML_REFERENCE and component references; added `skills/shared-libraries.md` and its validated fixture `tests/fixtures/shared_libraries_skill.yaml`. The skill documents first-build cost and the deliberate source-tree-change limitation. |

---

## Archived Work Packages (WP-D0.1 … WP-D7.1)

### WP-D0.1 — New `Sample` model

- **Goal**: a `Sample` that round-trips through Redis and rebuilds itself inside a Worker,
  without losing the M1 lazy-materialization contract.
- **Design refs**: §3; `V2/worker_design.md` §3.1, §5; `V2/Jarvis-HEP_v2_架构升级_新增与重构计划.md` §3.1.
- **Files**: `jarvishep/sample.py` (extend, do not rewrite — `materialize`, `create_info`,
  `logger_name` already exist), `jarvishep/base.py` (path tokens), `jarvishep/Sampling/sampler.py`
  (where samples are built), NEW `tests/test_sample_taskdict.py`.
- **Steps**:
  1. Add `u_coords` (alias `u_space`) as the one heavy field; add `execution_plan`/`opera_params`
     carriers on `info`.
  2. `to_task_dict()` → light JSON-able dict (`uuid`, `u_coords`, `execution_plan`,
     `opera_params`, `sample_artifacts`); **exclude** logger and live handles.
  3. `from_task_dict(d)` → reconstruct; `to_info_dict()` → result/monitor projection.
  4. Keep `@SampleID/@Sdir/@PackID` resolution working via the existing `info` adapter.
  5. While here (invariant #14): confirm the `sample.py:140,142` status bug is fixed (it was
     in M1) and that `cunsom_constants`→`custom_constants` (`sample.py:130`) is noted.
- **Accept**: `to_task_dict → from_task_dict` is lossless for the light fields; a lazy
  opera-only Sample still leaves `SAMPLE/` empty on success; existing `tests/test_lazy_materialization.py`
  + `tests/test_failure_replay_log.py` green; full suite green.
- **Rollback**: new methods are additive; thread mode ignores them.
- **Out of scope**: the Worker, Redis, the `UMapper`.

### WP-D0.2 — Redis schema + queue wrapper + serialization

- **Goal**: a thin, tested Redis access layer + the `[distributed]` dependency, with **no**
  runtime wired to it yet.
- **Design refs**: §7; `V2/factory_design.md` §4; `V2/worker_design.md` §8.
- **Files**: NEW `jarvishep/redis_queue.py`, `jarvishep/runtime_config.py` (add `redis` to
  `_VALID_RUNTIME_MODES`, parse a `Runtime.redis` block), `pyproject.toml` (optional extra),
  NEW `tests/test_redis_queue.py` (use `fakeredis` or skip when no server).
- **Steps**:
  1. Define keys (§7) as constants; `push_task/pull_task` (`rpush`/`blpop`), `calc:free`
     acquire/release, results, and `incr_op_count(kind)`.
  2. Serialization helpers (JSON now; leave a `msgpack` hook) supporting numpy/UUID.
  3. `pyproject.toml`: `[project.optional-dependencies] distributed = ["redis>=5.0",
     "msgpack", "aiofiles>=23.0"]`.
  4. Tests behind `fakeredis` (preferred) so CI needs no server; mark a real-Redis
     integration test `skipif` no `REDIS_URL`.
- **Accept**: queue round-trip + op_count tests green under `fakeredis`; `mode: redis` parses
  and stores config but changes no behavior yet; thread mode untouched.
- **Rollback**: unused module; `mode` still defaults to `auto`/`thread`.
- **Out of scope**: Worker, Factory wiring.

### WP-D0.3 — std-`logging` two-layer module

- **Goal**: replace loguru with a std-`logging` wrapper providing top-level + per-Sample
  layers, reusing the existing `SampleLogger`/`BufferedSampleLogger`.
- **Design refs**: §9; `V2/jarvis_hep_logging_design.md`.
- **Files**: `jarvishep/sample_logger.py`, `jarvishep/log_kv.py`, NEW `jarvishep/jlogging.py`
  (top-level setup + adapter), `jarvishep/core.py` (logger init), and a `grep -rn loguru
  jarvishep/` sweep, NEW `tests/test_logging_layers.py`.
- **Steps**:
  1. Inventory loguru usage (`grep -rn "loguru\|logger.bind\|\.success(" jarvishep/`).
  2. `setup_jarvis_logging()` + `get_jarvis_logger(name).bind(**ctx)` over std `logging`
     (+ optional `colorlog`, `QueueHandler` for non-blocking files).
  3. Keep `SampleLogger` (file) + `BufferedSampleLogger` (buffer) as the per-Sample sink;
     the new module wraps, does not replace, the M1 buffer/replay logic.
  4. Migrate call sites incrementally; preserve `docs/specs/LOGGING_CONTRACT.md` format.
- **Accept**: top-level + per-Sample logs produced with the contract format; failure-replay
  test still passes; no `loguru` import remains on the hot path; full suite green.
- **Rollback**: keep loguru import shim for one release if a call site is missed.
- **Out of scope**: Redis central sink, JSON-lines (future).

---

### WP-D0.4 — D0 review fixes + test backfill (gate D0)

- **Goal**: close the **2026-06-27 code review** of the `jarvis2` D0 skeleton — the two RedisQueue
  hard bugs, the unfinished `Sample` methods, and the logging non-blocking/contract details — and
  backfill the D0 unit tests, so D1 starts on a hardened, tested foundation.
- **Origin**: maintainer code review of the D0 commit (Sample + RedisQueue + two-layer logging).
- **Design refs**: components [`redis_queue.md`](components/redis_queue.md),
  [`sample.md`](components/sample.md), [`logger.md`](components/logger.md); DESIGN §3, §7, §9.
- **Files**: `jarvishep2/redis_queue.py`, `jarvishep2/sample.py`, `jarvishep2/logging/toplevel.py`,
  `jarvishep2/logging/sample.py`, `jarvishep2/sample_logger.py` (reused), NEW
  `tests/test_redis_queue.py`, NEW `tests/test_sample_taskdict.py`, NEW `tests/test_logging_layers.py`.
- **Steps**:

  **A. RedisQueue — fix first (these affect Worker + Factory correctness):**
  1. **Namespace bug (highest).** `_calc_free_key(name)` must return `calc:free:{name}` (the
     `CALC_FREE` constant), **not** `{name}:free`. Audit *every* key against `redis_queue.md` §2 and
     pin the exact strings in a test — a wrong free-pool key corrupts `CALC_STATUS` and breaks the
     monitor/external tools.
  2. **Race in `_adjust_calc_counts`.** The `HGETALL CALC_STATUS` read happens **outside** the
     pipeline → cross-Worker count drift. Refactor to atomic **`HINCRBY`** deltas (acquire: free −1 /
     busy +1; release: inverse) inside one pipeline, or `WATCH` + transaction-retry. Free/busy must
     stay consistent under concurrent acquire/release.
  3. `heartbeat`: drop the blanket `str()` coercion (numbers become strings) — keep numeric types or
     JSON-encode the value; `snapshot_raw`/monitor decode accordingly.
  4. Minimal **schema validation** on `push_task`/`submit_result` (required keys: `uuid`, plus
     `u_coords`/`execution_plan`) — fail fast, never enqueue a malformed task.
  5. `release_calc(name, pack_id)`: **use** `pack_id` (assert it matches the busy owner, for
     `pack_id` traceability) or drop the param — do not accept-and-ignore it.

  **B. Sample — complete the `sample.md` §3 surface:**
  6. Finish `materialize`, and add `resolve_token`, `start`/`close`, `record`, `to_info_dict`.
  7. Add a `bind_params(mapper: UMapper)` **placeholder** (interface reserved now; UMapper lands
     later) so the Worker call site is stable.
  8. Validate `ExecutionStep.type` ∈ {`calculator`, `opera`, `likelihood`, `nuisance_optimize`}
     (reject the empty string).

  **C. Logging — align with `logger.md`:**
  9. `setup_jarvis_logging`: wire `QueueHandler` + a `QueueListener` for **non-blocking** file sinks
     (the high-load requirement); stop + drain the listener at shutdown (no lost tail lines).
  10. Context binding emits `key=value` per `LOGGING_CONTRACT.md` (worker_id, sample_uuid, phase…);
      align `bind`/formatting with `logger.md` §3.
  11. Confirm per-Sample child loggers bind via `logger_name` **without** forcing materialization
      (M1 lazy contract).

  **D. Tests — backfill (fakeredis, no server):**
  12. `tests/test_redis_queue.py`: exact key-string assertions (catches the namespace bug),
      `op_count` increments, calc-pool cap (≤ n concurrent acquire; race-free free/busy under two
      "Workers"), pipeline atomicity, codec round-trip (numpy/UUID).
  13. `tests/test_sample_taskdict.py`: `to_task_dict → from_task_dict` round-trip, no-logger-on-wire,
      lazy materialization (opera-only success leaves `SAMPLE/` empty), failure replay.
  14. `tests/test_logging_layers.py`: two layers produced, contract format, failure-replay parity,
      no `loguru` on the hot path.
- **Accept**: both RedisQueue hard bugs fixed — key strings **exactly** match `redis_queue.md` §2,
  and free/busy stays consistent under concurrent acquire/release in a 2-Worker `fakeredis` test;
  `Sample` §3 methods present with round-trip + lazy tests green; logging non-blocking + `key=value`
  contract + failure-replay green; `pytest tests/test_redis_queue.py tests/test_sample_taskdict.py
  tests/test_logging_layers.py` all pass under `fakeredis`; full suite green.
- **Rollback**: revert this WP commit (a fix/hardening pass over D0; no new feature flag).
- **Out of scope**: `UMapper` implementation, TaskFactory/Worker (D1.1), `msgpack` codec (keep JSON;
  note as future).

---

### WP-D0.5 — D0 wrap-up: defensive hardening + spawn-pickling + integration test + polish

- **Goal**: finalize D0 before it is declared closed — strengthen payload validation, **prove
  spawn-safety**, add a Sample↔RedisQueue integration test, and bring the modules to 100% alignment
  with the component-doc tables. **Non-blocking for D1.1** (D1.1 may start after D0.4), but D0 is not
  closed until this is green.
- **Origin**: 2026-06-27 follow-up review — the post-D0.4 "polishing + defensive + test-completion"
  items (core functionality already passed; these de-risk D1).
- **Design refs**: components [`redis_queue.md`](components/redis_queue.md),
  [`sample.md`](components/sample.md), [`logger.md`](components/logger.md); invariants #7 (only light
  dicts in Redis), #8 (no logger across a process boundary), #10 (`spawn`).
- **Files**: `jarvishep2/redis_queue.py`, `jarvishep2/sample.py`, `jarvishep2/logging/toplevel.py`,
  NEW `tests/test_spawn_pickling.py`, NEW `tests/test_d0_integration.py`, extend
  `tests/test_redis_queue.py`.
- **Steps**:
  1. **Payload validation (extends D0.4 A.4).** Replace the minimal key-check in
     `push_task`/`submit_result` with a lightweight **validator** (a small dataclass / `TypedDict` /
     explicit field+type check): a task requires `uuid: str` + (`u_coords` | `execution_plan`); a
     result requires `uuid` + `status`. **Raise** on a malformed payload *before* it enters the queue
     — a bad task must never reach a Worker. Keep pipeline atomicity. Add failing-payload cases to
     `tests/test_redis_queue.py`.
  2. **Sample method alignment.** Finish `bind_params(self, mapper)` as a **clear stub** with a stable
     signature + docstring (UMapper not implemented yet, but the Worker call site must not move); bring
     `to_info_dict()` and `record()` to **100% agreement** with `sample.md` §3 (exact field set; no
     logger/handles).
  3. **Spawn-pickling tests** (invariant #10). NEW `tests/test_spawn_pickling.py`: round-trip
     `Sample` + `ExecutionStep` + a built `RedisQueue` config through
     `multiprocessing.get_context("spawn")` pickling, and a real child process that unpickles a
     `Sample` and rebuilds it; **assert no live handle/logger survives** (invariant #8). This is the
     direct de-risk for D1 Worker spawn.
  4. **D0 integration test.** NEW `tests/test_d0_integration.py`: the full path
     `Sample → to_task_dict → RedisQueue.push_task → pull_task → from_task_dict → materialize`, both
     the **success** path (opera-only ⇒ `SAMPLE/` stays empty) and the **failure** path (failure
     replay leaves a readable log). `fakeredis`, no server.
  5. **Polish.** Complete type hints + docstrings on all new classes; curate per-module `__all__`
     exports; reconcile each module against its component-doc member-function table (100% alignment).
     Optional: a small `bind()` hot-path micro-opt behind the existing `isEnabledFor` guard.
- **Accept**: malformed task/result payloads are rejected (tested); `bind_params`/`to_info_dict`/
  `record` match `sample.md` §3; `tests/test_spawn_pickling.py` green (Sample survives `spawn`, no
  logger/handle leak); `tests/test_d0_integration.py` green (round-trip + failure replay); type
  hints/docstrings/`__all__` complete; full suite green under `fakeredis`. **D0 is declared closed
  when D0.4 and D0.5 are both green.**
- **Rollback**: revert this WP commit (defensive + test hardening; no feature flag).
- **Out of scope**: `UMapper` implementation (D-later), TaskFactory/Worker (D1.1), `msgpack` codec
  (keep JSON).

---

### WP-D1.1 — TaskFactory skeleton + Worker process (opera-only MVP)

- **Goal / scope**: get the **single-Worker opera-only path** running end to end under
  `mode: redis`. The bar is: **one** Worker pulls Samples from Redis, runs only the opera (and
  likelihood) steps of the `execution_plan`, and submits results, with a Factory that can spawn
  it, observe it, and shut it down cleanly. Calculators, multi-Worker concurrency, and the
  Archiver are explicitly **not** in this WP.
- **Design refs**: §2, §4, §6; `V2/factory_design.md`; `V2/worker_design.md`; component docs
  [`components/factory.md`](components/factory.md), [`components/worker.md`](components/worker.md),
  [`components/core.md`](components/core.md).
- **Files** (package is **`jarvishep2/`** on the `jarvis2` branch; the old `jarvishep/` paths are
  the V1 line): `jarvishep2/factory.py` (`TaskFactory`), NEW `jarvishep2/worker.py`
  (`Worker(Process)`), `jarvishep2/core.py` (`Jarvis2Core.init_factory` mode switch +
  spawn/teardown), `jarvishep2/Sampling/sampler.py` (submit `task_dict` to Redis when
  `mode: redis`), NEW `tests/test_worker_mvp.py`.
- **Steps**:
  1. `TaskFactory.start_workers(n=1)` spawns `Worker(Process)` (spawn ctx; only `redis_config` +
     picklable `worker_config` cross the boundary — see `factory.md` §3a); Workers `blpop`
     `hep:task_queue`.
  2. Worker main loop: `from_task_dict → bind_params → materialize → run opera/likelihood layers
     → to_info_dict → submit_result` (`submit_result` bumps `hep:sample:op_count` once).
  3. Sampler: when `mode: redis`, `push_task(sample.to_task_dict())` instead of the thread-pool
     submit.
  4. Result collection: control process drains results → HDF5 writer (the D1.1 `SimpleArchiver`
     writes directly; the batched Archiver lands in D4).
  5. Boot precondition: `mode == redis` brings the Factory up; otherwise `init_factory` is a
     no-op. V2 carries **no thread fallback** (invariant #5) — an unpicklable config / unimportable
     user func is a hard, early failure, not a silent downgrade.
- **Accept** (each line independently verifiable under `fakeredis` + real spawn):
  1. **Spawn** — `TaskFactory.init_redis()` then `start_workers(1)` produces one live `Worker`
     process (`is_alive()` true; refuses a second `start_workers` while it is alive).
  2. **Pull + execute** — a pushed opera-only `task_dict` is pulled and run through the
     `from_task_dict → materialize → opera/likelihood → submit_result` pipeline; the result
     appears on the results path and `op_count("sample")` advances.
  3. **Opera-only parity** — an opera-only reference scan (`Jarvis2`, 1 Worker) produces DATABASE
     records **set-equal** to the captured V1 golden output.
  4. **Graceful shutdown** — `SIGINT`/`SIGTERM` (or `TaskFactory.shutdown()`) lets the in-flight
     Sample finish, joins the Worker (no orphan), and closes Redis.
  5. **Monitor snapshot** — `get_monitor_snapshot()` returns a dict with at least worker
     rows/counts and Redis queue/stats fields (`workers`, `workers_alive`, `task_queue_length`,
     `sample_stats`, `op_counts`); `op_count`-gated incremental fetch with ~120 Hz updater
     (idle ticks avoid `HGETALL`; read path is pure memory).
  6. Full suite green under `fakeredis`.
- **Rollback**: revert the WP commit (V2 has no `mode: thread`; see §5 rollback semantics).
- **Out of scope**: multi-worker + free-pool (D2.1), Archiver batching (D4),
  watchdog/respawn (D6.1), `--monitor` dashboard UI (D5.2).
- **Notes / deviations from the original design** (code is ground truth):
  - `start_workers` receives the live control-process `RedisQueue`, but only its picklable
    connection settings (`redis_config`) are kept for spawn (`factory.md` §3a).
  - `op_count`-gated snapshot landed in D1.1 (pulled forward from D5.1); D5.2 still owns
    `--monitor` dashboard + `get_run_metrics`.
  - No watchdog/respawn yet (D6.1).

### WP-D1.2 — Calculator in Worker + single-Worker parity

- **Goal**: a Worker holds and runs a `CalculatorModule`; single-Worker calculator parity.
- **Status**: **done** on `jarvis2` (2026-06-28).
- **Design refs**: §4; `V2/worker_design.md` §3.2, §4.1, §10.
- **Files**: `jarvishep/Module/calculator.py` (add `preload_templates()`,
  `execute(sample_info)` convenience over existing `run_command`), `jarvishep/worker.py`,
  `jarvishep/moduleManager.py` (per-Sample `execute_workflow` reused inside the Worker),
  NEW `tests/test_worker_calculator.py`, reuse `tests/parity_project`.
- **Steps**:
  1. `_init_calculators()` builds one instance per calculator type at Worker startup and
     `preload_templates()`.
  2. Worker executes the calculator layer of `execution_plan` via the held instance,
     resolving `@Sdir`/`@PackID` per Sample.
  3. Fix in passing if touched (invariant #14): `calculator.py:136` `custom_format` missing
     `self`.
  4. Parity: run `tests/parity_project` (`--check-modules` scale) under `mode: redis` (1
     Worker) vs the pinned thread-mode expectation.
- **Accept**: calculator DATABASE + SAMPLE trees match the captured V1 golden output for 1
  Worker; calculator tests green; full suite green.
- **Rollback**: `mode: thread`.
- **Out of scope**: cross-Worker free-pool, layer concurrency (D2).

---

### WP-D2.1 — Multi-Worker pool + held calculators + Redis free-pool

- **Status**: **done** on `jarvis2` (2026-06-28).
- **Goal**: N Workers with globally capped calculator concurrency and preserved `pack_id`.
- **Design refs**: §4, §7; Blueprint §3–§5.
- **Files**: `jarvishep/factory.py` (`start_workers(n)`, lifecycle), `jarvishep/worker.py`
  (`calc:free:<name>` acquire/release), `jarvishep/modulePool.py` (slot semantics →
  Redis pool), NEW `tests/test_worker_pool.py`.
- **Steps**:
  1. `start_workers(n)` from `Runtime.workers`; each Worker registers to `hep:worker:status:*`.
  2. Calculator concurrency: acquire a slot from `calc:free:<name>` (`blpop`) before running,
     `rpush` after — caps a heavy calculator across *all* Workers (the Redis form of the
     Blueprint semaphore). Generate a fresh `pack_id` per acquire.
  3. Update `hep:calculator:status` free/busy on acquire/release.
- **Accept**: 2-Worker vs 1-Worker DATABASE fingerprint parity (set-equal UUIDs, identical
  records); a heavy calculator never exceeds its configured `max_concurrent` across Workers
  (assert via status); throughput rises 1→2 Workers.
- **Rollback**: `mode: thread`.
- **Out of scope**: layer-internal concurrency (D2.2), clone_shadow (D2.3).

### WP-D2.2 — Layer-internal calculator concurrency

- **Status**: **done** on `jarvis2` (2026-06-28).
- **Goal**: same-layer calculators of one Sample run concurrently via a **per-Worker**
  `AsyncSubprocessScheduler` (invariant #6 — parallelism inside a Sample, not across).
- **Design refs**: §4; Discussion §7; reuse `jarvishep/async_subprocess.py`.
- **Files**: `jarvishep/worker.py`, `jarvishep/async_subprocess.py` (one instance per Worker),
  `jarvishep/workflow.py` (layer derivation feeding `execution_plan`), NEW
  `tests/test_layer_concurrency.py`.
- **Steps**:
  1. Each Worker owns one `AsyncSubprocessScheduler`; the Worker runs a Sample's DAG layer
     by layer, dispatching all calculators in a layer concurrently, awaiting the layer.
  2. Layer derivation: auto from `required_modules` (open question §13.1 — explicit override
     deferred).
- **Accept**: a 2-calculator independent layer completes in ≈max(t1,t2) not t1+t2 (timed
  test); output parity with serial execution; full suite green.
- **Rollback**: `mode: thread`, or a config flag forcing serial layers.
- **Out of scope**: cross-Sample concurrency.

### WP-D2.3 — `clone_shadow` isolation + LibDeps symlink path

- **Status**: **done** on `jarvis2` (2026-06-29).
- **Goal**: physical per-Sample isolation for non-thread-safe tools; symlink path for safe
  tools.
- **Design refs**: §4 (isolation); Discussion §4.
- **Files**: `jarvishep/Module/calculator.py`, `jarvishep/library.py` (LibDeps), `jarvishep/worker.py`,
  config schema, NEW `tests/test_clone_shadow.py`.
- **Steps**:
  1. Honor `clone_shadow: true|false` per calculator: `true` → per-Sample physical copy;
     `false` → symlink/`registered_executables` into the Sample dir.
  2. Optional `isolation: full_shadow|per_sample_dir` left as a future field (do not add now).
- **Accept**: a `clone_shadow` calculator gets an isolated dir per Sample (no cross-talk under
  concurrency); a safe tool runs via symlink with no copy; parity preserved.
- **Rollback**: default `clone_shadow: true` reproduces v1 isolation.
- **Out of scope**: `registered_executables` syntax (D3.1).

---

### WP-D3.1 — `registered_executables` + two-phase `CommandParser`

- **Status**: **done** on `jarvis2` (2026-06-29).
- **Goal**: register tools once; split static vs per-Sample token resolution.
- **Design refs**: §8; Discussion §5.
- **Files**: NEW `jarvishep/command_parser.py`, `jarvishep/config.py` (parse
  `LibDeps.registered_executables`), `jarvishep/base.py` (token resolution), `jarvishep/worker.py`,
  NEW `tests/test_command_parser.py`.
- **Steps**:
  1. Phase 1 (post-YAML): resolve `registered_executables`, LibDeps paths, all static tokens
     once.
  2. Phase 2 (Worker, pre-exec): resolve only `@SampleID/@Sdir/@PackID`.
  3. `resolution: direct_path` (default) | `symlink`.
- **Accept**: a command using a registered executable resolves correctly in both phases;
  Phase-1 output has no per-Sample tokens; parity preserved.
- **Rollback**: feature is opt-in (absence ⇒ v1 `ln -sf` path still works).
- **Out of scope**: `env_setup`.

### WP-D3.2 — `env_setup` capture-from-source + cache

- **Status**: **done** on `jarvis2` (2026-06-29).
- **Goal**: activate `source`-based environments (Rivet, …) inside isolated subprocesses.
- **Design refs**: §8; Discussion §6.
- **Files**: `jarvishep/Module/calculator.py`, NEW `jarvishep/env_setup.py`
  (`capture_env_from_source` + cache), `jarvishep/async_subprocess.py` (`SubprocessJob env=`),
  NEW `tests/test_env_setup.py`.
- **Steps**:
  1. `source xxx.sh && env` → parse to dict; merge with `os.environ`; pass via `SubprocessJob(env=…)`.
  2. **Cache** per source script per Worker (run once, not per Sample).
- **Accept**: a Calculator with `env_setup` sees the sourced variables; the script runs once
  per Worker (assert via a counter); parity preserved.
- **Rollback**: absence of `env_setup` ⇒ no change.
- **Out of scope**: general shell backend.

### WP-D3.3 — `Runtime.FileOperation.delete_method`

- **Goal**: configurable delete (`shutil` default, `rm` for mass-delete).
- **Design refs**: §5; `V2/command_execution_design.md`.
- **Files**: NEW `jarvishep/file_ops.py` (`delete_path(path, method)`), call sites in
  Worker cleanup + Archiver, `jarvishep/runtime_config.py`, NEW `tests/test_file_ops.py`.
- **Accept**: both methods delete files/dirs; bad method raises; default is `shutil`.
- **Rollback**: default `shutil` is v1-equivalent.
- **Out of scope**: rsync/tar backends.

---

### WP-D4.1 — Worker staging mv + Archiver process skeleton

- **Status**: **done** on `jarvis2` (2026-06-29).
- **Goal**: Workers hand finished Samples to a separate Archiver via a fast staging move.
- **Design refs**: §5; Discussion §2, §3.
- **Files**: NEW `jarvishep/archiver.py` (`Archiver(Process)`), `jarvishep/worker.py`
  (mv work_dir → staging, handoff), `jarvishep/core.py` (Archiver lifecycle), NEW
  `tests/test_archiver_handoff.py`.
- **Steps**:
  1. Worker: after result, `mv work_dir → staging/<uuid>` (metadata-only), push
     `(info_dict, product_list, staging_path)` to an archive queue (Redis list or
     control-side queue).
  2. Archiver process consumes the queue (skeleton: move staging → SAMPLE/, no batching yet).
- **Accept**: products land in `SAMPLE/<uuid>/` identical to thread mode; staging emptied;
  clean shutdown drains the queue.
- **Rollback**: `mode: thread` (synchronous archive).
- **Out of scope**: batching, format conversion, async I/O (D4.2).

### WP-D4.2 — Archiver batch persistence + output parity gate

- **Status**: **done** on `jarvis2` (2026-06-29). Format I/O (SLHA/JSON/xSLHA/portal/CSV) stays in
  **Jarvis-Portal** as the V1 backend; Jarvis-HEP Archiver owns transport only (staging handoff,
  batch move, DATABASE append).
- **Goal**: batched, NAS-optimized final persistence; lock output parity.
- **Design refs**: §5, §11.
- **Files**: `jarvishep/archiver.py` (batch ≈200, async/move, xlha/slha, delete),
  `jarvishep/hdf5writer.py` (Archiver-fed), `jarvishep/observable_io.py`, `jarvishep/dataconvert.py`,
  NEW `tests/test_archiver_parity.py`.
- **Steps**:
  1. Batch `Archiver.batch_size` (default 200) finished Samples; async write + `move` into
     SAMPLE/ + DATABASE/ (HDF5/CSV); xlha/slha formatting; delete staging
     (`delete_method`).
  2. Use `os.rename`/`shutil.move` (same-volume NAS) by default; `use_shell`/`rm` optional.
- **Accept**: DATABASE/SAMPLE/CSV **byte-identical** to the captured V1 golden output on a
  fixed scan (`--convert` parity); batch boundaries do not drop/duplicate Samples.
- **Rollback**: `mode: thread` / `Archiver.batch_size: 1`.
- **Out of scope**: monitor, checkpoint.

---

### WP-D5.1 — `op_count` writers + TaskFactory background snapshot

- **Goal**: read-only 60 Hz monitor snapshot driven by `op_count` change detection.
- **Design refs**: §6, §7; `V2/factory_design.md` §5, §6, §8.
- **Files**: `jarvishep/worker.py` + `jarvishep/Sampling/sampler.py` (`incr` on state change),
  `jarvishep/factory.py` (`_start_snapshot_updater`, `_collect_latest_status_from_redis`,
  `get_monitor_snapshot`), NEW `tests/test_monitor_snapshot.py`.
- **Steps**:
  1. Writers `incr hep:{worker,calculator,sample,task}:op_count` on each meaningful change.
  2. Background thread (~100–200 Hz): per subsystem, fetch only when `current > last_seen`;
     always refresh `task_queue_length`.
  3. `get_monitor_snapshot()` = in-memory `deepcopy` (<0.5 ms).
- **Accept**: snapshot reflects advancing counters; idle subsystems cost one `GET`/tick
  (assert call counts via `fakeredis`); 60 Hz calls do not perturb throughput.
- **Rollback**: monitor is observational; off ⇒ no behavior change.
- **Out of scope**: the dashboard UI.

### WP-D5.2 — `--monitor` dashboard + run_summary from Redis

- **Goal**: `--monitor` renders the snapshot; run_summary computed from Redis counters.
- **Design refs**: §6; `docs/specs/RUN_SUMMARY_METRICS.md` (frozen contract).
- **Files**: NEW `jarvishep/dashboard.py` (or reuse `jarvishep/monitor.py`), `jarvishep/client.py`
  (`--monitor --pid`), `jarvishep/monitoring/run_summary.py`, NEW `tests/test_dashboard_reader.py`.
- **Steps**:
  1. Dashboard reads `get_monitor_snapshot()` (or attaches to Redis directly — independent
     process, Blueprint §6) at 10 FPS.
  2. `run_summary` fields projected from `hep:sample:stats` + factory timing; **field-for-field
     equal** to the frozen schema.
- **Accept**: reader unit tests green; `run_summary.json` schema-identical pre/post; `--monitor`
  with no running scan exits with a clear message.
- **Rollback**: legacy curses monitor remains until cutover.
- **Out of scope**: Prometheus/Web UI (future).

---

### WP-D6.1 — Heartbeats + dead-Worker respawn + in-flight re-queue

- **Goal**: a crashed Worker cannot stall or corrupt a scan.
- **Design refs**: §6, §10; Blueprint §9; `V2/worker_design.md` §9.
- **Files**: `jarvishep/factory.py` (watchdog), `jarvishep/worker.py` (heartbeat writes),
  `jarvishep/redis_queue.py`, NEW `tests/test_worker_failure.py`.
- **Steps**:
  1. Worker writes `last_heartbeat` to `hep:worker:status:<id>`; Factory watchdog detects
     staleness.
  2. Dead Worker: release its held `calc:free` slots, re-queue its in-flight Sample
     (bounded retries), respawn a fresh Worker.
- **Accept**: SIGKILL a Worker mid-scan → scan completes, no missing/duplicate UUIDs; held
  slots recovered.
- **Rollback**: `mode: thread`.
- **Out of scope**: RNG/checkpoint (D6.2).

### WP-D6.2 — RNG spawning + distributed checkpoint/resume

- **Goal**: reproducibility across Worker counts + resume under the Redis model.
- **Design refs**: §10; `DESIGN_CHECKPOINT_1.7.0.md`; `docs/design/STATE_SAVER_RESUME_DESIGN.md`.
- **Files**: `jarvishep/Sampling/Source/MCMC/runtime_checkpoint.py`, `jarvishep/core.py`
  (`_preload_resume_checkpoint`, drain Redis on resume), `jarvishep/Sampling/sampler.py`,
  NEW `tests/test_distributed_resume.py`.
- **Steps**:
  1. `SeedSequence` master in the sampler checkpoint; per-Sample/per-Worker child streams
     (decouple reproducibility from Worker count).
  2. Resume: rebuild Redis pools from config; **drain** stale `hep:task_queue`; never restore
     in-flight; Sampler re-proposes.
  3. Keep checkpoint UX + payload minimalism; `state.pkl` location/heartbeat unchanged.
- **Accept**: kill-and-resume mid-scan → no duplicate/missing UUIDs; seeded rerun reproduces
  sampler trajectory independent of Worker count; v1 checkpoint refusal message intact.
- **Rollback**: `mode: thread` uses the existing checkpoint path unchanged.
- **Out of scope**: Redis RDB/AOF durability (open question §13.5).

---

### WP-D7.1 — Slow-regime acceptance gates

- **Goal**: lock the architecture against the metrics that matter for real workloads.
- **Design refs**: §1, §12; `docs/specs/RUN_SUMMARY_METRICS.md`.
- **Files**: NEW `tests/test_distributed_acceptance.py`, NEW calculator fixtures under
  `tests/benchmark_project/bin/`, `docs/benchmarks/` report.
- **Gates**:
  1. **Worker scaling** (slow regime): calculator throughput scales ~linearly to physical
     cores, and is demonstrated stable at high Worker counts (toward ~256 on capable
     hardware; scale the assertion to the test machine).
  2. **Archive latency**: NAS `move` archive sustains the Worker output rate; staging never
     unboundedly grows (assert bounded backlog).
  3. **Chaos**: SIGKILL Workers under load → scan completes, parity preserved.
  4. **Parity**: DATABASE/SAMPLE/CSV identical to the captured V1 golden output on a fixed
     calculator scan.
  5. **Monitor**: `get_monitor_snapshot()` sustains 60 Hz with no throughput regression.
- **Accept**: all gates green; numbers recorded in `docs/benchmarks/` and the ledger.
- **Rollback**: n/a (gate only).

### WP-D13.11 — 2026-07-29 — Code-review fixes

- **Goal**: close review findings §1.1–§1.5 without changing the sampler science surface.
- **Implemented**: control-process-owned `jarvis_install.json` with monotone
  `reinstall_epoch` and per-pack stamp fan-out; unmatched-feedback warnings in the
  sampler base; Lua-atomic calculator release with held-pack retention on errors;
  tmp + `os.replace` CSV exports; `generation_timeout` with legacy `timeout` alias and
  per-generation documentation.
- **Tests**: targeted D13.11 tests passed (39 tests); compileall and `git diff --check`
  passed. Full suite: 608 passed, 8 pre-existing failures in distributed timing,
  layer-concurrency path normalization, plot CLI compatibility, and legacy layer index.
- **Notes**: no source-tree content hashing was added. `jarvis_install.json` is written
  only by the control process; Workers write only their pack stamp. Pack summaries are
  refreshed best-effort during control-process shutdown.
- **Out of scope**: D13.12, D13.13, D8/Agent JSON.

---
