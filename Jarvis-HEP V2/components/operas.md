# Component — OperasModule (`jarvishep2/operas.py`)

**Role**: the **in-process** module backend — a Python operator that computes observables in-process
(microseconds–milliseconds, no external subprocess). The light half of the workflow.
**Status**: **As-built** @ Operas bridge (2026-07-10).
**Design refs**: [`../DESIGN_OPERAS_BRIDGE_2.0.md`](../DESIGN_OPERAS_BRIDGE_2.0.md),
[`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §11;
[worker.md](worker.md), [module_base.md](module_base.md).
**Reuses V1**: none by import. Operator catalog reuses the standalone **Jarvis-Operas** package.

> **As-built:** `operator` resolves via **importlib first**, then the Jarvis-Operas registry.
> Upgrading Operas can add catalog operators without a HEP release. Optional extra:
> `pip install 'jarvishep2[operas]'`.

---

## 1. Class defined — `OperasModule`

Lightweight Operas executor with Worker-side preload.

**Attributes** (from `__init__`):

| Attribute | Type | Meaning |
|-----------|------|---------|
| `name`, `config` | str, dict | module name + raw config |
| `operator` | str | registry name (`helper.eggbox2d`) or dotted import path |
| `input`, `output` | list | input/output observable specs |
| `kwargs` | dict | extra call kwargs |
| `call_mode` | str | `call` (sync) or `acall` (async) |
| `timeout_sec` | float\|None | bounded call |
| `_func` | Callable\|None | cached operator (after `preload`) |
| `_parse_locals`, `_numeric_modules` | dict | sympy symbols + numeric backend for input expressions |

**Member functions:**

| Method | Behavior |
|--------|----------|
| `_normalize_timeout` (`@staticmethod`) | `>0` float or None. |
| `_build_expression_context` (`@staticmethod`) | sympy symbols (`x,y,z,shift,LogL`) + numeric modules (`sin/cos/exp/log`). |
| `preload()` | `resolve_operator(operator, call_mode)` once per Worker. |
| `_build_input_observables(observables)` | assemble inputs: pass-through names, `entry` lookups, or `expression` evaluated via `sympify`+`lambdify`. |
| `_run_coro` / `_run_with_timeout` | async + timeout helpers. |
| `execute(observables, sample_info) -> dict` | build inputs, call operator, map declared outputs; require dict output. |

---

## 2. Module-level functions

| Function | Behavior |
|----------|----------|
| `resolve_operator(path, call_mode=…)` | importlib → Jarvis-Operas registry; raise `ValueError` if neither works. |
| `_resolve_dotted_callable(path)` | import a dotted callable. |
| `_try_jarvis_operas_registry` / `_wrap_operas_registry_operator` | optional Operas integration. |
| `_resolve_entry(data, entry)` | dotted lookup into a mapping. |
| `preload_operas(modules) -> dict[str,OperasModule]` | build + preload all configured opera modules. |

---

## 3. Execution within the Worker

```
Worker._init_runtime: self._operas = preload_operas(opera_configs)   # resolve + cache once
Worker._run_opera_step(step, sample):
    updated = op.execute(sample.observables, sample.info)
    sample.observables.update(updated)
```

---

## 4. Concurrency / determinism / failure semantics

- Preload once per Worker.
- Operator treated as pure (observables → observables).
- Unknown operator hard-fails at preload with install / path hints.
- Hung operator with timeout → `TimeoutError` fails the Sample, not the Worker.

---

## 5. Interfaces / collaborators

- **Worker** → `preload_operas` + `execute`.
- **Jarvis-Operas** (optional) → global registry for catalog operators.
- **testing.eggbox** — importlib-path fixture used by parity YAMLs.

---

## 6. Tests

- `tests/test_operas_bridge.py` — registry + importlib resolution.
- Existing opera parity suites keep importlib paths green.
