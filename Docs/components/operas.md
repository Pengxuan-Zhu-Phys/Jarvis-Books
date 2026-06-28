# Component — OperasModule (`jarvishep2/Module/operas.py`)

**Role**: the **in-process** module backend — a Python operator (Jarvis-Operas integration) that
computes observables in-process (microseconds–milliseconds), no external subprocess. The light
half of the workflow; the calculator is the heavy half.
**Status**: design — plan WP-D1.1 (opera path is the MVP backend).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §11;
[worker.md](worker.md) (`_run_opera_step`), [module_base.md](module_base.md).
**Reuses V1**: `OperasModule(Module)` (`jarvishep/Module/operas.py`) — the `_OperasSampleLoggerBridge`,
`call_mode` (sync/`acall` async), IO specs, registry resolution, timeout. Behavior unchanged; the
V2 work is **preloading + caching the operator function per Worker**.

---

## 1. Responsibilities

1. Resolve and call a registered Python operator on a Sample's observables/params.
2. Support `call_mode: sync | acall` (async operator dispatch — undocumented V1 feature, kept).
3. Bridge the per-Sample logger (`_OperasSampleLoggerBridge`) so operator logs land in the
   Sample log.
4. Honor `selection` cuts and `timeout`.
5. **V2**: be **preloaded once per Worker** (the operator function imported + cached at Worker
   startup), not imported per Sample.

---

## 2. Structure (delta over V1)

```python
class OperasModule(Module):
    def __init__(self, name, config): ...
    call_mode: str            # "sync" | "acall"
    timeout: float | None
    func: Callable            # the operator (cached)
    # --- V2 addition ---
    def preload(self) -> None: ...        # import + cache the operator once (Worker startup)
    def execute(self, observables, sample_info) -> dict: ...
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `preload` | `() -> None` | Resolve the operator via the registry + `importlib`, cache `self.func`. Called once per Worker. |
| `execute` | `(observables, sample_info) -> dict` | Build input observables, call the operator (sync or `acall` with timeout), return new observables. |
| `_build_input_observables` | `(observables, slogger=None) -> dict` | (V1) assemble the operator's inputs from the IO specs. |
| `_resolve_entry` | `(data, entry)` | (V1) pull a named entry from the operas registry. |
| `_run_with_timeout` | `(callable, timeout, label)` | (V1) bounded sync call. |
| `_run_coro` | `(coro)` | (V1) run an async operator (`acall`). |
| `set_funcs` / `set_logger` | `(...)` | (V1) wire the expression-funcs context + the per-Sample logger bridge. |

---

## 4. Execution within the Worker

```
Worker._init: for each OperasModule: op.preload()          # import + cache once
Worker._run_opera_step(step, sample):
    obs = op.execute(sample.info["observables"], sample.info)
    sample.info["observables"].update(obs)
```

Opera steps run **inline** on the Worker's main thread (they are CPU-light, no subprocess). A
layer of multiple operas runs sequentially (microseconds each); calculators are what fan out.

---

## 5. Concurrency / determinism / failure semantics

- **Preload once per Worker** (the V2 change): the dynamic `import` cost is paid at startup, not
  per Sample — material for light/fast scans.
- Operator is treated as **pure** (observables → observables); determinism gives golden parity.
- `acall` async operators run on the Worker's event loop with a timeout; a hung operator fails
  the Sample (logged), never the Worker.
- Selection cut short-circuits via the [expression engine](expression.md); a cut Sample skips
  later layers (same semantics as V1).
- Checkpoint: `export_blueprint`/`restore_blueprint` keep opera state across resume (carried
  from the throughput-core fix).

---

## 6. Interfaces

- **Worker** → `preload` (startup) + `execute` (per Sample).
- **Module base** → inherits `selection`/`evaluate_selection` ([module_base.md](module_base.md)).
- **Expression engine** → `set_funcs` context for operator-side formulas.
- **Logger** → `_OperasSampleLoggerBridge` to the per-Sample log.
- **Registry** → operator resolution (operas full-name map).

---

## 7. Tests (`tests/test_operas_v2.py`)

Unit:
1. **Parity** — `execute(observables, sample_info)` equals the V1 result for the same inputs
   (golden), sync and `acall`.
2. **Preload once** — operator imported once per Worker (counter), reused across Samples.
3. **Selection** — a failing cut short-circuits; later layers skipped; status correct.
4. **Timeout** — a hung operator fails the Sample with a log, not the Worker.
5. **Logger bridge** — operator log lines land in the per-Sample log via the bridge.
6. **Spawn** — operator resolves from importable refs in a child process (no closures).

Verification logic: test 1 keeps opera science identical; test 2 is the per-Worker preload
contract that removes per-Sample import cost.

---

## 8. Open questions

- Whether `acall` operators should share the Worker's calculator scheduler loop or a dedicated
  one (default: the Worker event loop; revisit if async operas + async calculators contend).
- Batched/vectorized operas are **retired** (incompatible with one-Sample-per-Worker).
