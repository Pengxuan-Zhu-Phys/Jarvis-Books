# Component — CommandParser (`jarvishep2/command_parser.py`)

**Role**: two-phase resolution of calculator commands and paths. Resolve everything static once
after YAML load; leave only strong per-Sample tokens for the Worker. Owns `registered_executables`.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `command_parser.py` 333 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §8.
**Reuses V1**: none by import (depends on [`base.py`](paths_tokens.md) decoders).

---

## 1. Classes & module constants

Constants: `SAMPLE_TOKENS = ("@SampleID","@Sdir","@PackID")`, regexes `_LIBDEPS_PATTERN`
(`${LibDeps:…}`), `_SCAN_PATTERN` (`${Scan:…}`), `_ROOT_PATH_PATTERN` (`@{ROOT path}`),
`_registered_name_pattern(name)`.

### 1.1 `ResolvedExecutable` — `@dataclass(frozen=True)`
Fields: `name`, `path`, `resolution` (`direct_path`|`symlink`).

### 1.2 `CommandParser`
**Attributes:** `project_root`, `scan_name`, `libdeps_paths: dict[str,str]`,
`libdeps_values` (`path`, `make_paraller`), `root_path`,
`registered: dict[str,ResolvedExecutable]`, `registered_symlink_root`.

| Method | Behavior |
|--------|----------|
| `from_picklable(payload)` (`@classmethod`) | rebuild from the picklable Worker-config payload. |
| `from_config(config, *, project_root=None, task_root=None, root_path=None, register_executables=True)` (`@classmethod`) | build from YAML; preflight can set `register_executables=False` until a LibDeps build makes an executable source exist. |
| `register(spec)` | resolve one `registered_executables` entry → `ResolvedExecutable` (`direct_path` abspath or created `symlink`); validates source existence. |
| `resolve_static(text)` | **Phase 1**: `expand_j` (`&J/`), `${Scan:…}`, `${LibDeps:…}`, `@{ROOT path}`, registered names, `~`; fail explicitly if ROOT has not been configured. |
| `resolve_static_config(config)` | apply Phase 1 across a whole config, **preserving** per-Sample tokens (`_walk_sample_aware_static`). |
| `resolve_sample(text, *, sample_info, pack_id=None, stage="execution", field="text")` | **Phase 2**: resolve `@PackID/@SampleID/@Sdir` (materializes on `@Sdir`); raise if a static token survived into Phase 2. |
| `has_static_tokens(text)` | True if any Phase-1 token remains (`&J`, LibDeps, Scan, a registered name not already a path). |
| `_replace_scan_token` / `_replace_libdeps_token` / `_replace_root_path_token` / `_replace_registered_names` | regex substitution helpers. |

---

## 2. Module-level functions

| Function | Behavior |
|----------|----------|
| `_contains_sample_tokens(text)` / `_walk_strings(value, callback)` | helpers (token check; recursive string walk). |
| `_registered_specs(config)` | delegate to `runtime_config.parse_registered_executables`. |
| `_build_libdeps_context(config, *, project_root)` | map LibDeps module names plus the block scalar tokens (`path`, `make_paraller`). |
| `prepare_calculator_modules(modules, parser)` | apply Phase-1 resolution to calculator module configs. |

---

## 3. Two-phase flow

```
YAML load (control):  resolve_static_config(config) → Phase-1 config (no static tokens left;
                       @SampleID/@Sdir/@PackID preserved)
Worker (per Sample):   resolve_sample(cmd, sample_info, pack_id) → final command
```

After Phase 1 a registered name (`eggboxlk`) becomes its absolute path; the Worker later resolves
`@Sdir` etc. A leftover static token in Phase 2 → hard error (no silent passthrough).

---

## 4. Interfaces / collaborators

- **core** ([core.md](core.md)) runs Phase 1 (`init_command_parser` / `build_worker_config`).
- **CalculatorModule** ([calculator.md](calculator.md)) `_resolve_runtime_tokens` calls
  `resolve_sample` (Phase 2).
- **base.py** ([paths_tokens.md](paths_tokens.md)) `expand_j` / `decode_path` / `scan_output_root`.
- **LibraryManager** ([library.md](library.md)) reads `registered`.
- **runtime_config** ([config_schema.md](config_schema.md)) `parse_registered_executables`.

---

## 5. Tests

`tests/test_command_parser.py` (12): phase separation (no static tokens after Phase 1), registered
executable direct_path/symlink, Phase-2 token resolution + leftover-token error, picklability.
