# Component — UMapper / MapperPipeline (`jarvishep2/mapper.py`)

**Role**: single `u → physical params` path for both the control process (`selection`) and
Workers (`Sample.bind_params`). Distribution layer first, then optional
`Sampling.Mapper` expressions.
**Status**: **As-built** (D22). Design:
[`../DESIGN_SAMPLING_MAPPER_2.0.md`](../DESIGN_SAMPLING_MAPPER_2.0.md).
**Reuses V1**: distribution math in
[`Sampling/variables.py`](parameters_variables.md)
(`Variable.map_standard_random_to_distribution`).

---

## 1. Module

`jarvishep2/mapper.py`. Primary exports:

| Symbol | Role |
|--------|------|
| `MapperPipeline` | Sole production mapper: `from_config` / `from_spec` / `map` |
| `MapperSpec` | Picklable pure-data spec (variables + ordered expressions) |
| `build_mapper_spec_from_config` | Card → `MapperSpec` (validation + DAG) |
| `build_mapper` | Worker factory from picklable config (`type` + optional pipeline payload) |
| `DistributionUMapper` / `FlatUMapper` / `IdentityParamMapper` | Internal / test helpers |

All mappers satisfy [`Sample.UMapperProtocol`](sample.md) (`map(u_coords) -> Mapping`).

---

## 2. YAML surface

Optional under `Sampling` — **flat** name → expression (no nested `derive`):

```yaml
Sampling:
  Mapper:
    x: "cos(t)"
    y: "sin(t)"
```

Omitted → distribution-only pipeline (backward compatible). Top-level `Mapper:` remains
rejected. Full rules: design doc §4–§5; reference [`YAML_REFERENCE_2.0.md`](../YAML_REFERENCE_2.0.md) §7.

Internal `MapperSpec` still uses fields named `derive_order` / `derive_exprs` — those are
code identifiers only, not YAML keys.

---

## 3. Classes

### 3.1 `MapperPipeline` (production)

1. Map `u` through each `Variables[].distribution` → sampling variable values.
2. Evaluate Mapper expressions in topological order → merge into the same dict.
3. `len(u) == len(Variables)` strictly (M5).

Control samplers call `MapperPipeline.from_config(config)` once in `set_config`.
Workers receive picklable `MapperSpec` via `worker_config` and rebuild with
`MapperPipeline.from_spec`.

### 3.2 Legacy helpers

- **`DistributionUMapper`** — distribution-only (pipeline’s first stage).
- **`FlatUMapper`** — min/max linear map; YAML-unreachable, tests only.
- **`IdentityParamMapper`** — pass-through keys; fallback when no Variables.

---

## 4. Determinism / resume

- `map(u)` is pure: closed expression namespace (no observables / RNG / I/O).
- Control and Worker share one implementation (M2).
- Checkpoint `mapper_hash` fingerprints Mapper + variable names; drift refuses `--resume`.

---

## 5. Collaborators

- **Sample.bind_params** — Worker applies pipeline after dequeue.
- **Samplers** (`randoms`, `grid`, `bridson`, `adaptive_bridson`, `mcmc`, `dynesty`,
  `fixed_set`) — control-side `selection` uses the same pipeline.
- **plot_scene** — axis preference prefers Mapper key write order.
- **worker_config._default_mapper** — emits pipeline / distribution / none / identity.

---

## 6. Tests

`tests/test_sampling_mapper.py` (D22 unit + closed-namespace + fingerprint).
Regression via worker / sampler suites that bind params through `build_mapper`.
