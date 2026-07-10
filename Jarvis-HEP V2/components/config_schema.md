# Component — Config loader & normalization (`jarvishep2/task_config.py`, `jarvishep2/runtime_config.py`, `jarvishep2/worker_config.py`)

**Role**: load the task YAML, normalize the optional `Runtime`/`Calculators` blocks with V2
defaults, and build the picklable Worker blueprint — all before any process spawns.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `runtime_config.py` 272 + `task_config.py` 83 +
`worker_config.py` 139 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §7.
**Reuses V1**: none by import.

> **As-built drift (large):** there is **no `config.py`, no `ConfigLoader`, no jsonschema
> validation, and no environment checks** (`check_ROOT`/`check_PYTHON_env`/…). Config handling is
> three small modules: YAML load + path normalization (`task_config.py`), block normalization with
> defaults (`runtime_config.py`), and Worker-blueprint assembly (`worker_config.py`). Validation is
> light (type coercion + defaulting), not schema-based.

---

## 1. `task_config.py`

| Function | Behavior |
|----------|----------|
| `load_task_yaml(path) -> dict` | read YAML, infer project root, derive scan name + `task_result_dir` (`outputs/<scan>`), normalize `Runtime`, stamp `task_yaml`/`task_root`/`project_root`/`task_result_dir`/`scan_name`. |
| `sampling_method(config)` | `Sampling.Method`. |
| `is_check_modules_task(config)` | `Sampling.mode == check_modules`. |
| `resolve_sampling_path(config, raw)` | resolve `&J/`, abspath, or task-yaml-relative path. |
| `check_modules_points_path(config)` | resolve `Sampling.data`/`points_csv` for check-modules. |

## 2. `runtime_config.py`

Defaults: `RUNTIME_DEFAULTS` (mode `auto`, workers 0, batch_size 256, sample_artifacts `auto`),
`ARCHIVER_DEFAULTS`, `CLEANUP_DEFAULTS`, `WATCHDOG_DEFAULTS`. Valid sets for modes/artifacts/
cleanup/archiver.

| Function group | Behavior |
|----------------|----------|
| `normalize_runtime_block` / `get_runtime_block` | normalize `Runtime` (mode∈{auto,redis}, workers, batch_size, sample_artifacts, `redis`, `Subprocess`, `FileOperation`, `Watchdog`). |
| `normalize_file_operation` / `get_delete_method` | `Runtime.FileOperation.delete_method`. |
| `normalize_cleanup_block` / `get_cleanup_config` / `get_staging_dir` / `handoff_to_staging_enabled` | `Calculators.Cleanup` (strategy `mv_to_staging`/`direct`, staging dir). |
| `normalize_archiver_block` / `get_archiver_config` | `Calculators.Archiver` (mode thread/process, batch_size, flush_interval, strategy, delete_after_archive). |
| `normalize_watchdog_block` / `get_watchdog_config` | `Runtime.Watchdog` (enabled, stale_sec, poll_interval_sec, max_sample_retries). |
| `get_calculators_block` | `Calculators` block. |
| `workflow_has_calculator` / `workflow_references_sdir` / `_mapping_contains_token` | lazy-materialization flags (replaces the designed Workflow properties). |
| `should_eager_materialize` / `should_materialize_on_failure` | drive [Sample](sample.md) materialization. |
| `parse_registered_executables` | raw `LibDeps.registered_executables` entries. |

## 3. `worker_config.py`

| Function | Behavior |
|----------|----------|
| `_default_mapper(cfg)` | choose mapper config by `Sampling.Method` (CSV→none, Variables→distribution, else identity). |
| `_config_references_sdir(modules)` | `@Sdir` present in calculator configs. |
| `build_command_parser(config)` | `CommandParser.from_config`. |
| `build_worker_config(config, *, task_result_dir, sample_dirs=None, opera_modules=None, calculator_modules=None, likelihood_expressions=None, parser=None, extra=None) -> dict` | assemble the **picklable Worker blueprint**: sample_config (+ lazy-materialization flags), mapper, opera/calculator modules (Phase-1 resolved), likelihood expressions, pull_timeout, delete_method, staging dir + handoff, cleanup/archiver config, the picklable command_parser payload, optional calculator_pools. |

---

## 4. Flow

```
load_task_yaml(path) → config (Runtime normalized, paths stamped)
get_runtime_block / get_archiver_config / get_watchdog_config …  → normalized blocks
build_worker_config(config, …) → picklable blueprint → Worker (spawn)
```

Fail-fast errors are explicit `ValueError`/`FileNotFoundError` (bad/missing YAML, missing
check-modules CSV, undefined sampling variables) — not schema messages.

---

## 5. Interfaces / collaborators

- **core** ([core.md](core.md)) `load_task_yaml` / `build_worker_config` / `get_*`.
- **CommandParser** ([command_parser.md](command_parser.md)) Phase-1 resolution in the blueprint.
- **Sample** ([sample.md](sample.md)) reads the materialization flags.
- **Archiver / Worker / Factory** consume the normalized blocks.

---

## 6. Tests

Exercised through `tests/test_cli.py` (9), `tests/test_d0_integration.py` (6),
`tests/test_distributed_acceptance.py` (6) (YAML load, Runtime/Archiver/Watchdog defaults, worker
blueprint), and the calculator/clone_shadow suites.
