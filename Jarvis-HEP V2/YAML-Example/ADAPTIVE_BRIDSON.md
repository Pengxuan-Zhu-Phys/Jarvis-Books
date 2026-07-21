# YAML Example — AdaptiveBridson

Public-facing task-YAML example for **`Sampling.Method: AdaptiveBridson`**
(feedback-driven level-set tracer, 2 ≤ d ≤ 5).

- **As-built code**: `jarvishep2/Sampling/adaptive_bridson.py`
- **Design**: [`components/adaptive_voronoi_contour.md`](../components/adaptive_voronoi_contour.md)
- **Full key inventory** (all methods): [`YAML_REFERENCE_2.0.md`](../YAML_REFERENCE_2.0.md)

Other top-level blocks (`EnvReqs`, `Calculators`, `Operas`, …) are shown only as needed.
**`Sampling` is fully specified** below for documentation use. Redis is an internal V2
broker (local `127.0.0.1:6379`); it is not configured in task YAML.

---

## Annotated skeleton

```yaml
project_name: adaptive-level-set-example

Scan:
  name: levelset-01
  # …

EnvReqs:
  V2:
    workers: 4                   # Worker count (default 0 → factory uses 1)
    batch_size: 256              # submit-group size (sampler submit batches)
  # Check_default_dependencies:  # optional V1-shaped defaults merge (§ YAML_REFERENCE §4.1)
  #   required: true
  #   default_yaml_path: "&J/deps/environment_default.yaml"

Sampling:
  # ============================================================
  # AdaptiveBridson — Sampling block (complete for this method)
  # ============================================================

  Method: AdaptiveBridson       # REQUIRED for this recipe
                                 # (registered stateless=False; uses hep:feedback)

  Seed: 42                       # int; alias: seed; default 0
                                 # Master SeedSequence for all generations
                                 # (gen-0 Bridson/Sobol + every refine generation)

  selection: null                # optional sympy bool over *physical* params
                                 # (e.g. "m0 + m12 < 3000")
                                 # Applied to gen-0 and refine candidates before submit;
                                 # rejected candidates are dropped (not counted as failures)

  Variables:                     # REQUIRED; length must be 2..5 (else ValueError)
    - name: m0                   # REQUIRED per entry; free sympy name for expressions
      description: soft mass     # optional; informational only
      distribution:
        type: Flat               # Flat | Log | Normal | Log-Normal | Logit
        parameters:
          min: 0.0               # Flat/Log: min, max (physical bounds)
          max: 5000.0
          # mean / stddev        # Normal / Log-Normal
          # location / scale     # Logit (defaults 0 / 1)
          # num: …               # unused by AdaptiveBridson (Grid-only)
          # length: …            # unused by AdaptiveBridson
                                 # (u-domain is always the unit cube [0,1]^d)
    - name: m12
      distribution:
        type: Log
        parameters:
          min: 100.0
          max: 3000.0
    # - name: tanb               # optional 3rd–5th dimension
    #   distribution: { type: Flat, parameters: { min: 1.0, max: 60.0 } }
    # d = 2–3 → Delaunay neighbor graph (exact)
    # d = 4–5 → kNN neighbor graph (proximity-approximate)
    # d < 2 or d > 5 → ValueError at set_config

  AdaptiveBridson:              # REQUIRED sub-block (name matches Method)
                                 # alias accepted: adaptive_bridson

    # ---- required ----------------------------------------------------------
    target_expression: "LogL"    # REQUIRED (non-empty string)
                                 # expression over *returned* observables
                                 # (and variable names). Compiled via shared
                                 # ExpressionContext (same as Likelihood).
                                 # Avoid free symbols named E / I / pi / gamma / …
                                 # (remaining sympy collision caveat).
    target_value: -2.9957        # REQUIRED (float); level-set f(obs) = target_value
                                 # e.g. 95% CL in ΔLogL units

    # ---- convergence (u-space geometry + function gap) ---------------------
    contour_precision: 0.01      # float; default 0.01
                                 # max ‖u_i − u_j‖ over *known* crossing edges
    function_tolerance: 0.05     # float; default 0.05
                                 # max |f_i − f_j| over *known* crossing edges
                                 # Converged only if BOTH are satisfied

    # ---- generation-0 spacing ----------------------------------------------
    initial_radius: 0.08         # float; default 0.08
                                 # d ≤ 4: Bridson Poisson-disk radius on [0,1]^d
                                 # d = 5: Sobol n0 ~ (1/radius)^d (capped; power-of-two)

    # ---- refinement schedule -----------------------------------------------
    refinement_factor: 0.5       # float; default 0.5 if d ≤ 3, else 0.65
                                 # refine ball radius = initial_radius
                                 #                   * factor^generation
    max_generations: 25          # int ≥ 1; default 25
                                 # stop after this many refine rounds (partial ok)
    max_points: 5000             # int ≥ 10; default 5000 if d ≤ 3, else 20000
                                 # hard cap on evaluated samples
    max_new_per_generation: 500  # int ≥ 1; default max_points // 10
                                 # refine budget per generation; edges ordered
                                 # longest first, then |Δf|, then (i,j)
    k_ref: 4                     # int ≥ 1; default 4
                                 # candidates drawn per crossing edge
                                 # (uniform in d-ball about edge midpoint)

    # ---- neighbor graph ----------------------------------------------------
    neighbor_graph: auto         # auto | delaunay | knn; default auto
                                 # auto → delaunay if d ≤ 3, knn if d ≥ 4
                                 # delaunay at d ≥ 4 allowed but cost-warned
                                 # knn at d ≤ 3 allowed (tests / calibration)
    knn_k: 8                     # int ≥ 1; default 4 * d
                                 # used when graph is kNN (symmetric cKDTree)

    # ---- high-d finalize (optional) ----------------------------------------
    # slice_pairs:               # list of [name_a, name_b]; d ≥ 4 only
    #   - [m0, m12]              # default: all unordered pairs of Variables
    #                            # exports 2-D projections in levelset.json

    # ---- d = 2 polyline polish (optional; currently reserved) --------------
    # simplify_tolerance: 0.002  # float; optional Douglas-Peucker-style hook
                                 # (parsed; polyline chaining always runs for d=2)

  # LogLikelihood:               # optional alias of Likelihood.expressions
  #   - name: LogL               # (lower precedence than top-level Likelihood)
  #     expression: …

Mapper:
  # …  (optional; usually auto-derived from Sampling.Variables)

LibDeps:
  # …

Calculators:
  # Modules: …

Operas:
  # Modules: …                   # typical: pure-Python f(obs) operators

Likelihood:
  # expressions: …               # if target_expression uses LogL / terms
```

