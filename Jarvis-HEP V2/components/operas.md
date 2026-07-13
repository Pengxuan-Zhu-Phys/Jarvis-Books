# Component — OperasModule (`jarvishep2/operas.py`)

**Role**: the **in-process** module backend — a Python operator that computes observables in-process
(microseconds–milliseconds, no external subprocess). The light half of the workflow.
**Status**: **As-built** @ `jarvis2` `0a5e85e` (Operas bridge + ExpressionContext + dynamic discovery).
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
| `_expression_context` | ExpressionContext | Worker-owned shared-language/compiler cache injected into the module |
| `_compiled_input_expressions` | dict | expression index → `CompiledExpression`, compiled once during Worker preload |
| `_input_expressions_compiled` | bool | idempotence guard for expression preload |

**Member functions:**

| Method | Behavior |
|--------|----------|
| `_normalize_timeout` (`@staticmethod`) | `>0` float or None. |
| `preload()` | compile every declared input expression, then `resolve_operator(operator, call_mode)`; both once per Worker. |
| `_compile_input_expressions()` | ask `ExpressionContext` to compile each `{name, expression}` exactly once and cache its `CompiledExpression`. |
| `_build_input_observables(observables)` | assemble inputs: pass-through names, `entry` lookups, or evaluate an already-compiled expression callable. |
| `_run_coro` / `_run_with_timeout` | async + timeout helpers. |
| `execute(observables, sample_info) -> dict` | build inputs, call operator, map declared outputs; require dict output. |

---

## 2. Module-level functions

| Function | Behavior |
|----------|----------|
| `resolve_operator(path, call_mode=…)` | importlib → Jarvis-Operas registry; raise `ValueError` if neither works. |
| `_resolve_dotted_callable(path)` | import a dotted callable. |
| `_try_jarvis_operas_registry` / `_snapshot_operas_registry_operator` | optional Operas integration; resolve `numpy_impl` once at preload. |
| `_resolve_entry(data, entry)` | dotted lookup into a mapping. |
| `preload_operas(modules, expression_context=…) -> dict[str,OperasModule]` | build + preload all configured opera modules with the Worker context. |

---

## 3. Execution within the Worker

```
Worker._init_runtime:
    expression_context = build_operas_expression_context(...)  # demand-gated
    self._operas = preload_operas(opera_configs, expression_context=expression_context)
        # per Worker: compile input expressions once + resolve operator once
Worker._run_opera_step(step, sample):
    # per Sample: numeric expression calls + cached operator call only
    updated = op.execute(sample.observables, sample.info)
    sample.observables.update(updated)
```

The Likelihood module follows the same lifecycle: its SymPy expressions are compiled once in
the Worker runtime and then reused. Small Operas expressions therefore no longer pay
`sympify`/`lambdify` cost for every Sample.

Operas now uses the same complete 38-function V1 Expression Core as Likelihood, Calculator
inputs, Selection, and AdaptiveLevelSet; see [expression.md](expression.md) for the catalog.
Registered Operas functions are included at startup by their qualified names. The numerical
implementations are snapshotted, so expression evaluation does not dispatch through the registry.

---

## 4. Concurrency / determinism / failure semantics

- Operator and input-expression preload once per Worker.
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

- `tests/test_operas_bridge.py` — registry + importlib resolution, V1 `Pi` expressions, and a
  compile-count invariant proving repeated Samples reuse cached input callables.
- Existing opera parity suites keep importlib paths green.
