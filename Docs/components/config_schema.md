# Component — Config loader & schema validation (`jarvishep2/config.py`, `jarvishep2/runtime_config.py`)

**Role**: load the task YAML, validate it against a formal schema, normalize optional sections,
and run environment/dependency checks — all before any process is spawned.
**Status**: design — auxiliary; D0.2 (Runtime/redis block) onward.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §7;
discussion `Jarvis-HEP_Discussion_Summary_2026-06-21.md` §1 (Pydantic/strictyaml idea);
plan invariant #1 (frozen, additive-only schema).
**Reuses V1**: `config.py` `ConfigLoader` (jsonschema Draft7, `card/schema/`), `runtime_config.py`
normalization, env checks (`check_PYTHON_env`/`check_ROOT`/`check_OS_requirement`).

---

## 1. Responsibilities

1. **Load** YAML (anchors/includes) and **validate** against the frozen schema; every new V2 key
   (`Runtime.mode: redis`, `Runtime.redis`, `Calculators.Archiver`, `LibDeps.registered_executables`,
   `env_setup`, `Runtime.FileOperation`, `logging`) is **optional** with a v1-equivalent default.
2. **Normalize optional sections** (e.g. inject an empty `Calculators` for opera-only tasks).
3. **Environment checks** (Python/ROOT/OS/package versions) with the V1 bugs fixed.
4. Produce the **normalized config** consumed by core/factory/workers — picklable, no live
   objects.

---

## 2. Structure

```python
class ConfigLoader(Base):
    schema: dict
    raw: dict
    normalized: dict
    def load_config(self, path) -> None: ...
    def validate_config(self) -> None: ...
    def normalize(self) -> dict: ...
    def runtime_block(self) -> dict: ...           # runtime_config.normalize_runtime_block
    def check_environment(self) -> None: ...
```

Schema strategy: keep **jsonschema Draft7** cards (`card/schema/`) as the contract, optionally
add a thin **Pydantic** typed view on top for editor help + clearer errors (discussion §1) —
the jsonschema remains the source of truth so the frozen contract is unchanged.

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `load_config` | `(path) -> None` | Read YAML (anchors); store `raw`. |
| `set_schema` / `validate_config` | `(...)` | Validate `raw` against the schema (Draft7); raise with a path-pointed message on failure. |
| `_normalize_optional_sections` | `() -> None` | Inject defaults (empty `Calculators`, default `Runtime`, etc.) so optional sections are schema-safe. |
| `runtime_block` | `() -> dict` | Normalize `Runtime` (`mode∈{auto,thread,process,redis}`, `workers`, `batch_size`, `redis`, `Archiver`, `FileOperation`); V2 adds `redis` to the valid modes. |
| `check_PYTHON_env` | `() -> None` | Package/version checks. |
| `check_ROOT` | `() -> None` | ROOT detection — **fixed**: `subprocess.run(check=True)` so failure is real (V1 bug #9: always reported success). |
| `check_OS_requirement` | `(req) -> None` | OS/version gate. |
| `get_modules` / `analysis_calculator` / `analysis_Library` | `(...)` | Resolve module/calculator/library config (paths left as tokens for the Worker; Phase-1 statics via [CommandParser](command_parser.md)). |

V1 bugs fixed here (invariant #14, if these files are touched): `config.py:327`
`FileExistsError`→`FileNotFoundError`; `config.py:125` bare `except:`; `config.py:233-241`
ROOT `check=True`.

---

## 4. Validation flow

```
load_config(path) → raw
_normalize_optional_sections()           # opera-only safe, Runtime defaults
validate_config()                        # jsonschema Draft7 (frozen contract)
runtime_block()                          # mode/redis/archiver/file-op normalization
check_environment()                      # Python/ROOT/OS/packages (fixed checks)
compile-and-validate all expressions     # expression.md §5 (free_symbols vs declared)
→ normalized config (picklable) → core/factory/workers
```

Fail-fast: schema, environment, **and** expression errors all surface at boot — never after a
long scan starts.

---

## 5. Concurrency / lifecycle / failure semantics

- Pure control-process work at boot; the normalized config is the picklable blueprint shipped to
  Workers (spawn).
- A schema violation raises with the JSON path to the offending key (actionable).
- Optional-key defaults mean **existing V1 YAMLs validate unchanged** (invariant #1) — a parity
  test runs a V1 task YAML through the V2 loader.
- Environment checks no longer silently pass (the V1 ROOT/`except:` traps are closed).

---

## 6. Interfaces

- **core.init_configparser** → load + validate + normalize.
- **CommandParser** → Phase-1 static resolution of the normalized config.
- **Expression engine** → boot-time compile/validate of all expressions.
- **runtime_config** → `Runtime` block normalization (shared with the V1 module's logic).

---

## 7. Tests (`tests/test_config_schema.py`)

Unit:
1. **V1 YAML compatibility** — a battery of existing V1 task YAMLs validate unchanged (invariant
   #1).
2. **Optional new keys** — `Runtime.mode: redis`, `Archiver`, `registered_executables`,
   `env_setup`, `FileOperation` parse and default correctly; absence = v1 behavior.
3. **Schema rejection** — malformed configs raise with a path-pointed message.
4. **Env checks fixed** — ROOT-absent reports failure (regression for V1 bug #9); missing file
   raises `FileNotFoundError` (bug #1); no bare `except` swallows `KeyboardInterrupt` (bug #8).
5. **Expression validation** — an undeclared symbol in a formula fails at load.
6. **Picklability** — the normalized config pickles under spawn.

Verification logic: test 1 is the frozen-contract gate; test 4 closes three known V1 silent
bugs as part of touching this file.

---

## 8. Open questions

- Pydantic typed layer now vs. later (default: jsonschema is the contract; Pydantic is an
  optional ergonomics layer, not the source of truth).
- `include`/modularization for large YAMLs (discussion §1 — deferred enhancement).