---

## Sampling key table (AdaptiveBridson)

### Shared `Sampling` keys

| Key | Required | Type / values | Default | Notes |
|-----|----------|---------------|---------|--------|
| `Method` | **yes** | `AdaptiveBridson` | — | Case-sensitive method name |
| `Seed` | no | int | `0` | Alias `seed`; seeds all generations |
| `selection` | no | sympy bool string | none | Physical-param filter before submit |
| `Variables` | **yes** | list of maps | — | Length **2–5**; each needs `name` + `distribution` |
| `Variables[].name` | **yes** | string | — | Parameter / sympy symbol |
| `Variables[].description` | no | string | — | Documentation only |
| `Variables[].distribution.type` | **yes** | `Flat` / `Log` / `Normal` / `Log-Normal` / `Logit` | — | Same catalog as other samplers |
| `Variables[].distribution.parameters` | **yes** | map | — | Bounds / mean / etc. per type |
| `AdaptiveBridson` | **yes** | map | — | Alias `adaptive_bridson` |
| `LogLikelihood` | no | list | — | Alias of top-level `Likelihood.expressions` |

### `Sampling.AdaptiveBridson` keys

| Key | Required | Type | Default | Notes |
|-----|----------|------|---------|--------|
| `target_expression` | **yes** | string | — | Sympy over returned observables |
| `target_value` | **yes** | float | — | Level-set constant |
| `contour_precision` | no | float | `0.01` | Max crossing-edge length in **u-space** |
| `function_tolerance` | no | float | `0.05` | Max \|f_i − f_j\| on known crossing edges |
| `initial_radius` | no | float | `0.08` | Gen-0 spacing scale (u-space) |
| `refinement_factor` | no | float | `0.5` (d≤3) / `0.65` (d≥4) | Radius decay per generation |
| `max_generations` | no | int ≥ 1 | `25` | Refine round limit |
| `max_points` | no | int ≥ 10 | `5000` (d≤3) / `20000` (d≥4) | Hard sample budget |
| `max_new_per_generation` | no | int ≥ 1 | `max_points // 10` | Per-generation refine budget |
| `k_ref` | no | int ≥ 1 | `4` | Candidates per crossing edge |
| `neighbor_graph` | no | `auto` / `delaunay` / `knn` | `auto` | Graph builder |
| `knn_k` | no | int ≥ 1 | `4 * d` | kNN degree (kNN modes) |
| `slice_pairs` | no | list of `[name, name]` | all pairs | d≥4 projections in `levelset.json` |
| `simplify_tolerance` | no | float | off | d=2 polish hook (parsed) |

