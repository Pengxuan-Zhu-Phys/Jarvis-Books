# V2 Component Designs

Per-class detailed designs for the V2 runtime. Each doc fixes **code structure**, **member
functions** (signature + behavior), **data/schema**, **concurrency/lifecycle/failure
semantics**, **inter-component interfaces**, and **tests + verification logic**.

These refine [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) (architecture) and
are executed by [`../V2_DISTRIBUTED_PLAN.md`](../V2_DISTRIBUTED_PLAN.md) (milestones D0–D7).
The V2 package name is tentatively `jarvishep2` (packaging decided in WP-D0.2).

> **Coverage map:** [`V1_TO_V2_MAP.md`](V1_TO_V2_MAP.md) maps **every** V1 `jarvishep/` module to
> its V2 disposition (own doc / reused / dissolved / retired). Use it as the checklist. Note: V2
> has **no `ModuleManager`** — module config is a picklable blueprint ([workflow.md](workflow.md)),
> execution is the [Worker](worker.md).

## Read order (dependency order)

**Core:**

| # | Component | Module (tentative) | Plan WP | Reuses V1 |
|---|-----------|--------------------|---------|-----------|
| 1 | [Sample](sample.md) | `jarvishep2/sample.py` | D0.1 | extends `jarvishep/sample.py` |
| 2 | [RedisQueue / schema](redis_queue.md) | `jarvishep2/redis_queue.py` | D0.2 | new |
| 3 | [Logger (two-layer)](logger.md) | `jarvishep2/logging/` | D0.3 | wraps `jarvishep/sample_logger.py` |
| 4 | [UMapper (u→x)](umapper.md) | `jarvishep2/mapping.py` | D0.1 | consolidates `variables.py` math |
| 5 | [Workflow / ExecutionPlan](workflow.md) | `jarvishep2/workflow.py` | D2.2 | extends `jarvishep/workflow.py` |
| 6 | [Worker](worker.md) | `jarvishep2/worker.py` | D1.1–D2.3 | reuses calculator + `AsyncSubprocessScheduler` |
| 7 | [CalculatorModule (V2 Δ)](calculator.md) | `jarvishep2/Module/calculator.py` | D1.2–D3 | extends `CalculatorModule` |
| 8 | [TaskFactory](factory.md) | `jarvishep2/factory.py` | D1.1, D5 | replaces `WorkerFactory` |
| 9 | [Sampler binding](sampler.md) | `jarvishep2/Sampling/sampler.py` | D0.1, D1.1 | extends `SamplingVirtial` |
| 10 | [DataRecorder / Archiver](datarecorder.md) | `jarvishep2/archiver.py`, `jarvishep2/hdf5writer.py` | D4 | reuses `GlobalHDF5Writer`, `observable_io` |
| 11 | [Core orchestrator](core.md) | `jarvishep2/core.py`, `client.py` | D1.1+ | adapts V1 init sequence |

**Supporting:**

| Component | Module | Plan WP |
|-----------|--------|---------|
| [CommandParser](command_parser.md) | `jarvishep2/command_parser.py` | D3.1 |
| [env_setup](env_setup.md) | `jarvishep2/env_setup.py` | D3.2 |
| [Monitor / Dashboard](monitor.md) | `jarvishep2/dashboard.py` | D5.2 |
| [Checkpoint / Resume](checkpoint.md) | `…/runtime_checkpoint.py` | D6.2 |
| [LogLikelihood](likelihood.md) | `jarvishep2/Module/likelihood.py` | D1.1+ |
| [Library / LibDeps](library.md) | `jarvishep2/library.py` | D2.3, D3.1 |

**Module & I/O backends:**

| Component | Module | Disposition |
|-----------|--------|-------------|
| [Module base contract](module_base.md) | `jarvishep2/Module/module.py` | unify `execute`; ModulePool → Redis free-pool |
| [OperasModule (in-process backend)](operas.md) | `jarvishep2/Module/operas.py` | preload-per-Worker |
| [I/O parameter system](io_system.md) | `jarvishep2/IOs/` | Layer-1 SLHA/JSON/xSLHA/portal + registry |
| [Subprocess scheduler](subprocess_scheduler.md) | `jarvishep2/async_subprocess.py` | reused, now per-Worker |
| [Parameters & Variables](parameters_variables.md) | `Module/parameters.py`, `Sampling/variables.py` | feed UMapper / input files |
| [Nuisance handling](nuisance.md) | `Module/nuisance_*`, `Sampling/nuisance_sampler.py` | nuisance loop → Worker |

**Sampling & measurement:**

| Component | Module | Disposition |
|-----------|--------|-------------|
| [Samplers catalog & MCMC kernel](samplers_catalog.md) | `jarvishep2/Sampling/` (all) | reused; submit path → Redis |
| [Distributor (dispatch + resume registry)](distributor.md) | `jarvishep2/distributor.py` | reused; selector only |
| [Benchmark mode](benchmark.md) | `jarvishep2/benchmark.py` | retargeted to distributed path + efficiency metric |

**Auxiliary / support systems** (the V1-style helper subsystems):

| Component | Module | Mirrors V1 |
|-----------|--------|-----------|
| [Expression engine (字母运算)](expression.md) | `jarvishep2/expression.py` | `inner_func.py` + `evaluate_selection` (sympy) |
| [CLI parsing (命令行解析)](cli.md) | `jarvishep2/client.py`, `card/argparser.json` | `client.py` + `card/argparser.json` |
| [Config loader & schema](config_schema.md) | `jarvishep2/config.py`, `runtime_config.py` | `config.py` (jsonschema) |
| [Paths & runtime tokens](paths_tokens.md) | `jarvishep2/base.py` | `base.py` (`&J/`, `@Sdir`/`@PackID`) |
| [Project tools](project_tools.md) | `project_scaffold/packager/official_library.py` | same (retargeted to `Jarvis2`) |
| [Utils / versioning / convert / plot](utils.md) | `utils.py`, `versioning.py`, `dataconvert.py`, `plot.py` | same |

Fully mapped — see [`V1_TO_V2_MAP.md`](V1_TO_V2_MAP.md). Only trivial glue has no dedicated doc and
is folded into another: `log_kv` → [logger](logger.md), `delete_method`/`file_ops` →
[datarecorder](datarecorder.md) §5, `io_manager` → [subprocess_scheduler](subprocess_scheduler.md) §5,
`sample_archive` → [datarecorder](datarecorder.md), `Module/library` → [library](library.md),
`bucketallocator` → [paths_tokens](paths_tokens.md), `__init__`/`__main__` (package entry).

## Shared conventions

- **One Sample per Worker**; same-layer calculators concurrent inside a Sample.
- **Redis is the only cross-process broker**; payloads are IDs + light dicts.
- **`logger` never crosses a process boundary.**
- **`spawn` multiprocessing context** everywhere.
- Parity is checked against **captured V1 golden outputs**; unit tests use `fakeredis`.
- Every class keeps a **stateless-where-possible** shape so it survives `spawn` pickling.
