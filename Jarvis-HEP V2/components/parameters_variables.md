# Component — Parameters & Variables (`jarvishep2/Sampling/variables.py`)

**Role**: the parameter-space definition layer. `Variable` defines one scanned parameter (name,
distribution, bounds) and maps a standard-uniform draw to the physical value — the math behind the
`u → x` mapping.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `Sampling/variables.py` 90 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §3;
[umapper.md](umapper.md), [sampler.md](sampler.md).
**Reuses V1**: none by import — V1-compatible distribution math reimplemented.

> **As-built drift:** the design referenced a `Module/parameters.py` `Parameters(Module)` class
> (`add_nuisance`, `analyze_ios`, `generate_parameters`, `execute`). **That module was not built.**
> Only `Sampling/variables.py` ships; params→observables is handled directly by
> [`Sample.bind_params`](sample.md) + the [UMapper](umapper.md).

---

## 1. Module & line count

`jarvishep2/Sampling/variables.py` (90 lines). Exports `Variable`, `load_variables`. Module
constant `_STANDARD_NORMAL = NormalDist()` for inverse-CDF transforms.

---

## 2. Classes defined

### `Variable`
One scanned parameter.
- **Attributes (private, exposed via `@property`):** `_name`, `_description`, `_distribution`,
  `_parameters` → read-only `name`, `description`, `distribution`, `parameters`.
- **`__init__(name, description, distribution, parameters)`**.
- **`map_standard_random_to_distribution(std_rand) -> float`** — map `u ∈ [0,1]` to a physical
  value by distribution type:

| `distribution` | Mapping |
|----------------|---------|
| `Flat` | `min + (max-min)·u` |
| `Log` | `exp(logmin + (logmax-logmin)·u)` |
| `Normal` | `mean + stddev·Φ⁻¹(u)` |
| `Log-Normal` | `exp(mean + stddev·Φ⁻¹(u))` |
| `Logit` | `(ln p − ln(1−p))·scale + location`, `p = clip(u)` |

  Unknown type → `ValueError`. `u` is clamped to `(tiny, 1−eps)` via `_unit_interval` for the
  inverse-CDF/logit cases (closes the V1 `log(0) → -inf` boundary bug).

---

## 3. Module-level functions

| Function | Behavior |
|----------|----------|
| `_unit_interval(value) -> float` | clamp to the open unit interval. |
| `load_variables(config) -> list[Variable]` | read `config["Sampling"]["Variables"]`, build a `Variable` per entry (name, `distribution.type`, `distribution.parameters`); raises if none defined. |

---

## 4. Determinism / parity

- `map_standard_random_to_distribution` is **pure + deterministic** → identical results for the
  same `u`.
- `Variable` is built from config (no closures) → picklable for spawn Workers.

---

## 5. Interfaces / collaborators

- **DistributionUMapper** ([umapper.md](umapper.md)) constructs `Variable`s and calls the map.
- **sampling_utils** ([expression.md](expression.md)) `map_u_to_physical` / `map_row_to_physical`
  / `row_to_u_coords` also use `Variable`.
- **Samplers** ([samplers_catalog.md](samplers_catalog.md)) call `load_variables` to size the
  parameter space.

---

## 6. Tests

Distribution mapping is exercised through `tests/test_samplers_catalog.py` and the worker/mapper
end-to-end tests (each sampler maps `u`/CSV rows to physical params via `Variable`).
