# V1 → V2 Component Coverage Map

A complete sweep of the V1 `jarvishep/` tree mapping **every module** to its V2 disposition, so
no necessary component is lost. Updated as component docs are written.

Legend — **Doc**: has its own V2 component design doc · **Reused**: carried into V2 largely
unchanged (behavior frozen) · **Merged**: folded into another V2 doc · **Dissolved**: its role
moves elsewhere (e.g. into the Worker / Redis) · **Retired**: not in V2 · **Control-local**: stays
in the control process only.

## Top-level runtime

| V1 module | V2 disposition | Where |
|-----------|----------------|-------|
| `client.py` | Doc | [cli.md](cli.md) |
| `core.py` | Doc | [core.md](core.md) |
| `config.py` | Doc | [config_schema.md](config_schema.md) |
| `runtime_config.py` | Doc | [config_schema.md](config_schema.md) |
| `base.py` | Doc | [paths_tokens.md](paths_tokens.md) |
| `sample.py` | Doc | [sample.md](sample.md) |
| `sample_logger.py` | Doc | [logger.md](logger.md) |
| `log_kv.py` | Merged | [logger.md](logger.md) (`format_two_column_log` helper) |
| `workflow.py` | Doc | [workflow.md](workflow.md) |
| `factory.py` | Doc (replaced) | [factory.md](factory.md) — `WorkerFactory` → `TaskFactory` |
| `moduleManager.py` | **Dissolved — no V2 equivalent** | module *config* → picklable blueprint ([config_schema.md](config_schema.md) + [workflow.md](workflow.md)); module *execution* → Worker ([worker.md](worker.md)). No orchestrator class. |
| `modulePool.py` | Dissolved | [module_base.md](module_base.md) — live-object pool → Redis free-pool |
| `async_subprocess.py` | Doc (reused) | [subprocess_scheduler.md](subprocess_scheduler.md) — now per-Worker |
| `io_manager.py` | Merged | [subprocess_scheduler.md](subprocess_scheduler.md) §6 — calculator-path I/O executor |
| `hdf5writer.py` | Doc | [datarecorder.md](datarecorder.md) |
| `observable_io.py` | Merged | [datarecorder.md](datarecorder.md) + [utils.md](utils.md) (CSV schema) |
| `dataconvert.py` | Merged | [utils.md](utils.md) |
| `benchmark.py` | Doc | [benchmark.md](benchmark.md) |
| `monitor.py` | Doc (replaced) | [monitor.md](monitor.md) |
| `monitoring/run_summary.py` | Merged | [monitor.md](monitor.md) §3 |
| `inner_func.py` | Doc | [expression.md](expression.md) |
| `utils.py` | Doc | [utils.md](utils.md) |
| `versioning.py` | Merged | [utils.md](utils.md) |
| `plot.py` | Merged | [utils.md](utils.md) |
| `distributor.py` | Doc | [distributor.md](distributor.md) — sampler dispatch + resume registry |
| `library.py` | Doc | [library.md](library.md) |
| `project_scaffold.py` · `project_packager.py` · `official_project_library.py` | Doc | [project_tools.md](project_tools.md) |

## Module backends (`Module/`)

| V1 module | V2 disposition | Where |
|-----------|----------------|-------|
| `Module/module.py` (`Module` base) | Doc | [module_base.md](module_base.md) |
| `Module/calculator.py` (`CalculatorModule`) | Doc | [calculator.md](calculator.md) |
| `Module/operas.py` (`OperasModule`) | Doc | [operas.md](operas.md) |
| `Module/likelihood.py` (`LogLikelihood`) | Doc | [likelihood.md](likelihood.md) |
| `Module/parameters.py` (`Parameters`) | Doc | [parameters_variables.md](parameters_variables.md) |
| `Module/library.py` | Merged | [library.md](library.md) |
| `Module/nuisance_LogLikelihood.py` · `nuisance_passCondition.py` | Doc | [nuisance.md](nuisance.md) |

## I/O parameter system (`IOs/`)

| V1 module | V2 disposition | Where |
|-----------|----------------|-------|
| `IOs/IOs.py` (`IOfile`, `InputFile`, `OutputFile`) | Doc | [io_system.md](io_system.md) |
| `IOs/Input.py` (SLHA/JSON writers) | Doc | [io_system.md](io_system.md) |
| `IOs/Output.py` (SLHA/JSON/xSLHA/FileOutput readers) | Doc | [io_system.md](io_system.md) |
| `IOs/registry.py` (`IORegistry`, adapters) | Doc | [io_system.md](io_system.md) |
| `IOs/portal.py` (portal in/out) | Doc | [io_system.md](io_system.md) |
| `IOs/parameter.py` | Merged | [io_system.md](io_system.md) |

## Sampling (`Sampling/`)

