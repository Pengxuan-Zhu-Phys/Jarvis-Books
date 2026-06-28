# Component — UMapper (`jarvishep2/mapping.py`)

**Role**: the global `u → x` mapper. Turns a Sampler's normalized draw (`u_coords ∈ [0,1]^d`)
into physical parameters (`x`) according to the variable/distribution definitions. Held by each
Worker; never on the Sampler hot path.
**Status**: design — plan WP-D0.1 (with Sample) → D1.1 (Worker holds it).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §3, §4;
discussion `worker_design.md` §2/§3.1.
**Reuses V1**: the distribution math in `jarvishep/Sampling/variables.py`
(`map_*_to_distribution`) and the per-sampler `map_point_into_distribution`
(`randoms.py`/`bridson.py`). UMapper **consolidates** these into one reusable object.

---

## 1. Responsibilities

1. Load the variable schema (names, ranges, distribution types, flat/log/custom) once.
2. Map `u_coords → params` deterministically and **identically to V1** (parity).
3. Be **picklable/rebuildable under spawn** (config in, no live closures), so each Worker can
   own one instance.
4. Expose the inverse / bookkeeping needed by samplers that report in `x`-space.

Why a dedicated object: in V1 the mapping is split across the sampler classes and runs in the
control process (a bottleneck in process mode — the "worker-owned sample lifecycle" direction).
V2 moves it into the Worker as one shared mapper.

---

## 2. Structure

```python
class UMapper:
    def __init__(self, variables: list[VariableSpec]): ...
    variables: list[VariableSpec]          # name, kind, bounds, params
    _order: list[str]                       # stable u-index → name order

@dataclass
class VariableSpec:
    name: str
    kind: str            # flat | log | normal | custom | ...
    low: float
    high: float
    extra: dict = field(default_factory=dict)
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `from_config` | `(cls, variable_yaml) -> UMapper` | Build from the YAML `Sampling.Variables` block; fix `_order`. |
| `map` | `(u_coords: np.ndarray) -> dict[str, float]` | `u → x` per variable kind; returns `{name: value}`. Deterministic. |
| `map_array` | `(u_coords) -> np.ndarray` | Same in array form (x-vector in `_order`). |
| `inverse` | `(params: dict) -> np.ndarray` | `x → u` (for samplers/diagnostics that need it). |
| `dim` | `() -> int` | Number of variables. |
| `signature` | `() -> dict` | Stable hash of the variable schema (checkpoint integrity, parity key). |

Edge cases carried from the V1 bug list: guard `u=0`/`u=1` for log/logit transforms
(`variables.py:48` `np.log(p/(1-p))` → `-inf` at `p=0`) — fix with a test in this WP
(invariant #14).

---

## 4. Determinism / parity

- `_order` is fixed at construction (stable u-index ↔ variable mapping), so `u_coords[i]`
  always maps to the same variable — essential for seeded parity and checkpoint resume.
- `map(u)` reproduces the V1 `map_point_into_distribution` output exactly for the same `u`
  (golden test). This is what makes "same draws ⇒ same science" hold after moving mapping into
  the Worker.

---

## 5. Concurrency / lifecycle

- **Immutable after construction** → safe to share read-only; each Worker builds its own from
  config at spawn (no cross-process sharing of the object).
- No file or network I/O — pure numeric, cheap, thread-safe.

---

## 6. Interfaces

- **Sample.bind_params(mapper)** calls `mapper.map(self.u_coords)`.
- **Worker._init_mapper** builds it from the shipped config.
- **Sampler** does **not** call it (mapping left to the Worker), but uses `dim()`/`signature()`
  for proposal bookkeeping and checkpoint integrity.

---

## 7. Tests (`tests/test_umapper.py`)

Unit:
1. **V1 parity** — for a grid of `u` values, `UMapper.map(u)` equals the V1
   `map_point_into_distribution` output (allclose) for flat/log/normal/custom kinds.
2. **Order stability** — `_order` is deterministic; `u_coords[i]` ↔ same variable across builds.
3. **Round-trip** — `inverse(map(u)) ≈ u` within tolerance for invertible kinds.
4. **Edge guards** — `u=0`/`u=1` do not produce `-inf`/`nan` (fixes `variables.py:48`); a
   regression test pins the boundary behavior.
5. **Picklability** — builds under `spawn` from config; no closures captured.
6. **Signature stability** — `signature()` changes iff the variable schema changes (checkpoint
   integrity).

Verification logic: test 1 is the parity gate that lets the mapping move off the control
process without changing results; test 4 closes a known V1 silent-`-inf` bug.

---

## 8. Open questions

- Custom-distribution callables: ship as importable refs (picklable) rather than lambdas.
- Whether nuisance parameters get their own mapper or share this one (defer with the
  nuisance-in-Worker work).
