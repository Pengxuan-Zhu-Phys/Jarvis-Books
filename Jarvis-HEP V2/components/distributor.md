# Component — Distributor (sampler dispatch) (`jarvishep2/distributor.py`)

**Role**: select the sampler implementation by name and declare resume support. The single place
that maps a YAML `Sampling.Method` string to a concrete sampler class.
**Status**: **As-built** @ registry rewrite (2026-07-10).
**Design refs**: [`../DESIGN_DISTRIBUTOR_REGISTRY_2.0.md`](../DESIGN_DISTRIBUTOR_REGISTRY_2.0.md),
[`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §11;
[sampler.md](sampler.md), [samplers_catalog.md](samplers_catalog.md).
**Reuses V1**: none by import.

> **As-built:** hard-coded `match`/`case` is gone. Samplers register via
> `Distributor.register(name, factory, stateless=…, resume=…)`. Built-ins include the four
> coverage methods **plus** AdaptiveBridson, the MCMC family, Dynesty, and MultiNest.
> **D25.1** makes this table the single catalog (manifest / contracts / mapper / man must
> not keep private copies of the 17 names).

---

## 1. Module surface

| Symbol | Behavior |
|--------|----------|
| `SamplerRegistration` | `name`, `factory`, `stateless`, `resume` |
| `STATELESS_METHODS` | live view (`in` / iterate) derived from the registry |
| `register_builtin_samplers()` | installs the four V2 methods (called at import) |
| `Distributor.register(...)` | add/override a factory |
| `Distributor.unregister` / `clear_registry_for_tests` | test cleanup |
| `Distributor.available_methods()` | sorted names |
| `Distributor.get_resume_status(method)` | status or `"intentionally unsupported"` |
| `Distributor.set_method(method)` | lookup + factory(); unknown → `NotImplementedError` with Available list |
| `Distributor.RESUME_SUPPORT_STATUS` | dict mirror rebuilt on register |

---

## 2. Lifecycle / failure semantics

Control-process only, one-shot at boot. Lazy factories keep sampler modules out of the import
graph until selected. Unknown `Sampling.Method` fails at boot with the registered list.

---

## 3. Interfaces / collaborators

- **core.init_sampler_from_config** / **core.run** → `set_method` + `STATELESS_METHODS`.
- **samplers** are dispatch targets via factories in `register_builtin_samplers`.

---

## 4. Tests

`tests/test_samplers_catalog.py` — resolve four builtins, unknown method lists Available,
register/unregister `UnitDummy` without editing `set_method`.
