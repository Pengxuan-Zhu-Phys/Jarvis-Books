# Component — Parameters & Variables (`jarvishep2/Module/parameters.py`, `jarvishep2/Sampling/variables.py`)

**Role**: the parameter-space definition layer. `Variable` defines one scanned parameter (name,
distribution, bounds); `Parameters` is the module that turns a Sample's params into the
observables/inputs the workflow consumes. Together they back the `u → x` mapping and the input
files.
**Status**: design — plan WP-D0.1, reused with light V2 adaptation.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §3; [umapper.md](umapper.md)
(consumes the distribution math), [sampler.md](sampler.md), [io_system.md](io_system.md).
**Reuses V1**: `Sampling/variables.py` (`Variable`, `map_standard_random_to_distribution`) and
`Module/parameters.py` (`Parameters(Module)`). Distribution math is unchanged (parity); the V2
work is making them feed the [UMapper](umapper.md) and stay picklable.

---

## 1. Responsibilities

1. **`Variable`**: define one parameter — name, description, distribution kind, bounds/params —
   and map a standard-uniform draw to the physical value (`map_standard_random_to_distribution`).
2. **`Parameters`**: the workflow's first "module" — expose the Sample's parameters as
   observables/inputs (so downstream calculators/operas can read them), assign IDs, attach
   nuisance parameters.
3. Provide the variable **schema + order** that the [UMapper](umapper.md) consolidates for `u → x`.

---

## 2. Structure (reused)

```python
class Variable:
    name; description; distribution; parameters
    def generate(self): ...                                   # sample a value
    def map_standard_random_to_distribution(self, std_rand): ...   # u_i → x_i (the parity-critical map)

class Parameters(Module):
    def __init__(self, name, parameter_definitions=None): ...
    def add_nuisance(self, nuisances): ...
    def analyze_ios(self): ...
    def generate_parameters(self): ...
    def execute(self, observables, sample_info) -> dict: ...   # unified Module signature
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `Variable.map_standard_random_to_distribution` | `(std_rand) -> float` | `u_i ∈ [0,1] → x_i` per distribution (flat/log/normal/custom). The function the [UMapper](umapper.md) reuses. |
| `Variable.generate` | `() -> float` | Draw a value (used by samplers that need a direct draw). |
| `Parameters.generate_parameters` | `() -> dict` | Build the param dict for a Sample. |
| `Parameters.add_nuisance` | `(nuisances) -> None` | Attach nuisance parameters (see [nuisance.md](nuisance.md)). |
| `Parameters.analyze_ios` | `() -> None` | Resolve the parameter module's input/output specs. |
| `Parameters.execute` | `(observables, sample_info) -> dict` | Expose params as observables (the workflow's entry layer); unified `Module` signature. |

---

## 4. Relationship to UMapper

```
variables.py  Variable.map_standard_random_to_distribution   ← the math (per variable)
umapper.py    UMapper consolidates all Variables, fixes u-index order, maps u_coords → params
parameters.py Parameters.execute exposes params as observables for the workflow
```

Edge guards (carried bug fix): the `u=0`/`u=1` log/logit boundary (`variables.py` → `-inf`) is
fixed in the UMapper/Variable map with a test (invariant #14, see [umapper.md](umapper.md) §3).

---

## 5. Concurrency / determinism / failure semantics

- `Variable` maps are **pure + deterministic** → identical to V1 for the same `u` (golden parity).
- `Variable`/`Parameters` must be **picklable from config** (no closures) so Workers build the same
  parameter space (spawn).
- `Parameters.execute` runs as the workflow's first layer in the Worker (params → observables),
  uniform `Module` contract.
- Custom distributions ship as importable refs (picklable), not lambdas.

---

## 6. Interfaces

- **UMapper** → reuses `map_standard_random_to_distribution`; owns u-index order.
- **Sampler** → reads variable schema (`dim`, bounds) for proposals; does **not** map (Worker does).
- **Parameters module** → first workflow layer (params → observables).
- **nuisance** → `add_nuisance`.
- **io_system / expression** → param values feed input files and formulas.

---

## 7. Tests (`tests/test_parameters_variables.py`)

Unit:
1. **Map parity** — `map_standard_random_to_distribution` equals V1 for flat/log/normal/custom on
   a grid of `u` (golden).
2. **Boundary guard** — `u=0`/`u=1` produce finite values (fix for the `-inf` bug).
3. **Parameters.execute** — exposes params as observables; unified `Module` signature; parity vs
   V1.
4. **Picklability** — `Variable`/`Parameters` rebuild from config under spawn; custom dists are
   importable refs.
5. **Nuisance attach** — `add_nuisance` wires nuisance params into the parameter set.

Verification logic: test 1 is the science-parity foundation (same draws ⇒ same x); test 2 closes
the boundary `-inf` bug.

---

## 8. Open questions

- Whether `Parameters` stays a `Module` or becomes a plain pre-step of the Worker (default: keep as
  a Module for uniformity).
- Nuisance parameters: separate mapper vs. shared (defer with nuisance-in-Worker, [nuisance.md](nuisance.md)).
