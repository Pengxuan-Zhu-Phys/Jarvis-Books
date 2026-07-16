# Component — Adaptive Level-Set Sampler (`jarvishep2/Sampling/adaptive_level_set.py`)

**Role**: a feedback-driven sampler (2 ≤ d ≤ 5) that traces a target level-set
(`f(observables) = target_value`) efficiently: a global low-discrepancy pass explores
`[0,1]^d`, then **crossing detection on a neighbor graph** over the evaluated points
identifies where the level-set passes, and refinement batches are spent **only** near those
crossings until the level-set is resolved to a preset precision. The neighbor graph is
**exact Delaunay adjacency for d ≤ 3** (the Bridson–Voronoi flagship) and an **approximate
kNN graph for d = 4–5** (the pragmatic "details reasonable, not perfect" regime).
**Status**: ✅ **Implemented** on `jarvis2` — `adaptive_level_set.py`,
`hep:feedback` channel, `Distributor` registration `AdaptiveLevelSet` (`stateless=False`).
Target expressions use shared `ExpressionContext`. YAML inventory: §6.9 of
[`YAML_REFERENCE_2.0.md`](../YAML_REFERENCE_2.0.md) and
[`YAML-Example/ADAPTIVE_LEVEL_SET.md`](../YAML-Example/ADAPTIVE_LEVEL_SET.md).
Milestone **D10**: D10.1–D10.5 **done** (2026-07-16: Hausdorff §9.1–9.8; d=3 sphere /
d=4 proximity shell / d=5 Sobol gen-0; resume-safe `run_adaptive`).
**Origin**: maintainer's Bridson + Voronoi adaptive-contour idea, elaborated in two external
drafts (2026-07-11: 2-D detailed design; same day: dimension-hybrid extension) and
**corrected here against the as-built runtime** — the deltas from those drafts are listed in
§2 and are binding.
**Design refs**: [sampler.md](sampler.md), [feedback_sampler.md](feedback_sampler.md),
[samplers_catalog.md](samplers_catalog.md), [checkpoint.md](checkpoint.md),
[distributor.md](distributor.md), [redis_queue.md](redis_queue.md),
[expression.md](expression.md),
[`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §3/§11.
**Related but distinct**: Jarvis-PLOT's `likelihood_report` (PLOT bridge) does *post-hoc*
Voronoi cell analysis of a finished scan; this sampler puts the same geometric idea *in the
proposal loop* — the two share intuition, not code.

---

## 1. Motivation

Grid / Random / Bridson spend most calculator calls far from the physics: for a CL boundary
(ΔLogL = 2.30 / 5.99), an exclusion edge, or any user-defined level-set, the information is
concentrated on a (d−1)-dimensional surface embedded in the d-dimensional box. With external
calculators at seconds–minutes per point (the slow regime V2 is built for), focusing
refinement on the crossing region typically saves **5–20×** calculator calls at equal
level-set quality in 2-D, with the gain growing with dimension (the surface-to-volume ratio
shrinks). Typical uses: 68%/95% CL boundaries, exclusion curves/surfaces, profile-likelihood
contours, any `f(params/observables) = c` level-set.

### 1.1 Dimension policy (binding)

| d | Neighbor graph | Generation-0 | Output | Fidelity target |
|---|---|---|---|---|
| **2** | Delaunay ridges (exact, via `scipy.spatial.Voronoi`) | `Bridson_sampling` | ordered polylines + crossing cloud | high-quality geometry (flagship) |
| **3** | Delaunay ridges (exact) | `Bridson_sampling` | crossing cloud (+ optional surface mesh, open question Q5) | high-quality point set |
| **4–5** | symmetric kNN graph (approximate, via `scipy.spatial.cKDTree`) | Bridson (d=4) / **Sobol QMC** (d=5) | crossing cloud + 2-D slice projections | **"details reasonable"**: denser sampling near the level-set, honest approximation — no full hypersurface reconstruction |
| **≥6** | — | — | — | rejected at `set_config` (`ValueError`); use plain Bridson/Random or a future MCMC |

One algorithm, two graph builders: crossing detection, refinement, convergence, generation
barriers, determinism, checkpointing and the feedback channel are **identical across modes**
(§4); only the edge-set construction (§4.3) and the finalize stage (§4.5) are
dimension-dependent. This is deliberately tighter than the second draft's "two modes with
different logic" — see correction C12.

---

## 2. Binding corrections vs the 2026-07-11 drafts

The drafts are good algorithm sketches but reference components that **do not exist** in the
as-built runtime and contain some real bugs. Implementations must follow this table, not the
drafts.

### 2.1 Against the 2-D detailed draft

| # | Draft said | As-built reality / correction |
|---|---|---|
| C1 | Active cells found by reading `points[v].f` for `v in voronoi.regions[i]` | **Bug.** `regions[i]` indexes into `voronoi.vertices` (geometric Voronoi vertices) — those carry no `f`; `f` lives on the input **sites**. Use **`voronoi.ridge_points`** (pairs of neighboring site indices, = Delaunay edges): a ridge `(i, j)` is *crossing* iff `(f_i − t)·(f_j − t) ≤ 0`. See §4.3 |
| C2 | `ExpressionContext.compile(...)` | Implemented through the shared runtime ([expression.md](expression.md)); target is a `CompiledExpression`, rebuilt from config on resume. Inherits the remaining A.6 undeclared-symbol collision caveat. |
| C3 | `UMapper.from_config(...)`, `self._umapper.map` + inverse | No `UMapper` class; use `Sampling/variables.load_variables` + `sampling_utils.map_u_to_physical` for the sampler-side u→x it needs (exporting physical coords, `selection`). No inverse mapping is required — all geometry stays in u-space |
| C4 | `Bridson(radius=…).generate_batch(n)` | No such class API. Reuse the **module-level `Bridson_sampling(dims, radius, k, hypersphere_sample)`** function from `Sampling/bridson.py` for generation-0 (d ≤ 4); refinement candidates are drawn per-edge (§4.4), not via a local Bridson instance |
| C5 | `__init__(self, config)` | V2 samplers are no-arg constructed by a `Distributor` factory and configured via `set_config(config)` |
| C6 | Inherit `SamplingVirtial`; add a `match` case in `Distributor` | Inherit **`FeedbackSampler`** (D13.1; sits on `CheckpointedSampler` so checkpoint heartbeat + resume come free); register via **`Distributor.register("AdaptiveLevelSet", factory, stateless=False, resume="implemented")`** — the registry landed with D9.2, no `match` edits exist anymore |
| C7 | `Variables: [m0, m12]` in the YAML example | Wrong shape — `Sampling.Variables` is a list of mappings (`name` + `distribution`), per `YAML_REFERENCE_2.0.md` §6.2. Corrected recipe in §6 |
| C8 | Results consumed ad-hoc from `hep:results` inside the loop | The archive queue has exactly one consumer (the Archiver); a sampler pulling from it would race the persistence path. A **new opt-in feedback channel** is required (§5) — the only new runtime infrastructure this sampler needs |
| C9 | Refinement whenever "new results" arrive (timeout-polled) | Non-deterministic: refinement decisions would depend on worker-race arrival order, breaking seeded reproducibility (a V2 hard invariant). Corrected to **generation barriers** (§4.1): propose generation *g*, wait for *all* of generation *g*, then refine. Slightly lower worker utilization at generation edges; exact reproducibility for free |

### 2.2 Against the dimension-hybrid draft

| # | Draft said | Correction |
|---|---|---|
| C10 | Generation-0 "仍然用全局 Bridson（支持任意 dim）" | Two hard blockers: the as-built `Bridson` sampler class guards **2 ≤ d ≤ 4** (`bridson.py:135-136`), and `Bridson_sampling`'s background grid is `O((√d/r)^d)` memory — at d=5, r=0.08 that is ~28⁵ ≈ 17M cells ≈ **344 MB** (verified). Generation-0 uses `Bridson_sampling` for d ≤ 4 and **`scipy.stats.qmc.Sobol`** (scrambled, seeded from the generation-0 child SeedSequence) for d = 5 |
| C11 | Seed points selected by `\|f − target\| < threshold` (primary criterion) | Scale-dependent: a sensible threshold requires knowing the magnitude of `f` up front (LogL scales vary by orders of magnitude across models). The **sign-change-on-edge criterion is scale-free** and already proven in the 2-D design — it stays the *only* primary criterion in every mode. Local-variance / gradient hints may enter later as refinement *prioritization* (Q6), never as the crossing definition |
| C12 | 4–5D mode = separate logic (seed points + Gaussian jitter `u + N(0, σ)` per point) | Unified instead: 4–5D uses the **same crossing-edge machinery** with the Delaunay edge set replaced by a **symmetric kNN edge set** (union of each point's `knn_k` nearest neighbors, `cKDTree`). Refinement stays edge-based (ball around the crossing-edge midpoint, §4.4) — one code path, one set of convergence criteria, one determinism argument, and Gaussian jitter's unbounded tails never leave `[0,1]^d` guards to chance |
| C13 | 3-D "输出 mesh" stated as a given | Honest scoping: guaranteed 3-D output is the **crossing-point cloud** (+ slice projections). Watertight surface extraction (marching tetrahedra over the Delaunay simplices) is real additional work — kept as open question Q5, not promised |

Everything else in the drafts (motivation, u-space geometry, radius decay with a more
conservative factor in high-d, priority budgeting idea, honesty requirements on 4–5D output,
slice-projection visualization) is adopted with light edits.

---

## 3. Class structure

```python
@dataclass
class LevelSetPoint:
    u: np.ndarray            # shape (d,), u-space
    x: dict[str, float]      # physical params (via map_u_to_physical; for export)
    f: float | None          # target expression over returned observables; None = in flight
    uuid: str = ""
    generation: int = 0

class NeighborGraph(Protocol):
    def build(self, points_u: np.ndarray) -> np.ndarray: ...   # returns (m, 2) int edge array

class DelaunayGraph:      # d ≤ 3: exact adjacency = voronoi.ridge_points (qhull "QJ")
class KNNGraph:           # d = 4–5: symmetric union of knn_k nearest-neighbor edges (cKDTree)

class AdaptiveLevelSetSampler(FeedbackSampler):   # D13.1: FeedbackSampler → CheckpointedSampler
    method = "AdaptiveLevelSet"

    # --- config (set_config, from Sampling.AdaptiveLevelSet) ---
    _target_expression: str            # REQUIRED
    _target_value: float               # REQUIRED
    _contour_precision: float          # default 0.01   (u-space max crossing-edge length)
    _function_tolerance: float         # default 0.05   (max |f_i - f_j| across a crossing edge)
    _initial_radius: float             # default 0.08   (generation-0 spacing scale, u-space)
    _refinement_factor: float          # default 0.5 (d ≤ 3) / 0.65 (d ≥ 4) — conservative in high-d
    _max_generations: int              # default 25
    _max_points: int                   # default 5000 (d ≤ 3) / 20000 (d ≥ 4)
    _knn_k: int                        # default 4·d; d ≥ 4 only
    _neighbor_graph: str               # auto | delaunay | knn  (auto: delaunay d≤3, knn d≥4)

    # --- runtime state (all picklable → export_runtime_state) ---
    _dim: int                          # = len(Variables); 2 ≤ d ≤ 5 enforced
    _graph: NeighborGraph              # rebuilt from config on resume (not serialized)
    _points: list[LevelSetPoint]
    _generation: int
    _pending_uuids: set[str]           # in-flight members of the current generation
    _accepted_index: int               # drives deterministic uuids
    _seed_seq_state: dict              # np.random.SeedSequence serialized (per-gen children)
    _converged: bool
    _levelset: dict | None             # final output payload (§4.5)
```

Compiled sympy callables, the `NeighborGraph` instance, and any scipy spatial objects are
**never serialized** — all are rebuilt from `_points` / config on resume (same rule as
`likelihood.py`'s compiled expressions).

---

## 4. Algorithm

### 4.1 Generation loop (control process; barrier-synchronized — C9; mode-independent)

```
set_config → compile target expression; validate 2 ≤ len(Variables) ≤ 5;
             pick NeighborGraph per §1.1 (or explicit neighbor_graph override)
generation 0:
    d ≤ 4: us = Bridson_sampling(dims=[1]*d, radius=_initial_radius, k=30, surface_sample)
    d = 5: us = qmc.Sobol(d, scramble=True, seed=gen0_child).random(n0)      # C10
           (n0 chosen so mean spacing ≈ _initial_radius: n0 ≈ ceil((1/_initial_radius)^d
            capped by _max_points/4))
    submit_generation(us)                     # uuids deterministic (§7); feedback flag on (§5)
loop:
    wait until every uuid of this generation has a feedback record (barrier)
    fill LevelSetPoint.f from feedback observables via compiled expression
    edges = _graph.build(all_u)                                              # C1 / C12
    crossing = [(i, j) for (i, j) in edges if (f[i]-t)*(f[j]-t) <= 0]
    if not crossing:                 → stop: "level-set not present in domain" (report)
    if converged(crossing):          → break
    if generation >= _max_generations or len(points) >= _max_points: → break (report partial)
    new_us = refine(crossing, radius=_initial_radius * _refinement_factor**generation)
    submit_generation(new_us)
finalize()                                                                    # §4.5, per dim
```

`converged(crossing)`: **both** `max ‖u_i − u_j‖ < _contour_precision` **and**
`max |f_i − f_j| < _function_tolerance` over all crossing edges.

### 4.2 What the Worker sees

Nothing new. Tasks are ordinary light task-dicts (`u_coords` + execution plan); the target
expression is evaluated **control-side** over the observables that come back — Workers do not
know they are running an adaptive scan. The only Worker-visible addition is the feedback
publish flag (§5).

### 4.3 Crossing detection (C1/C12, corrected & unified)

An edge `(i, j)` of the neighbor graph crosses the level-set when
`(f_i − target)·(f_j − target) ≤ 0` (points exactly on the level-set count as crossing).

- **DelaunayGraph (d ≤ 3)**: `scipy.spatial.Voronoi(points_u, qhull_options="QJ").ridge_points`
  — the exact adjacency; no crossing between adjacent sites can be missed.
- **KNNGraph (d = 4–5)**: `cKDTree.query(points_u, k=_knn_k+1)`, symmetric union of the
  resulting directed edges. Approximate: a crossing "between" two points that are not among
  each other's k nearest neighbors is invisible *this generation* — but refinement makes the
  local cloud denser each round, so persistent structure is picked up in later generations.
  This is the accepted accuracy/complexity trade of the 4–5D regime (§1.1); `knn_k` is the
  tuning knob (too small ⇒ fragmented graph, slower discovery; default 4·d).

Failed samples (`status == "Failed"` or expression `KeyError`) get `f = None` and their
edges are treated as **crossing** (conservative: keeps refinement alive near unknowns) but
are excluded from the convergence maxima — a region that keeps failing is reported in the
final summary instead of silently blocking convergence (bounded by `_max_generations`).

### 4.4 Refinement (per crossing edge; mode-independent)

For each crossing edge `(i, j)` with midpoint `m = (u_i + u_j)/2` and current-generation
radius `r`:

1. draw `k_ref` (default 4) candidates uniformly in the **d-ball** of radius
   `max(r, ‖u_i − u_j‖/2)` around `m`, from the **generation's own seeded RNG** (§7);
2. clip to `[ε, 1−ε]^d`;
3. reject any candidate closer than `r/2` to an existing or already-accepted-this-round
   point (keeps blue-noise-like spacing; O(n·k) with a `cKDTree`).

**Budgeting (high-d)**: when one generation's crossing-edge count × `k_ref` would exceed
`max_new_per_generation` (default `_max_points // 10`), edges are processed in a
**deterministic priority order** — longest edge first, ties by `|f_i − f_j|` descending,
then by `(i, j)` index — and the round is cut at the budget. Deterministic given barrier
semantics, so reproducibility holds (this realizes the draft's priority-queue idea without
its arrival-order dependence).

### 4.5 Finalize (dimension-dependent)

**All dims** — crossing points: for each converged crossing edge, linear interpolation
`u* = u_i + (t − f_i)/(f_j − f_i) · (u_j − u_i)` puts a vertex on the level-set.

- **d = 2**: walk crossing ridges through shared Voronoi cells to chain vertices into
  ordered polylines (disconnected components emerge as separate chains); optional
  Douglas-Peucker simplification (`simplify_tolerance`, default off).
- **d = 3**: crossing-point cloud, exported as-is. Watertight surface extraction is Q5.
- **d = 4–5**: crossing-point cloud **plus** 2-D slice projections: for every axis pair
  `(a, b)` (or a user-listed subset `slice_pairs`), the cloud projected onto `(u_a, u_b)`
  with the remaining coordinates recorded — enough for PLOT to render "adaptive density near
  the level-set" panels. **Honesty rule (from the draft, kept)**: the output declares
  `"fidelity": "proximity-approximate"` and the run log states that no full hypersurface
  reconstruction is claimed.

**Output**: `levelset.json` in `task_result_dir` — `{dim, mode, polylines_u/x (d=2),
crossing_points_u/x, slice_projections (d≥4), target_value, precision_achieved,
n_points_total, n_generations, failed_regions, fidelity}` — plus the ordinary
DATABASE/SAMPLE output for every evaluated point (unchanged archive path). The PLOT bridge
consumes `levelset.json` directly.

---

## 5. New infrastructure: the result feedback channel (C8)

The only piece V2 lacks today. **Spec:**

- New Redis key `hep:feedback` (List). When `worker_config["publish_feedback"]` is true
  (default **false** — zero cost for every existing sampler), the Worker, in the same place
  it calls `submit_result`, also `rpush`es a **light** record:
  `{uuid, status, observables}` (no paths, no product list — the Archiver payload stays the
  single source for persistence).
- `RedisQueue` gains `publish_feedback(info)` / `pull_feedback(timeout)` mirroring the
  task-queue helpers, plus drain-on-resume (stale feedback from a previous run is discarded
  the same way `drain_task_queue` handles stale tasks).
- The sampler sets `publish_feedback` via its worker-config contribution when registered as
  the active method; `core.run()` dispatches non-stateless registrations
  (`registration.stateless == False`) to a new `sampler.run_adaptive()` entry instead of
  `run_distributed()` — the third branch next to check-modules and stateless scans.
- Note: `RedisQueue.store_result_hash` (per-uuid hash, currently **zero callers**) was
  pre-built for this kind of handoff; a list is preferred here because the barrier wants
  blocking pops, not n-key polling. Decide in D10.1 whether to delete or wire that orphan.

---

## 6. YAML configuration (corrected — C7)

```yaml
Sampling:
  Method: AdaptiveLevelSet
  Seed: 42                            # int (alias seed); drives ALL generations (§7)
  Variables:                          # 2–5 entries (ValueError otherwise)
    - name: m0
      distribution: {type: Flat, parameters: {min: 0.0, max: 5000.0}}
    - name: m12
      distribution: {type: Log,  parameters: {min: 100.0, max: 3000.0}}
    # - name: tanb   …                # add up to 5; d ≥ 4 switches to proximity mode (§1.1)
  AdaptiveLevelSet:
    target_expression: "LogL"         # REQUIRED; sympy over returned observables (A.6 caveat)
    target_value: -2.9957             # REQUIRED; e.g. 95% CL in ΔLogL
    contour_precision: 0.008          # u-space; default 0.01
    function_tolerance: 0.05          # default 0.05
    initial_radius: 0.08              # generation-0 spacing; default 0.08
    refinement_factor: 0.45           # default 0.5 (d ≤ 3) / 0.65 (d ≥ 4)
    max_generations: 25               # default 25
    max_points: 5000                  # default 5000 (d ≤ 3) / 20000 (d ≥ 4)
    # neighbor_graph: auto            # auto | delaunay | knn (auto: delaunay d≤3, knn d≥4)
    # knn_k: 16                       # d ≥ 4 only; default 4·d
    # max_new_per_generation: 500     # refinement budget per round; default max_points//10
    # slice_pairs: [[m0, m12]]        # d ≥ 4: which 2-D projections to export (default: all pairs)
    # simplify_tolerance: 0.002       # d = 2 only: optional polyline simplification (default off)
```

Key conventions follow `YAML_REFERENCE_2.0.md`: missing `target_expression`/`target_value`
raise `ValueError` at `set_config`; the sub-block name matches the method name; `selection`
is honored on generation-0 candidates (rejected candidates are dropped before submission —
with the same fixed-budget semantics as the other samplers). `length` is **not** used (the
u-domain is always the unit cube) — one fewer place for the A.11 inconsistency. Explicitly
overriding `neighbor_graph: delaunay` with d ≥ 4 is allowed but logged with a cost warning;
`knn` with d ≤ 3 is allowed (useful for testing graph-mode agreement, §9.8). On
implementation, add the block to `YAML_REFERENCE_2.0.md` (new §6.9 + Key Index rows).

---

## 7. Determinism, checkpoint, resume

- **Seeding**: one master `np.random.SeedSequence(Seed)`; child `spawn()` per generation
  (generation-0 child feeds `Bridson_sampling`'s `np.random.seed` the way `bridson.py` does,
  or the Sobol `seed=` at d = 5; refinement generations get their own child RNG). Same seed
  ⇒ identical candidate sets in every mode.
- **Uuids**: `deterministic_sampler_uuid(prefix="alevelset", seed, sample_index=
  accepted_index)` — accepted order is deterministic given C9 barriers and the §4.4
  deterministic priority order, so uuids are stable across reruns and worker counts (same
  argument as the D6.2 samplers).
- **Barrier = safe barrier**: `at_safe_barrier()` is true exactly between generations
  (`_pending_uuids` empty) — the checkpoint heartbeat therefore snapshots only at generation
  edges, where state is minimal and consistent.
- **Checkpoint payload**: `_points` (with `f`), `_dim`, `_generation`, `_pending_uuids`,
  `_accepted_index`, serialized SeedSequence state, config echo. Graph/compiled expression
  rebuilt on import (§3).
- **Resume**: drain `hep:feedback` + `hep:task_queue`; re-propose `_pending_uuids` (their
  u-coords are in `_points` with `f=None`); continue the loop. Matches the existing
  "never restore in-flight, re-propose" rule.

---

## 8. Numerical edge cases

- Degenerate/collinear point sets (d ≤ 3): `qhull_options="QJ"` (joggle) on every build.
- Domain boundary: candidates clipped to `[ε, 1−ε]^d` with `ε = np.finfo(float).eps`-scale
  guards (same guards as `variables.py`); level-sets exiting the domain terminate polylines
  (d = 2) at the last interior crossing.
- Plateaus (`f ≡ target` over a region): every internal edge is "crossing"; the
  `function_tolerance` criterion converges immediately, and the finalize step reports a
  *band/blob* (flagged in `levelset.json`) rather than pretending a thin surface.
- kNN fragmentation (d ≥ 4): a `knn_k` too small can hide crossings between graph
  components; mitigation = default 4·d, the per-generation densification (§4.3), and a
  final-report `graph_components` count so the user can see fragmentation.
- Sobol n₀ (d = 5): `qmc.Sobol.random` powers-of-two balance — round n₀ up to the next
  power of two (`random_base2`) for best uniformity.
- d outside [2, 5]: `set_config` raises `ValueError` (≥6-D is out of scope by design, §1.1).

---

## 9. Tests (`tests/test_adaptive_level_set.py`)

1. **Synthetic parity (d=2)**: circle / ellipse / two disjoint blobs via an opera-only `f`;
   Hausdorff distance of the polyline to the analytic contour < 2×`contour_precision`
   (u-space).
2. **Seeded reproducibility (all modes)**: same seed + different `workers` ⇒ byte-identical
   point set, uuids, and output payload (the C9 barrier + §4.4 priority order make this pass).
3. **Precision monotonicity (d=2)**: 0.05 / 0.01 / 0.005 ⇒ point count increases, Hausdorff
   distance decreases monotonically.
4. **Efficiency gate (d=2)**: total points ≤ 30% of a dense Bridson run achieving the same
   Hausdorff distance on the ellipse fixture (the raison d'être, asserted).
5. **Checkpoint round-trip**: SIGKILL-style stop mid-generation → resume → identical final
   output vs the uninterrupted run.
6. **Failure handling**: an `f` that raises for a sub-region → run completes,
   `failed_regions` reported, no hang.
7. **Feedback-channel unit tests**: publish flag off ⇒ zero `hep:feedback` writes; drain on
   resume; barrier releases only on full generation.
8. **Graph-mode agreement (d=2)**: on a dense evaluated point set, `knn` (k = 8) and
   `delaunay` graphs find the same crossing regions (crossing-point clouds within
   2×`contour_precision` Hausdorff) — validates the kNN approximation against exact adjacency
   where ground truth is available.
9. **d=3 sphere fixture**: `f = ‖u − c‖`, target r → crossing cloud within tolerance of the
   analytic sphere; converges.
10. **d=4 hypersphere-shell fixture (proximity mode)**: crossing cloud mean distance to the
    analytic shell < `contour_precision`; density near shell ≥ 5× the density far from it
    (the "details reasonable" assertion); Sobol path exercised at d=5 with a smoke run.

---

## 10. Work packages (plan milestone D10) & open questions

| WP | Delivers | Depends |
|----|----------|---------|
| D10.1 | Feedback channel: `hep:feedback` list, `publish_feedback`/`pull_feedback`, worker flag, drain-on-resume; decide fate of orphan `store_result_hash` | — |
| D10.2 | Sampler core (d=2 flagship): generation loop, `NeighborGraph` protocol + `DelaunayGraph`, crossing detection, edge refinement + deterministic budget order, convergence; registered `stateless=False`; `core.run_adaptive` dispatch branch | D10.1 |
| D10.3 | Finalize + outputs (d=2): polyline chaining, `levelset.json`, PLOT-bridge overlay hook | D10.2 |
| D10.4 | Determinism + checkpoint/resume + the §9 core test suite (items 1–8); YAML_REFERENCE §6.9 entry — **done 2026-07-16** | D10.2 |
| D10.5 | Dimension extension: `KNNGraph` + proximity mode (d=4–5), Sobol generation-0 (d=5), d=3 crossing cloud, slice projections, dimension-dependent defaults, §9 items 9–10 — **done 2026-07-16** (strict 5× near/far density not CI-hard; cloud accuracy + knn/slices + ≥uniform near-band) | D10.2 |

**Open questions** (decide during implementation, none blocking D10.1):

1. Multiple simultaneous levels (`target_values: [v1, v2]`) — union of crossing sets per
   level; cheap to add, deferred for scope.
2. Incremental Voronoi/kNN for very large point sets — full rebuilds are O(n log n) and fine
   at the default budgets; revisit only if budgets grow 10×.
3. Worker idling at generation barriers — acceptable for the slow regime; if it hurts,
   a bounded "speculative next generation" is possible but costs determinism (would need a
   design amendment; do not sneak it in).
4. Higher-d slices (fix parameters, sweep 2-D runs) — orchestration-level loop over 2-D
   runs; belongs to the Agent/skills layer, not this sampler.
5. **d=3 surface mesh** (marching tetrahedra over the Delaunay simplices with the already-
   computed crossing edges) — natural extension of D10.5, but proper orientation/watertight
   handling is real work; point cloud ships first (C13).
6. **Refinement prioritization hints** beyond edge length — local f-variance / gradient
   estimates from the kNN neighborhood could rank steep regions higher (second draft's
   suggestion); admissible because prioritization only reorders a deterministic queue (§4.4),
   never redefines crossing (C11).
