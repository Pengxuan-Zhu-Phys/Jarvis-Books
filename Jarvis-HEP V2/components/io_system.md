# Component — I/O parameter system (`jarvishep2/io_portal.py`, `jarvishep2/file_ops.py`)

**Role**: the calculator **Layer-1 I/O** — write a calculator's input file and read its output file.
Per-Sample, per-Module file traffic inside the Worker.
**Status**: **As-built** @ Portal integration (2026-07-10). Formats are delegated to **Jarvis-Portal**.
**Design refs**: [`../DESIGN_PORTAL_IO_2.0.md`](../DESIGN_PORTAL_IO_2.0.md),
[`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §5;
[calculator.md](calculator.md), [paths_tokens.md](paths_tokens.md).
**Reuses V1**: none by import. Format logic reuses the standalone `Jarvis-HEP-Portal` package.

> **As-built:** `io_json.py` is **removed**. Calculator I/O goes through
> `jarvishep2/io_portal.py` → `jarvis_portal.create_entry_point_registry()`. Unknown `type`
> values raise `UnsupportedIOTypeError` (no silent skip). New formats require a Portal
> upgrade only, not a HEP code change.

---

## 1. `io_portal.py` (HEP ↔ Portal bridge)

| Function | Behavior |
|----------|----------|
| `get_io_registry()` | Process-local cached Portal registry (`create_entry_point_registry`). |
| `reset_io_registry_for_tests()` | Drop cache (tests only). |
| `available_io_formats(direction=None)` | List registered format names. |
| `evaluate_io_expression(expression, values)` | HEP-owned sympy eval (`Pi`/`pi`, numeric coerce). Passed into Portal as `IOContext.evaluate_expression`. |
| `build_io_context(...)` | Map sample/module/pack fields → `jarvis_portal.IOContext`. |
| `write_io_input` / `write_io_input_sync` | Resolve format → `adapter.write_input`. |
| `read_io_output` / `read_io_output_sync` | Resolve format → `adapter.read_output`. |
| `UnsupportedIOTypeError` | Unknown type; message includes registered formats. |

### Ownership

| Concern | Owner |
|---------|--------|
| Token path expansion (`@Sdir` / …) | Calculator / CommandParser (before Portal call) |
| Expression evaluation | HEP (`evaluate_io_expression`) |
| Format serialization | Portal adapters |
| Registry / entry points (`jarvishep.io`) | Portal |

### Formats available without HEP changes

Whatever the installed Portal exposes (builtins + entry points). As of Portal 1.3.0:

`JSON`, `CSV`, `TSV`, `DAT`, `Wolfram`

(SLHA/xSLHA appear when Portal registers and exposes them.)

---

## 2. `file_ops.py`

See [datarecorder.md](datarecorder.md) §4 — `delete_path` / `delete_paths` /
`normalize_delete_method` (`shutil` | `rm`), used for transient + staging cleanup. Unrelated to
format adapters.

---

## 3. Flow inside a calculator step

```
CalculatorModule.load_input / _load_input_sync:
  for each input spec:
    path = resolve tokens (@Sdir/…)
    context = build_io_context(sample, pack, module, params)
    portal_obs = write_io_input(spec, params, context, path)
    result ← params ∪ portal_obs

CalculatorModule.read_output / _read_output_sync:
  for each output spec:
    path = resolve tokens
    result ← portal read_io_output(spec, context, path)
```

Unknown or empty `type` raises. Sync path uses `asyncio.run` when no event loop is running.

---

## 4. Concurrency / failure semantics

- Registry is process-local; each spawn Worker builds once on first use.
- No silent skip of non-JSON types.
- Missing Portal package → `ImportError` with install hint.
- Adapter/file errors propagate to fail the Sample (Worker continues).

---

## 5. Interfaces / collaborators

- **CalculatorModule** ([calculator.md](calculator.md)) — sole production caller.
- **Jarvis-Portal** — `IOContext`, `IORegistry`, format adapters.
- **paths_tokens** / **CommandParser** — path token resolution before the bridge.

Dependency: `Jarvis-HEP-Portal>=1.3.0` in `pyproject.toml`.

---

## 6. Tests

- `tests/test_io_portal.py` — registry, unknown type, JSON expression write/read, CSV bridge, calculator integration.
- EggBox / worker calculator suites continue to exercise JSON via Portal end-to-end.
