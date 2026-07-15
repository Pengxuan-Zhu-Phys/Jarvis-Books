# DESIGN — Jarvis-Portal IO Integration (V2)

**Status**: **implemented**. V2 surface exposes SLHA/xSLHA (Portal ≥1.4.0); V1 surface unchanged.
HEP-v2 imports ``jarvis_portal.v2`` only (interface packages are version-split).
**Milestone**: D9.2 (IO half) / Portal integration  
**Date**: 2026-07-10 · as-built review 2026-07-13
**Scope**: wire `jarvishep2` calculator Layer-1 I/O through **Jarvis-Portal** so new formats ship by upgrading Portal only.

---

## 1. Problem

Today V2 calculator I/O is hardcoded:

| Location | Behavior |
|----------|----------|
| `jarvishep2/io_json.py` | JSON-only Dump write + dotted-entry read |
| `Module/calculator.py` | four `type == "JSON"` checks; **unknown types are silently skipped** |

That blocks real HEP calculators (CSV/SLHA/…), violates OCP, and duplicates work already done in `Jarvis-Portal` (registry, adapters, `jarvishep.io` entry points, CLI examples).

## 2. Goals

1. **Portal owns formats.** Format parse/write logic lives only in `jarvis_portal` (and third-party packages that register under `jarvishep.io`).
2. **HEP owns workflow semantics.** YAML IO-block parsing, `@Sdir`/`@SampleID`/`@PackID` tokens, expression evaluation, save/copy/archive policy, and which variables are selected stay in V2.
3. **Upgrade Portal ⇒ new formats without HEP code changes** for formats that Portal exposes (builtins + entry points).
4. **Unknown `type` hard-fails** with the list of registered formats (no silent skip).
5. **Parity preserved** for existing EggBox / JSON calculator tests (DATABASE golden keys `x,y,z,LogL`).

### Non-goals

- Re-implement SLHA/xSLHA/ROOT inside HEP.
- Port `--convert` / `--plot` (separate packages).
- Change Task-YAML schema (existing `execution.input[]` / `output[]` shape is frozen).
- Finish Distributor sampler registry (other half of D9.2).

---

## 3. Ownership boundary

```
Task YAML ──► CalculatorModule (parse specs, resolve tokens)
                    │
                    │  build IOContext + concrete adapter spec
                    ▼
              jarvishep2.io_portal  (thin bridge)
                    │
                    │  registry.get(type, direction)
                    ▼
              jarvis_portal adapters (JSON/CSV/TSV/DAT/Wolfram/…)
                    │
                    ▼
              filesystem (input write / output read)
```

| Concern | Owner |
|---------|--------|
| YAML `input[]` / `output[]` structure | HEP |
| Path markers `&J`, `@Sdir`, `@SampleID`, `@PackID` | HEP (`CommandParser` / calculator token resolve) |
| Expression eval (`x * Pi`, …) | HEP via `IOContext.evaluate_expression` |
| SAMPLE save/copy/delete (`save: true`, `.temp`, cleanup) | **HEP `FileOperationService`** (dedicated process per Worker; not Portal) |
| Adapter registration & lookup | Portal |
| Format-specific serialization (variable R/W only) | Portal |
| Optional extras for heavy formats | Portal package extras |

**As-built (2026-07-15):** Portal adapters no longer perform SAMPLE copies. HEP builds
`IOContext` **without** `sample_save_dir` for Portal calls; after each Portal write/read,
`CalculatorModule` runs `apply_hep_io_save` via the Worker’s `FileOperationService`.
YAML `save:` syntax is unchanged.

---

## 4. HEP surface design

### 4.1 New module: `jarvishep2/io_portal.py`

| API | Role |
|-----|------|
| `get_io_registry() -> IORegistry` | Process-local cached `create_entry_point_registry()` |
| `reset_io_registry_for_tests()` | Test isolation |
| `evaluate_io_expression(expression, values)` | Sympy evaluator with `Pi`/`pi` (and numeric coerce) — **HEP-owned** |
| `build_io_context(*, sample_info, pack_id, module, evaluate_expression=…)` | Map Worker/sample fields → `IOContext` |
| `async write_io_input(spec, data, *, context) -> dict` | Lookup adapter `input`, call `write_input` |
| `async read_io_output(spec, *, context) -> dict` | Lookup adapter `output`, call `read_output` |
| `write_io_input_sync` / `read_io_output_sync` | `asyncio.run` wrappers for the scheduler sync path |
| `UnsupportedIOTypeError` | Wraps Portal `MissingAdapterError` with HEP wording |

Registry creation:

```python
from jarvis_portal import create_entry_point_registry
registry = create_entry_point_registry()  # builtins + jarvishep.io entry points
```

If `jarvis_portal` is not importable, raise `ImportError` with install hint (`pip install Jarvis-HEP-Portal`).

### 4.2 Spec handoff

HEP builds the adapter spec as a plain dict (Portal does not parse YAML):

```python
spec = {
    **raw_yaml_spec,           # name, type, save, actions/variables, header, …
    "path": resolved_path,     # tokens already expanded by HEP
}
```

Path tokens are resolved **before** the Portal call (same as today). `IOContext.resolve_path` is optional; with absolute/resolved paths, `context.path` is identity + expanduser.

### 4.3 `CalculatorModule` changes

Replace JSON-only branches in:

- `load_input` / `_load_input_sync`
- `read_output` / `_read_output_sync`

with:

```
for each input/output spec:
  type = required non-empty string
  path = resolve tokens
  call io_portal write/read
  merge returned observables
```

**Merge policy (HEP-owned, parity-preserving):**

