# DESIGN — Jarvis-Operas Operator Bridge (V2)

**Status**: **operator bridge + dynamic expression functions implemented; Eggbox Bridson V1-YAML gate passed**; remaining strict-call/UI work is D11.4
**Date**: 2026-07-10 · as-built reviews 2026-07-13
**Scope**: resolve `Operas.Modules[].operator` through **Jarvis-Operas** when possible, so new operators ship by upgrading Operas without HEP code changes. Keep importlib dotted callables as the escape hatch.

---

## 1. Problem

V2 `OperasModule` only does `importlib` dotted paths:

```yaml
operator: jarvishep2.testing.eggbox.eggbox2d_numpy
```

The catalog of physics/helper operators lives in **Jarvis-Operas** (`helper.eggbox2d`, `math.add`, DM limits, …). V1 already loads that registry; V2 does not. Adding an operator today requires either shipping code inside HEP or forcing users to write full Python import paths.

## 2. Goals

1. **Prefer importlib** for valid dotted package callables.
2. **Fall back to Jarvis-Operas registry** for catalog names (`helper.eggbox2d`, …).
3. **Upgrade Operas ⇒ new operators** usable from YAML without HEP release (when names resolve in the installed Operas package).
4. Preserve call_mode `call` / `acall`, timeout, input expressions, output mapping.
5. Optional dependency: pure importlib workflows still run without Operas installed.
6. Keep the hot path numerical: compile small input expressions once per Worker through the
   shared `ExpressionContext`, not once per Sample.
7. Discover persisted and entry-point user functions at process load and expose their qualified
   names to every expression consumer without changing the V1 YAML shape.

### Non-goals

- Move OperasModule execution into the Operas package.
- Change YAML schema for Modules.

---

## 3. Resolution order

```
operator string
    │
    ├─1─ try importlib dotted callable (fast; no Operas bootstrap)
    │
    ├─2─ else if jarvis_operas importable AND registry.resolve_name(operator) succeeds
    │       → snapshot declaration.numpy_impl in the Worker
    │
    └─3─ else raise ValueError with both paths explained
```

**Why importlib first:** Worker preload of local operators (e.g. `jarvishep2.testing…`) must not
pay the cost of bootstrapping the full Operas catalog. Catalog names like `helper.eggbox2d`
fail importlib quickly (`ModuleNotFoundError`) and then resolve via the registry.

---

## 4. HEP surface

### 4.1 `jarvishep2/operas.py` (extend)

| API | Role |
|-----|------|
| `resolve_operator(operator, *, call_mode) -> Callable` | Resolution order above; cache result on module |
| `_snapshot_operas_registry_operator(name)` | resolve `numpy_impl` once; Worker calls the local function thereafter |
| `_resolve_dotted_callable` | existing importlib helper |
| `_compile_input_expressions()` | compile and cache every `{name, expression}` as a shared `CompiledExpression` once per Worker |

`OperasModule.preload()` first compiles declared input expressions, then uses
`resolve_operator` instead of only importlib. Both phases are idempotent within the Worker;
`execute()` only evaluates the cached callables and calls the cached operator implementation.

### 4.2 Dependency

```toml
[project.optional-dependencies]
operas = ["Jarvis-Operas>=1.3.0"]
```

Core remains installable without Operas. When YAML uses a registry name and Operas is missing → clear `ImportError` / `ValueError` at preload.

### 4.3 Dynamic expression-function snapshot

`jarvishep2/operas_functions.py` demand-loads the registry when an expression contains
`namespace.function(...)` or an `Operas.Modules[].operator` needs registry resolution. Entry points are discovered once
per process registry. The resulting parse-local and NumPy tables are snapshotted into one
Worker-owned `ExpressionContext`, shared by Operas inputs, Calculator/Portal Dump expressions,
and Likelihood. Control-process sampler expressions use the same builder.

No registry lookup occurs during repeated expression evaluation. See
[`OPERAS_DYNAMIC_FUNCTION_DISCOVERY_2026-07-13.md`](archive/reviews/OPERAS_DYNAMIC_FUNCTION_DISCOVERY_2026-07-13.md).

### 4.4 Call contract (unchanged)

`OperasModule.execute` still builds kwargs:

```python
call_kwargs["observables"] = input_observables
for k, v in input_observables.items():
    call_kwargs.setdefault(k, v)
call_kwargs.update(self.kwargs)
```

Jarvis-Operas `helper.eggbox2d` expects `x=…, y=…` kwargs — satisfied by the flatten step.

Output must remain a mapping; scalar returns are not auto-wrapped (callers declare `output` entries).

---

## 5. Failure semantics

| Case | Behavior |
|------|----------|
| Operas name found | use registry |
| Operas not installed, name only in Operas | error: install Jarvis-Operas |
| Operas installed, unknown name + bad import path | error listing tried paths |
| importlib path works | use importlib immediately; the registry is not bootstrapped |

If both paths can resolve the same string, **importlib wins**. This matches §3 and the
as-built implementation; it also keeps local Worker preload cheap.

---

## 6. Acceptance

1. Design doc + `components/operas.md` updated.
2. `operator: helper.eggbox2d` executes via Jarvis-Operas (unit test).
3. Existing `jarvishep2.testing.eggbox.eggbox2d_numpy` fixtures still pass (importlib path).
4. Unknown operator hard-fails with actionable message.
5. Full suite green with Operas installed in the test env.
6. Repeated execution of the same module does not call `lambdify` again after `preload()`.

As-built verification on 2026-07-13 used the local `Jarvis-Operas 1.3.8` worktree; a real
`helper.eggbox2d` registry call succeeded. The unmodified
`Jarvis-Examples/Eggbox/bin/Example_Bridson_Operas.yaml` then completed a real Redis-backed
run: 10,034 submitted/finished, zero failed, and exact recomputation parity for both `z` and
`LogGauss`. V1 expression constants `Pi`/`pi`/`PI` are accepted by the V2 Operas input mapper.
Operas input expressions are compiled once per Worker and reused for all Samples, matching the
Likelihood lifecycle. Both now use the same object model and standard function namespace as
Calculator/Portal, Selection, and AdaptiveLevelSet expressions.
See [`EGGBOX_BRIDSON_OPERAS_ACCEPTANCE_2026-07-13.md`](archive/reviews/EGGBOX_BRIDSON_OPERAS_ACCEPTANCE_2026-07-13.md).

---

## 7. Follow-ups

- Strictly reject `call_mode` values outside `call`/`acall` (current code silently treats unknown
  values as sync).
- Align signature filtering and sample-logger context with the V1/registry execution contract.
- Add thin `Jarvis2 operas list/info` discovery without duplicating the registry.