### Related scheduling keys (not under `Sampling`, but affect this method)

| Key | Effect |
|-----|--------|
| `EnvReqs.V2.workers` | Worker process count (factory uses 1 when ≤ 0) |
| `EnvReqs.V2.batch_size` | Sampler submit-group size (default 256 after normalize) |

Distributed AdaptiveBridson always uses the internal Redis runtime (local service required).
Workers set `publish_feedback: true` automatically when `Method: AdaptiveBridson` (no YAML flag).

---

## Dimension policy (quick)

| d = `len(Variables)` | Gen-0 | Neighbor graph | Output emphasis |
|----------------------|-------|----------------|-----------------|
| 2 | Bridson | Delaunay | Polylines + crossing cloud |
| 3 | Bridson | Delaunay | Crossing cloud |
| 4 | Bridson | kNN | Crossing cloud + slice projections; `fidelity: proximity-approximate` |
| 5 | **Sobol** | kNN | Same as d=4 |
| else | — | — | `ValueError` at config time |

---

## Outputs

| Artifact | Path / notes |
|----------|----------------|
| DATABASE / SAMPLE | Unchanged archive path for every evaluated point |
| Level-set payload | `<task_result_dir>/levelset.json` |

Typical `levelset.json` fields: `dim`, `mode`, `target_expression`, `target_value`,
`crossing_points_u` / `_x`, `polylines_u` / `_x` (d=2), `slice_projections` (d≥4),
`n_points_total`, `n_generations`, `converged`, `stop_reason`, `fidelity`,
`graph_components`, `failed_regions`.

---

## Minimal worked example (2-D circle / opera-only)

```yaml
project_name: circle-levelset
Scan:
  name: circle-r2
EnvReqs:
  V2:
    workers: 2
    batch_size: 8
Sampling:
  Method: AdaptiveBridson
  Seed: 7
  Variables:
    - name: x
      distribution: { type: Flat, parameters: { min: 0.0, max: 1.0 } }
    - name: y
      distribution: { type: Flat, parameters: { min: 0.0, max: 1.0 } }
  AdaptiveBridson:
    target_expression: "r2"
    target_value: 0.04          # circle radius 0.2 about (0.5, 0.5)
    contour_precision: 0.05
    function_tolerance: 0.08
    initial_radius: 0.12
    max_generations: 12
    max_points: 800
Operas:
  Modules:
    - name: Circle
      operator: jarvishep2.testing.eggbox.circle_r2
      call_mode: call
      input:
        - { name: x, expression: x }
        - { name: y, expression: y }
      output:
        - { name: r2, entry: r2 }
```
