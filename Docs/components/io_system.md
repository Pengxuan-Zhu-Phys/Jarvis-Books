# Component — I/O parameter system (`jarvishep2/IOs/`)

**Role**: the calculator **Layer-1 I/O** — write a calculator's input file and read its output
file in the configured format (SLHA / JSON / xSLHA / portal), via a pluggable format registry.
This is the per-Sample, per-Module synchronous file traffic the design keeps **unchanged**.
**Status**: design — plan WP-D1.2 (calculator path), reused from V1.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §5 (Layer 1 vs
Layer 2); `../../v1/design/IO_PLUGIN_LAYER_DESIGN.md` (the registry direction);
[calculator.md](calculator.md), [paths_tokens.md](paths_tokens.md).
**Reuses V1**: `IOs/` — `IOfile`/`InputFile`/`OutputFile`, `IORegistry` + adapters, SLHA/JSON/
xSLHA/FileOutput/portal handlers. Behavior unchanged; the V2 work is making each handler **resolve
per-Sample tokens at the Worker** and stay independent of any global state.

---

## 1. Responsibilities

1. **Write input** files for a calculator from parameter/observable values (SLHA, JSON, portal).
2. **Read output** files into observables (SLHA, JSON, xSLHA, generic FileOutput, portal).
3. Resolve **paths + runtime tokens** (`@Sdir/@SampleID/@PackID`) at write/read time (Phase 2).
4. Dispatch by **format name + direction** through the `IORegistry` (the IO_PLUGIN_LAYER design).

This is **Layer 1**: it runs inside the Worker, per Sample, per calculator step, **synchronously**
— the calculator chain must finish its file I/O before the Sample is done (design §5). It is
distinct from the Archiver (Layer 2, batched final persistence).

---

## 2. Structure (reused)

```python
class IOfile(Base):                       # base: path resolution + sync fs helpers
    def decode_path(self, path) -> None: ...
    def _resolve_runtime_tokens(self, text) -> str: ...      # @Sdir/@SampleID/@PackID
    def create(self, ...): ...            # write input
    def load(self, ...): ...              # read output
class InputFile(IOfile): ...              # SLHAInputFile, JsonInputFile, PortalInputFile
class OutputFile(IOfile): ...             # SLHAOutputFile, JsonOutputFile, xSLHAOutputFile, FileOutput, PortalOutputFile

class IORegistry:
    def register(self, format_name, direction, adapter): ...
    def get(self, format_name, direction) -> IOAdapter: ...
    def available_formats(self, direction=None) -> list[str]: ...
```

Handlers by format × direction:

| Format | Input | Output |
|--------|-------|--------|
| SLHA | `SLHAInputFile` | `SLHAOutputFile`, `xSLHAOutputFile` |
| JSON | `JsonInputFile` | `JsonOutputFile` |
| file (generic) | — | `FileOutput` |
| portal | `PortalInputFile` | `PortalOutputFile` |

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `IORegistry.register` | `(format, direction, adapter)` | Register a format handler (input/output). |
| `IORegistry.get` | `(format, direction) -> adapter` | Look up the handler for a calc's IO spec. |
| `IOfile.decode_path` | `(path) -> None` | Resolve `&J/` + per-Sample tokens to a concrete path. |
| `IOfile.create` | `(...)` | Write the input file from param/observable values (format-specific `_write_sync`). |
| `IOfile.load` | `(...)` | Read the output file into observables (format-specific `_read_sync`). |
| `*InputFile._write_sync` | `(param_values)` | SLHA/JSON/portal write (expression-evaluated entries). |
| `*OutputFile._read_sync` | `()` | SLHA/JSON/xSLHA/file/portal read → observables. |

---

## 4. Flow inside a calculator step

```
Worker._run_calculator_step:
   input  = IORegistry.get(spec.format, "input").create_handler(...)
   input.decode_path(@Sdir/in.slha); input.create(param_values)     # WRITE (sync)
   calc.run_command(...)                                            # RUN (subprocess)
   output = IORegistry.get(spec.format, "output").create_handler(...)
   output.decode_path(@Sdir/out.slha); obs = output.load()         # READ (sync)
```

All three steps are synchronous and per-Sample; the work directory is exclusively owned by the
current Sample during the chain (design §3).

---

## 5. Concurrency / determinism / failure semantics

- **No global state**: each handler resolves its own per-Sample path; safe under Worker
  concurrency and `clone_shadow` (each Sample's dir is isolated).
- Token resolution is **Phase 2** (Worker-side) via [paths_tokens.md](paths_tokens.md).
- A malformed/missing output file → the calculator step fails with a clear error and a Sample log
  entry (no silent empty observables).
- Expression-valued input entries go through the [expression engine](expression.md) (e.g. derived
  SLHA fields).
- Format/handler chosen by the registry; an unknown format fails at config load (boot), not at
  run time.

---

## 6. Interfaces

- **CalculatorModule** → `create` input before `run_command`, `load` output after.
- **paths_tokens** → `decode_path` / `@Sdir` resolution (materialize on first touch).
- **expression engine** → expression-valued input entries.
- **config_schema** → validates declared formats against `IORegistry.available_formats`.

---

## 7. Tests (`tests/test_io_system.py`)

Unit:
1. **Round-trip per format** — write then read SLHA/JSON/xSLHA/portal reproduces the values
   (golden vs V1 fixtures).
2. **Registry dispatch** — `get(format, direction)` returns the right handler; unknown format
   raises at config validation.
3. **Token paths** — `decode_path` resolves `@Sdir/@SampleID` under the per-Sample save dir.
4. **Isolation** — two concurrent Samples writing the same calculator's input don't collide
   (distinct dirs, no shared buffer).
5. **Bad output** — a missing/garbled output file raises a clear error + Sample log entry.
6. **Expression entries** — an expression-valued SLHA field evaluates correctly on write.

Verification logic: test 1 keeps calculator I/O byte-compatible with V1 (the frozen Layer-1
behavior); test 4 ensures Worker concurrency safety.

---

## 8. Open questions

- Completing the `IO_PLUGIN_LAYER_DESIGN` migration (registry fully replacing any hardcoded format
  dispatch) — track as a V2 cleanup, not a behavior change.
- New formats (e.g. HepMC for Rivet) register through `IORegistry` without touching the Worker.
