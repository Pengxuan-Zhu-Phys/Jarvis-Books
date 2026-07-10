# V2 Component Designs — As-Built Reference

Per-class detailed references for the V2 runtime (`jarvishep2`). Each doc fixes the **shipped code
structure**: which classes are defined, every member function (signature + behavior), the class /
instance attributes, inter-component interfaces, drift from the original design, and the tests that
exercise it.

> **As-built @ `jarvis2` `d0de31a`.** These docs were refreshed from a full code review of the
> shipped `jarvishep2/` package (50 modules / ~75 classes / 26 test files). They supersede the
> design-era specs that these files used to contain. The architecture rationale still lives in
> [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md); the milestone plan in
> [`../V2_DISTRIBUTED_PLAN.md`](../V2_DISTRIBUTED_PLAN.md). For the full design↔code drift list and
> per-component completeness, see **[`../CODE_REVIEW_2.0.md`](../CODE_REVIEW_2.0.md)**.

> **No `ModuleManager`** — module config is a picklable blueprint ([config_schema.md](config_schema.md)),
> execution is the [Worker](worker.md). **No shared `Module` ABC** — `CalculatorModule` and
> `OperasModule` are standalone ([module_base.md](module_base.md)).

## Read order (dependency order)

**Core:**

| # | Component | Shipped module(s) | Status |
|---|-----------|-------------------|--------|
| 1 | [Sample](sample.md) | `jarvishep2/sample.py` | done |
| 2 | [RedisQueue / schema](redis_queue.md) | `jarvishep2/redis_queue.py`, `calculator_pools.py` | done |
| 3 | [Logger (two-layer)](logger.md) | `jarvishep2/logging/`, `sample_logger.py`, `log_kv.py` | done |
| 4 | [UMapper (u→x)](umapper.md) | `jarvishep2/mapper.py` | done |
| 5 | [Workflow / ExecutionPlan](workflow.md) | `jarvishep2/workflow.py` | done (functions, no class) |
| 6 | [Worker](worker.md) | `jarvishep2/worker.py` | done |
| 7 | [CalculatorModule](calculator.md) | `jarvishep2/Module/calculator.py` | done |
| 8 | [TaskFactory](factory.md) | `jarvishep2/factory.py` | done (+ watchdog) |
| 9 | [Sampler base](sampler.md) | `jarvishep2/Sampling/sampler.py` | done (minimal) |
| 10 | [DataRecorder / Archiver](datarecorder.md) | `jarvishep2/archiver.py`, `database.py`, `archive_handoff.py`, `file_ops.py` | done (JSON-rows HDF5) |
| 11 | [Core orchestrator](core.md) | `jarvishep2/core.py`, `client.py` | done |

**Supporting:**

| Component | Shipped module(s) | Status |
|-----------|-------------------|--------|
| [CommandParser](command_parser.md) | `jarvishep2/command_parser.py` | done |
| [env_setup](env_setup.md) | `jarvishep2/env_setup.py` | done |
| [Monitor / Dashboard](monitor.md) | `jarvishep2/dashboard.py`, `monitoring/run_summary.py` | done (text only) |
| [Checkpoint / Resume](checkpoint.md) | `Sampling/runtime_checkpoint.py`, `Sampling/checkpointed_sampler.py` | done |
| [LogLikelihood](likelihood.md) | `jarvishep2/likelihood.py` | done |
| [Library / LibDeps](library.md) | `jarvishep2/library.py` | done (thin) |

**Module & I/O backends:**

| Component | Shipped module(s) | Note |
|-----------|-------------------|------|
| [Module contract & spawn](module_base.md) | `operas.py`, `Module/calculator.py`, `mp_context.py` | no shared ABC |
| [OperasModule](operas.md) | `jarvishep2/operas.py` | preload-per-Worker |
| [I/O parameter system](io_system.md) | `jarvishep2/io_portal.py`, `file_ops.py` | **Portal registry** (JSON/CSV/TSV/DAT/Wolfram; new formats via Portal upgrade) |
| [Subprocess scheduler](subprocess_scheduler.md) | `jarvishep2/async_subprocess.py` | per-Worker |
| [Parameters & Variables](parameters_variables.md) | `Sampling/variables.py` | no `Module/parameters.py` |
| [Nuisance handling](nuisance.md) | — | ⚠️ **stubbed only** |

**Sampling & measurement:**

| Component | Shipped module(s) | Note |
|-----------|-------------------|------|
| [Samplers catalog](samplers_catalog.md) | `Sampling/{bridson,randoms,grid,csv_sampler,seeded_sampler,checkpointed_sampler,stateless_batch,sampling_utils}.py` | 4 stateless samplers (no MCMC/nested/gradient) |
| [Distributor (dispatch)](distributor.md) | `jarvishep2/distributor.py` | Bridson/Random/Grid/CSV only |
| [Adaptive level-set sampler](adaptive_voronoi_contour.md) | `Sampling/adaptive_level_set.py` | ✅ D10 core — feedback-driven level-set tracer, 2≤d≤5 hybrid (Delaunay d≤3 / kNN d=4–5) |
| [Benchmark mode](benchmark.md) | — | ⚠️ **not a module** (see `core.run_check_modules`) |

**Auxiliary / support systems:**

| Component | Shipped module(s) | Note |
|-----------|-------------------|------|
| [Expression helpers (字母运算)](expression.md) | `inner_func.py`, `Sampling/sampling_utils.py` | no `ExpressionEngine` |
| [CLI parsing (命令行解析)](cli.md) | `jarvishep2/client.py` | plain argparse (no `card/argparser.json`) |
| [Config loader & normalization](config_schema.md) | `task_config.py`, `runtime_config.py`, `worker_config.py` | no jsonschema / env checks |
| [Paths & runtime tokens](paths_tokens.md) | `jarvishep2/base.py` | functions, no `Base` class |
| [Project tools](project_tools.md) | — | ⚠️ **not ported** |
| [Utils / versioning / convert / plot](utils.md) | — | ⚠️ **not ported** |

> **Folded-in glue:** `log_kv` → [logger](logger.md); `calculator_pools` → [redis_queue](redis_queue.md);
> `archive_handoff` / `file_ops` / `database` / `io_json` → [datarecorder](datarecorder.md);
> `run_summary` → [monitor](monitor.md); `mp_context` → [module_base](module_base.md);
> `task_config` / `worker_config` → [config_schema](config_schema.md);
> `stateless_batch` / `sampling_utils` / `checkpointed_sampler` → [samplers_catalog](samplers_catalog.md);
> `testing/eggbox.py` is a test operator fixture.

## Shared conventions

- **One Sample per Worker**; same-layer calculators concurrent inside a Sample.
- **Redis is the only cross-process broker**; payloads are IDs + light dicts.
- **`logger` never crosses a process boundary.**
- **`spawn` multiprocessing context** everywhere (`mp_context.get_spawn_context`).
- **No V1 import** — `jarvishep` (V1) is never imported from `jarvishep2`.
- Unit tests use `fakeredis`; integration tests run real spawn Workers.
