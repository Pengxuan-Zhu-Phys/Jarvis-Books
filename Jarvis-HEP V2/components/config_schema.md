# Component — Config loader & normalization (`jarvishep2/task_config.py`, `jarvishep2/runtime_config.py`, `jarvishep2/worker_config.py`)

**Role**: load the task YAML, merge optional `EnvReqs` defaults, normalize internal runtime /
`Calculators` blocks with V2 defaults, and build the picklable Worker blueprint — all before any
process spawns.
**Status**: **As-built** @ `jarvis2` `0a5e85e` (`EnvReqs.V2` public surface; top-level `Runtime`
rejected; Redis internalized).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §7;
[`../YAML_REFERENCE_2.0.md`](../YAML_REFERENCE_2.0.md) §1 / §4.1 / §5.
**Reuses V1**: none by import. V1-shaped `EnvReqs.Check_default_dependencies` path is retained as
the external defaults entry point; only `EnvReqs.V2` from that file is consumed by V2.

> **As-built:** there is **no `config.py`, no `ConfigLoader`, no jsonschema validation**, and no
> V1 environment checks (`check_ROOT` / `check_PYTHON_env` / …). Config handling is three small
> modules: YAML load + `EnvReqs` merge + path normalization (`task_config.py`), internal block
> normalization with defaults (`runtime_config.py`), and Worker-blueprint assembly
> (`worker_config.py`). Validation is light (type coercion + defaulting + focused hard errors),
> not schema-based.

---

## 1. `task_config.py`

Public YAML scheduling knobs live under **`EnvReqs.V2`** only:

| Key | Default | Notes |
|-----|---------|--------|
| `workers` / singular `worker` | `0` | mutually exclusive aliases; factory uses 1 when ≤ 0 |
| `batch_size` | `256` | sampler submit-group size |

Any other key under `EnvReqs.V2` raises `ValueError`. Top-level **`Runtime` is rejected** with a
migration message. Sibling `EnvReqs` keys (`Python`, `CERN_ROOT`, …) are preserved as metadata
and not validated.

Optional merge sources (task values always win):

1. `EnvReqs.Check_default_dependencies` with `required: true` + `default_yaml_path` → load that
   file and take `EnvReqs.V2` only.
2. `EnvReqs.Runtime.default_runtime_settings` → load that file and take `Runtime` / top-level
   mapping for workers/batch_size (compat path).

| Function | Behavior |
|----------|----------|
| `load_task_yaml(path) -> dict` | read YAML; reject top-level `Runtime`; merge `EnvReqs.V2`; build **internal** `config["Runtime"]` via `normalize_runtime_block({"mode": "redis", **v2_settings})`; stamp `task_yaml` / `task_root` / `project_root` / `task_result_dir` / `scan_name`. |
| `sampling_method(config)` | `Sampling.Method`. |
| `is_check_modules_task(config)` | `Sampling.mode == check_modules`. |
| `resolve_sampling_path(config, raw)` | resolve `&J/`, abspath, or task-yaml-relative path. |
| `check_modules_points_path(config)` | resolve `Sampling.data`/`points_csv` for check-modules. |

## 2. `runtime_config.py`

Defaults: `RUNTIME_DEFAULTS` (mode forced redis from loader, workers 0, batch_size 256,
sample_artifacts `auto`, internal redis / FileOperation / Watchdog), `ARCHIVER_DEFAULTS`,
`CLEANUP_DEFAULTS`, `WATCHDOG_DEFAULTS`.

| Function group | Behavior |
|----------------|----------|
| `normalize_runtime_block` / `get_runtime_block` | normalize the **internal** Runtime dict (workers, batch_size, sample_artifacts, redis, Subprocess, FileOperation, Watchdog). Not a public YAML block. |
| `normalize_file_operation` / `get_delete_method` | internal `FileOperation.delete_method`. |
| `normalize_cleanup_block` / `get_cleanup_config` / `get_staging_dir` / `handoff_to_staging_enabled` | `Calculators.Cleanup`. |
| `normalize_archiver_block` / `get_archiver_config` | `Calculators.Archiver`. |
| `normalize_watchdog_block` / `get_watchdog_config` | internal Watchdog defaults. |
| `get_calculators_block` | `Calculators` block. |
| `workflow_has_calculator` / `workflow_references_sdir` / `_mapping_contains_token` | lazy-materialization flags. |
| `should_eager_materialize` / `should_materialize_on_failure` | drive [Sample](sample.md) materialization. |
| `parse_registered_executables` | raw `LibDeps.registered_executables` entries. |

Redis connection defaults are owned by `jarvishep2.redis_queue.INTERNAL_REDIS_CONFIG`
(`127.0.0.1:6379`); control process fails early on `ping()` if the local service is down.
Optional YAML redis overrides are planned under D12.4, not as-built.

## 3. `worker_config.py`

| Function | Behavior |
|----------|----------|
| `_default_mapper(cfg)` | choose mapper config by `Sampling.Method` (CSV→none, Variables→distribution, else identity). |
| `_config_references_sdir(modules)` | `@Sdir` present in calculator configs. |
| `build_command_parser(config)` | `CommandParser.from_config`. |
| `build_worker_config(...)` | assemble the **picklable Worker blueprint**: sample_config, mapper, opera/calculator modules (Phase-1 resolved), likelihood expressions, timeouts, cleanup/archiver, command_parser payload, optional calculator_pools. |

---

## 4. Flow

```
load_task_yaml(path)
   → merge EnvReqs defaults + task EnvReqs.V2
   → config["Runtime"] internal adapter (mode=redis, workers, batch_size, …)
   → stamp task paths
get_runtime_block / get_archiver_config / get_watchdog_config …
build_worker_config(config, …) → picklable blueprint → Worker (spawn)
```

Fail-fast errors are explicit `ValueError` / `FileNotFoundError` (top-level Runtime, bad/missing
defaults file, unsupported EnvReqs.V2 keys, missing check-modules CSV) — not schema messages.

---

## 5. Interfaces / collaborators

- **core** ([core.md](core.md)) `load_task_yaml` / `build_worker_config` / `get_*`.
- **CommandParser** ([command_parser.md](command_parser.md)) Phase-1 resolution in the blueprint.
- **Sample** ([sample.md](sample.md)) reads the materialization flags.
- **Archiver / Worker / Factory** consume the normalized blocks.

---

## 6. Tests

- `tests/test_task_config_compat.py` — EnvReqs merge, worker/workers alias, reject top-level Runtime,
  unsupported V2 keys.
- Also exercised through `tests/test_cli.py`, `tests/test_d0_integration.py`,
  `tests/test_distributed_acceptance.py`, and calculator/clone_shadow suites.
