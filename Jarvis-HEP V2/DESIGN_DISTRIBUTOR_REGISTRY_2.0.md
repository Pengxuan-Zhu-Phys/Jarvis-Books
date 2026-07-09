# DESIGN — Distributor Sampler Registry (D9.2 remainder)

**Status**: approved for implementation  
**Milestone**: D9.2 (Distributor half)  
**Date**: 2026-07-10  
**Scope**: replace `Distributor.set_method` hard-coded `match` with a register table so new samplers do not require editing three parallel lists.

---

## 1. Problem

Today (`distributor.py`):

```python
STATELESS_METHODS = frozenset({"Bridson", "Random", "Grid", "CSV"})
RESUME_SUPPORT_STATUS = {…: "implemented"}
set_method: match/case → lazy import each class
```

Adding a sampler means editing **three** places. That blocks the MCMC/nested family roadmap.

## 2. Goals

1. Single registration entry per sampler: name, factory, `stateless`, `resume` status.
2. `set_method(name)` looks up the table; unknown → `NotImplementedError` listing available methods.
3. `STATELESS_METHODS` and resume map are **derived** from the registry (no parallel lists).
4. Behavior-preserving for existing Bridson/Random/Grid/CSV.
5. Acceptance test: register a dummy sampler without editing `set_method` body.

### Non-goals

- Port MCMC/Dynesty/MultiNest implementations.
- Plugin entry points for third-party samplers (can add later; in-tree register is enough now).
- FixedSetSampler refactor (D9.7).

---

## 3. Design

### 3.1 Registration record

```python
@dataclass(frozen=True)
class SamplerRegistration:
    name: str
    factory: Callable[[], SamplingVirtial]
    stateless: bool = True
    resume: str = "implemented"  # implemented | intentionally unsupported | planned
```

### 3.2 API

| API | Role |
|-----|------|
| `Distributor.register(name, factory, *, stateless=True, resume="implemented", override=False)` | Classmethod or module function used at import for built-ins |
| `Distributor.set_method(method)` | Lookup + call factory |
| `Distributor.available_methods()` | Sorted registered names |
| `Distributor.get_resume_status(method)` | From registration, default `"intentionally unsupported"` |
| `STATELESS_METHODS` | Property or function `stateless_methods()` derived from table |

Keep module-level `STATELESS_METHODS` as a **dynamic view** for `core.py` compatibility:

```python
def stateless_methods() -> frozenset[str]:
    return frozenset(r.name for r in _REGISTRY.values() if r.stateless)

# Backward-compatible name used by core:
# either keep STATELESS_METHODS updated on register, or make core call stateless_methods().
```

**Decision**: maintain `STATELESS_METHODS` as a module-level frozenset rebuilt on each `register()` so existing `from distributor import STATELESS_METHODS` and `in STATELESS_METHODS` keep working without changing core.

### 3.3 Built-in registration

At end of `distributor.py` (or `_register_builtins()` called on import):

```python
Distributor.register("Bridson", lambda: Bridson(), stateless=True, resume="implemented")
…
```

Use lazy factories to avoid import-time cost:

```python
def _bridson():
    from jarvishep2.Sampling.bridson import Bridson
    return Bridson()
```

### 3.4 Decorator (optional sugar)

```python
@Distributor.register_class("Bridson", stateless=True, resume="implemented")
class Bridson(...): ...
```

**Out of this WP** if it forces sampler modules to import Distributor (cycle risk). Prefer explicit `register` in `distributor.py`.

---

## 4. Failure semantics

| Case | Behavior |
|------|----------|
| Unknown method | `NotImplementedError(f"Sampling.Method '{method}' is not implemented… Available: …")` |
| Duplicate register without override | `ValueError` |
| Empty name | `ValueError` |

---

## 5. Acceptance

1. Design doc present; `components/distributor.md` updated.
2. No `match`/`case` in `distributor.py`.
3. Bridson/Random/Grid/CSV still resolve; resume status `"implemented"`.
4. Test: register `"UnitDummy"` factory without modifying `set_method` source; `set_method("UnitDummy")` works; cleanup with override/reset.
5. Full suite green (or targeted sampler + core paths green).
6. Plan D9.2 marked **done**.

---

## 6. Implementation steps

1. Rewrite `distributor.py` with registry + builtins.
2. Extend `tests/test_samplers_catalog.py` (or `test_distributor_registry.py`).
3. Update docs + plan ledger.
4. Commit.