```
input path:
  result ← copy of input param/observable dict   # historical V2 behavior
  result ← update(portal write_input observables)  # expression dumps, save paths

output path:
  result ← update(portal read_output observables)
```

Missing / empty `type` → `ValueError`. Unknown type → `UnsupportedIOTypeError` listing `registry.available_formats(direction)`.

### 4.4 `io_json.py`

Keep as a **thin local helper** only if still useful for pure unit tests; calculator production
path **must not** import it. As built, `io_json.py` was deleted and expression ownership moved
to the shared `jarvishep2.expression.ExpressionContext`; `io_portal` retains only the Portal
callback wrapper and Calculator-specific numeric-string coercion.

Decision for this WP: **calculator uses only `io_portal`**. `io_json.py` remains for one release as a private fallback utility (not called from calculator) *or* is removed if nothing imports it — prefer removal once tests use portal.

### 4.5 Dependency

`pyproject.toml`:

```toml
dependencies = [
  …
  "Jarvis-HEP-Portal>=1.3.0",
]
```

Dev installs may use editable path:

```bash
pip install -e ../Jarvis-Portal
pip install -e ".[dev,distributed]"
```

---

## 5. Runtime / concurrency

- Portal adapters are **async**; they use `IOContext.run_blocking` for sync file work.
- Calculator **async** path: `await write_io_input` / `await read_io_output`.
- Calculator **sync** path (scheduler attached): `write_io_input_sync` / `read_io_output_sync` via `asyncio.run` when no running loop.
- No global mutable adapter state; registry is process-local and read-mostly after first create (Workers are spawn processes → each Worker builds its own registry once).

---

## 6. Failure semantics

| Case | Behavior |
|------|----------|
| Portal not installed | `ImportError` with install message at first IO call / registry create |
| Unknown format | `UnsupportedIOTypeError` / `MissingAdapterError` — **hard fail**, lists available formats |
| Empty `type` | `ValueError` |
| Adapter internal parse error | propagate (no silent empty observables for missing required output when adapter raises) |
| Expression without evaluator | Portal JSON raises; HEP always supplies `evaluate_expression` |

---

## 7. How new formats appear without HEP changes

1. Author adapter in Portal (or third-party package).
2. Register in `register_builtins()` and/or `jarvishep.io` entry point.
3. Expose via Portal `EXPOSED_FORMATS` when using entry-point discovery filters.
4. Ship new Portal version; users `pip install -U Jarvis-HEP-Portal`.
5. Task YAML sets `type: NewFormat` — HEP looks up by name, no HEP release required.

HEP only needs a new release if the **adapter contract** (`write_input` / `read_output` / `IOContext` fields) changes.

---

## 8. Acceptance criteria

1. Design doc present and `components/io_system.md` updated to as-built Portal bridge.
2. `CalculatorModule` has **zero** `type == "JSON"` string branches.
3. `grep -n 'write_json_input\|read_json_output' jarvishep2/Module/calculator.py` is empty.
4. Unknown IO type test fails loudly with available formats.
5. JSON EggBox unit + worker calculator parity tests green.
6. At least one non-JSON format (CSV) write/read unit test via the HEP bridge green (proves multi-format socket).
7. Existing suite (or targeted calculator + portal tests) green under `pytest`.
8. Plan ledger: Portal/IO portion of D9.2 marked **done** (Distributor registry remains todo if not in this WP).

---

## 9. Implementation plan (this WP)

| Step | Work |
|------|------|
| 1 | Land this design doc + update `components/io_system.md` |
| 2 | Add `jarvishep2/io_portal.py` + dependency |
| 3 | Rewire `CalculatorModule` load/read paths |
| 4 | Remove dead `io_json` usage (delete or keep unused shim) |
| 5 | Tests: `tests/test_io_portal.py` + keep EggBox suite |
| 6 | Run verification; commit |

---

## 10. Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Portal JSON returns expression vars (`xx`,`yy`) into observables | Golden normalize keeps `x,y,z,LogL`; full records may grow — acceptable V1-aligned behavior |
| Expression semantics drift (sympy locals) | Single HEP `evaluate_io_expression` matching prior `io_json` Pi handling |
| Portal version skew | Pin `>=1.3.0`; contract tests against registry API |
| Sync-in-async loop mistakes | Sync wrappers only used when no running loop; async path always awaits |
| Optional heavy formats (SLHA) not yet exposed | Documented as Portal-side exposure; HEP already ready when Portal registers them |

---

## 11. Follow-ups (out of this WP)

- Distributor sampler registry (remainder of D9.2).
- Optional: CLI `Jarvis2 formats` listing `get_io_registry().available_formats()`.
- When Portal exposes SLHA/xSLHA: add HEP integration example under `tests/parity_project` without changing bridge code.
- PLOT / Operas plugin bridges (same pattern, separate designs).

## 12. As-built integration status (2026-07-13)

- Calculator production I/O goes through `jarvishep2.io_portal`; JSON hardcoding is no longer
  the execution path.
- Dump-variable formulas use the same process-local `ExpressionContext` /
  `CompiledExpression` runtime as Operas, Likelihood, Selection, and AdaptiveLevelSet.
- Installed `Jarvis-HEP-Portal 1.3.0` exposes `CSV`, `DAT`, `JSON`, `TSV`, and `Wolfram` for
  discovery. Real CSV/JSON bridge and calculator tests pass.
- Portal contains SLHA/xSLHA adapter code, but those adapters are not in the current exposed
  builtin/entry-point set. This is a Portal exposure/package gap; V2 should not reimplement them.
- V2 still lacks the V1-equivalent `portal formats` discovery command. D11.3 adds a thin CLI
  proxy plus a real SLHA fixture after Portal exposure lands.
