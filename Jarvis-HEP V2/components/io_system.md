# Component — I/O parameter system (`jarvishep2/io_portal.py`, `jarvishep2/file_ops.py`, `jarvishep2/file_operation_service.py`)

**Role**: the calculator **Layer-1 I/O** — write a calculator's input file and read its output file.
Per-Sample, per-Module file traffic inside the Worker. SAMPLE save/copy/delete is **HEP-owned**.
**Status**: **As-built** @ `jarvis2` `399633b`+ (Portal format R/W + HEP `FileOperationService`).
**Design refs**: [`../DESIGN_PORTAL_IO_2.0.md`](../DESIGN_PORTAL_IO_2.0.md),
[`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §5;
[calculator.md](calculator.md), [paths_tokens.md](paths_tokens.md), [worker.md](worker.md).
**Reuses V1**: none by import. Format logic reuses the standalone `Jarvis-HEP-Portal` package.

> **As-built:** Calculator I/O goes through `jarvishep2/io_portal.py` →
> `jarvis_portal.create_entry_point_registry()`. Unknown `type` raises
> `UnsupportedIOTypeError` (no silent skip). **YAML `save: true` is unchanged** — execution
> is HEP `FileOperationService` / `file_ops.apply_io_save_policy`, **not** Portal adapters.

---

## 1. Ownership boundary

| Concern | Owner |
|---------|--------|
| YAML `input[]` / `output[]` / `save:` syntax | HEP (unchanged) |
| Token path expansion (`@Sdir` / `@PackID` / …) | Calculator / CommandParser (before Portal call) |
| Expression evaluation | HEP (`evaluate_io_expression` → shared `ExpressionContext`) |
| Format serialization (variable R/W only) | **Portal** adapters |
| SAMPLE save/copy (`.temp` / `basename@Module`) | **HEP `FileOperationService`** |
| Transient path delete | HEP `file_ops` / FileOperation (Worker cleanup) |
| Registry / entry points (`jarvishep.io`) | Portal |

Portal `IOContext.sample_save_dir` remains on the dataclass for API compatibility but HEP
passes **`for_portal=True`** so adapters never receive a SAMPLE root and **must not** copy files.

---

## 2. `io_portal.py` (HEP ↔ Portal bridge)

| Function | Behavior |
|----------|----------|
| `get_io_registry()` | Process-local cached Portal registry. |
| `reset_io_registry_for_tests()` | Drop cache (tests only). |
| `available_io_formats(direction=None)` | List registered format names. |
| `evaluate_io_expression(...)` | shared `ExpressionContext` (`Pi`/`pi`, functions, coerce). |
| `build_io_context(..., for_portal=True)` | Map sample/module/pack → `IOContext`; **omits** `sample_save_dir` when `for_portal`. |
| `apply_hep_io_save(...)` | SAMPLE save/.temp via `file_ops` or attached `FileOperationService`. |
| `write_io_input` / `_sync` | Resolve format → `adapter.write_input`. |
| `read_io_output` / `_sync` | Resolve format → `adapter.read_output`. |
| `UnsupportedIOTypeError` | Unknown type; message includes registered formats. |

### Formats available without HEP changes

Whatever the installed Portal exposes (builtins + entry points). Portal ≥1.4.0 V2 surface:

`JSON`, `CSV`, `TSV`, `DAT`, `Wolfram`, `SLHA`, `xSLHA` (when registered).

---

## 3. `file_ops.py` + `file_operation_service.py`

| Symbol | Behavior |
|--------|----------|
| `delete_path` / `delete_paths` | `shutil` or `rm -rf` backends (`Runtime.FileOperation.delete_method`). |
| `save_io_copy` / `apply_io_save_policy` | V1 policy: input only if `save:true`; output `save:true` → SAMPLE/, else → SAMPLE/`.temp`/. |
| `sample_artifact_filename` | `basename` or `basename@Module`. |
| `FileOperationService` | **Dedicated process** (default) or `inline` for tests. Ops: `save_io`, `delete`, `shutdown`. |

Worker starts one service per process (`file_operation_mode: process|inline`), attaches it to
each `CalculatorModule`, and routes cleanup deletes through it.

---

## 4. Flow inside a calculator step

```
CalculatorModule.load_input:
  ensure SAMPLE dir if save/temp needed
  for each input spec:
    path = resolve tokens
    context = build_io_context(..., for_portal=True)   # no sample_save_dir
    portal_obs = write_io_input_sync(spec, data, context, path)  # variables only
    apply_hep_io_save(..., direction="input", file_ops=...)     # SAMPLE copy if save
    merge new keys (file path observable when save:true)

CalculatorModule.read_output:
  same pattern; direction="output" → save or .temp
```

Unknown or empty `type` raises. Sync path uses `asyncio.run` when no event loop is running.

---

## 5. Concurrency / failure semantics

- Registry is process-local; each spawn Worker builds once on first use.
- FileOperation process is daemon; Worker `run()` finally shuts it down.
- Missing Portal package → `ImportError` with install hint.
- Adapter/file errors propagate to fail the Sample (Worker continues).
- Save failures should not silently skip when `save:true` and SAMPLE is materialized.

---

## 6. Interfaces / collaborators

- **CalculatorModule** ([calculator.md](calculator.md)) — sole production Portal caller.
- **Worker** ([worker.md](worker.md)) — starts/attaches `FileOperationService`.
- **Jarvis-Portal** — format adapters (variable R/W only).
- **Sample** — `ensure_sample_materialized` / `save_dir` for SAMPLE landing.

Dependency: `Jarvis-HEP-Portal` in `pyproject.toml` (V2 surface).

---

## 7. Tests

- `tests/test_io_portal.py` — registry, unknown type, JSON expression write/read.
- `tests/test_io_save.py` — `save:true` lands under SAMPLE via HEP (with/without service).
- `tests/test_file_operation_service.py` — inline + process save/delete.
- `tests/test_file_ops.py` — delete backends + Worker cleanup.
