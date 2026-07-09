# DESIGN — Jarvis-Operas Operator Bridge (V2)

**Status**: approved for implementation  
**Date**: 2026-07-10  
**Scope**: resolve `Operas.Modules[].operator` through **Jarvis-Operas** when possible, so new operators ship by upgrading Operas without HEP code changes. Keep importlib dotted callables as the escape hatch.

---

## 1. Problem

V2 `OperasModule` only does `importlib` dotted paths:

```yaml
operator: jarvishep2.testing.eggbox.eggbox2d_numpy
```

The catalog of physics/helper operators lives in **Jarvis-Operas** (`helper.eggbox2d`, `math.add`, DM limits, …). V1 already loads that registry; V2 does not. Adding an operator today requires either shipping code inside HEP or forcing users to write full Python import paths.

## 2. Goals

1. **Prefer Jarvis-Operas registry** for operator names it knows (`helper.eggbox2d`, …).
2. **Fall back to importlib** for package paths (`jarvishep2.testing.eggbox.eggbox2d_numpy`).
3. **Upgrade Operas ⇒ new operators** usable from YAML without HEP release (when names resolve in the installed Operas package).
4. Preserve call_mode `call` / `acall`, timeout, input expressions, output mapping.
5. Optional dependency: pure importlib workflows still run without Operas installed.

### Non-goals

- Move OperasModule execution into the Operas package.
- Port V1 `Operas.Functions` whitelist into likelihood funcs (later).
- Change YAML schema for Modules.

---

## 3. Resolution order

```
operator string
    │
    ├─1─ try importlib dotted callable (fast; no Operas bootstrap)
    │
    ├─2─ else if jarvis_operas importable AND registry.resolve_name(operator) succeeds
    │       → wrap registry.call / registry.acall
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
| `_wrap_operas_registry_call(name, call_mode)` | `**kwargs` → `registry.call` / async `registry.acall` |
| `_resolve_dotted_callable` | existing importlib helper |

`OperasModule.preload()` uses `resolve_operator` instead of only importlib.

### 4.2 Dependency

```toml
[project.optional-dependencies]
operas = ["Jarvis-Operas>=1.3.0"]
```

Core remains installable without Operas. When YAML uses a registry name and Operas is missing → clear `ImportError` / `ValueError` at preload.

### 4.3 Call contract (unchanged)

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
| importlib path works | use importlib (even if Operas installed) **only if Operas resolve failed** |

If both exist for the same string (unlikely), **Operas wins** — document that HEP-local test operators should keep a unique package path (`jarvishep2.testing…`).

---

## 6. Acceptance

1. Design doc + `components/operas.md` updated.
2. `operator: helper.eggbox2d` executes via Jarvis-Operas (unit test).
3. Existing `jarvishep2.testing.eggbox.eggbox2d_numpy` fixtures still pass (importlib path).
4. Unknown operator hard-fails with actionable message.
5. Full suite green with Operas installed in the test env.

---

## 7. Follow-ups

- Likelihood `Operas.Functions` alias registration (V1 parity).
- Worker-side optional preload of the global registry once per process (already cached inside Operas).
