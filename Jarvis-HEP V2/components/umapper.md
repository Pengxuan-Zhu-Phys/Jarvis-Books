# Component — UMapper (`jarvishep2/mapper.py`)

**Role**: the Worker-held `u → x` mapper. Turns a Sampler's normalized draw (`u_coords ∈ [0,1]^d`)
into physical parameters (`x`) per the variable/distribution definitions. Held by each Worker;
never on the Sampler hot path.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `mapper.py` 113 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §3, §4.
**Reuses V1**: none by import — the distribution math lives in
[`Sampling/variables.py`](parameters_variables.md) (`Variable.map_standard_random_to_distribution`),
which `DistributionUMapper` delegates to.

---

## 1. Module & line count

`jarvishep2/mapper.py` (113 lines). Exports `DistributionUMapper`, `FlatUMapper`,
`IdentityParamMapper`, `build_mapper`. All satisfy
[`Sample.UMapperProtocol`](sample.md) (a single `map(u_coords) -> Mapping`).

> **As-built drift:** the design proposed a single `UMapper` + `VariableSpec` with
> `from_config/map/map_array/inverse/dim/signature`. **Shipped: three small mapper classes, each
> with only `map()`, plus a `build_mapper` factory.** There is no `inverse`/`map_array`/
> `signature`/`dim` and no file named `mapping.py`.

---

## 2. Classes defined

### 2.1 `DistributionUMapper`
Map `u_coords` using V1-compatible `Variable` distributions.
- **Attributes:** `_variables: list[Variable]` (built from config mappings; each carries name,
  description, distribution type, parameters).
- **`__init__(variables: Sequence[Mapping])`** — for each variable dict, read
  `distribution.type` / `distribution.parameters` and construct a `Variable`.
- **`map(u_coords) -> dict[str, float]`** — reshape to 1-D, require `len ≥ #variables`, map each
  coordinate via `Variable.map_standard_random_to_distribution`. Raises `ValueError` if too short.

### 2.2 `FlatUMapper`
Map normalized coords to flat-distributed parameters using each variable's `min`/`max`.
- **Attributes:** `_names: list[str]`, `_bounds: list[tuple[float,float]]`.
- **`map(u_coords) -> dict[str, float]`** — `lo + u*(hi-lo)` per variable.

### 2.3 `IdentityParamMapper`
Test helper that passes coords straight through.
- **Attributes:** `_keys: tuple[str, ...]`.
- **`map(u_coords) -> dict[str, float]`** — `{key: coord}` per configured key, or `{"u": coord0}`
  when no keys.

---

## 3. Module-level functions

| Function | Behavior |
|----------|----------|
| `build_mapper(config) -> UMapperProtocol \| None` | Factory by `config["type"]`: `none`→None, `identity`→`IdentityParamMapper(keys)`, `distribution`→`DistributionUMapper(variables)`, default/`flat`→`FlatUMapper(variables)`. Returns `None` when no variables. |

---

## 4. Determinism / parity

- `map(u)` is pure and deterministic; coordinate `u_coords[i]` always maps to the i-th configured
  variable (construction order is the stable u-index ↔ variable mapping).
- Built from **picklable config** at Worker spawn (no closures), so every Worker maps identically.

---

## 5. Interfaces / collaborators

- **Sample.bind_params(mapper)** ([sample.md](sample.md)) calls `mapper.map(self.u_coords)`.
- **Worker** ([worker.md](worker.md)) builds the mapper from `worker_config["mapper"]` via
  `build_mapper`; default mapper config is produced by
  [`worker_config._default_mapper`](config_schema.md).
- **Variable** ([parameters_variables.md](parameters_variables.md)) supplies the distribution math.

---

## 6. Tests

Exercised via the mapper path in `tests/test_worker_calculator.py`,
`tests/test_worker_mvp.py`, `tests/test_samplers_catalog.py`, and the distributed acceptance
suite (Sample params bound through `build_mapper`).
