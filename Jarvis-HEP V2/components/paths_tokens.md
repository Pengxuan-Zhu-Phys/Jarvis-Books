# Component — Paths & Runtime Tokens (`jarvishep2/base.py`)

**Role**: the path foundation. Resolves the `&J/` project-root marker and per-scan output layout.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `base.py` 84 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §3, §8;
[command_parser.md](command_parser.md), [sample.md](sample.md).
**Reuses V1**: none by import.

> **As-built drift:** there is **no `Base` class** — `base.py` is a set of module functions. The
> per-Sample tokens `@SampleID`/`@Sdir`/`@PackID` are resolved by
> [`CommandParser.resolve_sample`](command_parser.md) and [`Sample.resolve_token`](sample.md), not
> here. `base.py` covers only the **static** `&J/` / path-decoding side.

---

## 1. Module constants

`_PROJECT_MARKERS = (".jarvis-project.json", "jarvis.project.yaml")`,
`_TASK_ROOT_ENV_VARS = ("JARVIS_HEP_TASK_ROOT", "JHEP_TASK_ROOT")`.

## 2. Module-level functions

| Function | Behavior |
|----------|----------|
| `env_task_root() -> str\|None` | abspath from `JARVIS_HEP_TASK_ROOT`/`JHEP_TASK_ROOT` if set. |
| `infer_project_root(start=None) -> str` | walk up to a project marker (or a `bin/` parent), else the start dir. |
| `expand_j(text, *, project_root) -> str` | replace `&J/` / `&J` with the absolute project root; **rejects** the legacy `&J/src/card/` prefix. |
| `decode_path(path, *, project_root, base_dir=None) -> str` | resolve `&J/`, `~`, URLs, abs/relative → absolute (anchored on `base_dir` or project root); raise on an unresolved `&` token. |
| `scan_output_root(*, project_root, scan_name) -> str` | `outputs/<scan_name>`. |

---

## 3. Token split

- **Static** (`&J/`, `${LibDeps:…}`, `${Scan:…}`, registered names) → Phase 1, control process,
  via `expand_j`/`decode_path` + [CommandParser](command_parser.md).
- **Per-Sample** (`@SampleID`, `@Sdir`, `@PackID`) → Phase 2, Worker — resolving `@Sdir` triggers
  `Sample.materialize()` (lazy). See [command_parser.md](command_parser.md) / [sample.md](sample.md).

---

## 4. Lifecycle / failure semantics

- `&J/` and path decoding are deterministic and cheap; an unresolved `&` token raises (no silent
  passthrough).
- The output layout (`outputs/<scan>/{DATABASE,SAMPLE,staging}`, `checkpoints/<scan>/<sampler>/
  state.pkl`) is owned across `base.py` (`scan_output_root`), `archive_handoff.resolve_staging_dir`,
  and `runtime_checkpoint.checkpoint_path`.

---

## 5. Interfaces / collaborators

- **CommandParser** ([command_parser.md](command_parser.md)) uses `expand_j`/`decode_path`/
  `scan_output_root` for Phase-1 resolution.
- **task_config** ([config_schema.md](config_schema.md)) uses `infer_project_root` /
  `scan_output_root`.
- **Sample** ([sample.md](sample.md)) resolves per-Sample tokens + materializes on `@Sdir`.

---

## 6. Tests

Exercised through `tests/test_command_parser.py` (12) (`&J/`, LibDeps, Scan resolution) and the
calculator/integration suites (output-layout paths).