| V1 module | V2 disposition | Where |
|-----------|----------------|-------|
| `Sampling/sampler.py` (`SamplingVirtial`) | Doc | [sampler.md](sampler.md) |
| `Sampling/variables.py` (`Variable`) | Doc | [parameters_variables.md](parameters_variables.md) + [umapper.md](umapper.md) |
| `Sampling/bucketallocator.py` | Control-local | [paths_tokens.md](paths_tokens.md) §4 (Worker gets resolved `save_dir`) |
| `Sampling/sample_archive.py` | Reused (bug-fixed) | [datarecorder.md](datarecorder.md) — fix race #13 |
| concrete samplers (Random, Grid, Bridson, MCMC family, Dynesty, MultiNest, Diver, …) | Reused | [samplers_catalog.md](samplers_catalog.md) |
| `Sampling/Source/MCMC/*` (state machines, engines, controller, checkpoint) | Reused | [samplers_catalog.md](samplers_catalog.md) + [checkpoint.md](checkpoint.md) |
| `Sampling/nuisance_sampler.py` · `Source/Nuisance/profile1d.py` | Doc | [nuisance.md](nuisance.md) |
| `Sampling/csv_sampler.py` · `rl_sampler_base.py` · `nested_checkpoint_bridge.py` | Reused | [samplers_catalog.md](samplers_catalog.md) |

## Retired (not in V2)

| V1/throughput-core | Why |
|--------------------|-----|
| SoA `StructuredBatch`, SPSC rings, shm coordination region, `JarvisState` shm, vectorized-opera gates | retired throughput-core — archived in the Jarvis-HEP repo (`docs/archive/2026-06_v2_throughput_core/`) |
| `Module/calculator.py:analyze_config_multi` (`pass`), `Source/MCMC/mcmc_chain.py` (dead) | dead code; do not port |
| placeholder engines (`engine_hmc/mala/nuts` `gradient_contract_level=placeholder`) | carry placeholders as-is until real gradient impls land |

## Coverage status

**Complete (2026-06-24 audit).** **32 component design docs** cover runtime core, module/IO
backends, sampling, and the V1-style auxiliary subsystems. A full sweep of `jarvishep/`
(top-level + `Module/` + `IOs/` + `monitoring/` + `Sampling/`) confirms **every necessary module
is mapped** — every *Doc* row resolves to a file in this folder, and every *Reused/Merged/
Dissolved/Retired/Control-local* row names its destination. Vendored trees
(`Sampling/Source/{Dynesty,Diver}/`) and the concrete samplers are covered en masse by
[samplers_catalog.md](samplers_catalog.md). No necessary component remains undocumented.

## As-built disposition (jarvis2 `d0de31a`)

> This map was written against the **design**. After the full build, several rows resolve
> differently — the per-component docs are now as-built. Key deltas (full list in
> [`../archive/reviews/CODE_REVIEW_2.0.md`](../archive/reviews/CODE_REVIEW_2.0.md)):
>
> - **Renamed/relocated:** `mapping.py`→`mapper.py`, `hdf5writer.py`→`database.py`,
>   `expression.py`→`inner_func.py`+`sampling_utils.py`, `config.py`→`runtime_config.py`+
>   `task_config.py`+`worker_config.py`, `IOs/`→`io_json.py`+`file_ops.py`,
>   `Module/operas.py`→`operas.py`, `Module/likelihood.py`→`likelihood.py`. No `Module/module.py`
>   (no shared ABC); no `Base` class (`base.py` is functions).
> - **Not built / stubbed:** `benchmark.py` (no module; only `core.run_check_modules` + acceptance
>   JSON), nuisance modules (only `Sample` stub hooks), `project_*` tools, `utils/versioning/
>   dataconvert/plot/observable_io`, and the **entire MCMC / nested / gradient sampler families** —
>   V2 ships only the four stateless samplers (Bridson/Random/Grid/CSV).
> - **New, undesigned:** `archive_handoff.py`, `calculator_pools.py`, `mp_context.py`,
>   `stateless_batch.py`, `seeded_sampler.py`, `checkpointed_sampler.py`, `monitoring/run_summary.py`,
>   `testing/eggbox.py`.

## V1 delta to carry forward (2026-07-09 sample command logging)

V1 gained a refined sample command logging contract during the HinoLLP/check-modules work. V2
should carry the behavior forward even if the implementation names differ. Canonical design:
[logger.md](logger.md) §3 (API) + §4 (command-block contract) + §3.6 (as-built gap).

| V1 surface | V2 target | Docs |
|------------|-----------|------|
| `sample_logger.py` (`SampleLogger` + `BufferedSampleLogger` + `_CONSOLE_LOCK`) | same ownership: file sink always; optional `console=` mirror of identical rendered text | [logger.md](logger.md) §3 |
| `_format_sample_log_text` shared formatter | one formatter for file and console (no dual formatting paths) | [logger.md](logger.md) §3.2 |
| `Sample.info["sample_console"]` | internal check-modules-only flag; not a normal scan/wire field; passed as `console=` into both logger constructors | [sample.md](sample.md) §4.1 |
| live console on `BufferedSampleLogger` | check-modules stays interactive before materialization; `replay_to` is file-only (no double console) | [logger.md](logger.md) §3.4 |
| calculator `SubprocessJob.meta` command fields | scheduler emits formatted header → raw stdout/stderr → raw command summary **before** job completion | [subprocess_scheduler.md](subprocess_scheduler.md) §4.1 |
| `--check-modules` test samples | mirror sample command logs to terminal through `SampleLogger`; production scans remain sample-file-only | [logger.md](logger.md) §4.2 |
