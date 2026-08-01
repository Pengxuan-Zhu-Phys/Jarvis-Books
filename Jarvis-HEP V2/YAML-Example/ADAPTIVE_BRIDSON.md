# YAML Example — AdaptiveBridson

Public task-YAML recipes for **`Sampling.Method: AdaptiveBridson`**.

| Doc | Role |
|-----|------|
| **This file** | Minimal + full YAML examples |
| [`DESIGN_ADAPTIVE_BRIDSON_LIVE_BAND.md`](../DESIGN_ADAPTIVE_BRIDSON_LIVE_BAND.md) | **Complete algorithm flow** (binding) |
| [`YAML_REFERENCE_2.0.md`](../YAML_REFERENCE_2.0.md) §6.9 | Full key inventory |
| Code | `jarvishep2/Sampling/adaptive_bridson.py` |

Redis is an internal V2 broker; it is **not** configured in task YAML.

---

## Algorithm in one page

Geometry in **u-space** \(u\in(0,1)^d\). Physical map only for evaluation.

1. **Gen-0** — global Bridson with `initial_radius` \(r_0\); evaluate \(f\).
2. **Classify** absolute bands  
   - core: \(\lvert f-T\rvert \le w_{\mathrm{core}}\) (densify centers)  
   - frontier: \(w_{\mathrm{core}} < \lvert f-T\rvert \le w_{\mathrm{outer}}\) (support)  
   - default \(w_{\mathrm{core}} = w_{\mathrm{outer}}/8\)
3. **Gate** \(t_{\min}, t_{\max}\) from finite \(f\) inside a **\(2\,r_g\)** ball about best  
   (\(\arg\min\lvert f-T\rvert\)).
4. **Same-\(r_g\) fill** (many `fill_pass`, generation does **not** advance):  
   root-correction on straddles → active endpoints (omni probes) → MST bridge  
   → local Bridson densify in each core’s \(r_g\) ball (blue-noise sep \(=r_g\)).
5. **When fill is quiet** — if \(t_{\max}-t_{\min} < \tau\) **and** cores cover the  
   contour → **converged**; else shrink \(r_g \leftarrow \max(r_{\min}, r_g\times\rho)\),  
   `generation += 1`, rebuild endpoints, densify finer.
6. Stop also at `min_radius`, `max_generations`, or `max_points`.

See the design doc for brackets A/B/C/D, coverage, and non-goals.

---

## Public knobs (new cards)

| Key | Required | Default | Meaning |
|-----|----------|---------|---------|
| `target_expression` | **yes** | — | sympy over returned observables |
| `target_value` | **yes** | — | level \(T\) |
| `outer_half_width` | no | `0.02` | discovery \(\lvert f-T\rvert\le w_{\mathrm{outer}}\) |
| `min_radius` | no | `1/200` | u-space Euclidean floor for \(r_g\) |

Derived automatically:

- `core_half_width = outer_half_width / 8`
- `threshold = core_half_width`  (stop on \(t_{\max}-t_{\min}\))

Everything else has safe internals. Prefer the **minimal** card unless you
need to override budgets or radii.

---

## 1. Minimal example (recommended)

Opera-only 2-D circle — no external calculator. Run from any project with
V2 installed:

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
    # Optional public tuning (defaults are fine for this toy):
    # outer_half_width: 0.05
    # min_radius: 0.005

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

```bash
Jarvis2 run path/to/circle.yaml
Jarvis2 convert path/to/circle.yaml    # DATABASE/samples.hdf5 → samples.csv
```

### Physics-style minimal (iDM-shaped Sampling only)

```yaml
Sampling:
  Method: AdaptiveBridson
  Seed: 21
  Variables:
    - name: MChi
      distribution: { type: Log, parameters: { min: 0.1, max: 100.0 } }
    - name: Y
      distribution: { type: Log, parameters: { min: 1.0e-10, max: 1.0e-3 } }
  selection: "Y < 9.5e-4"       # optional physical filter before submit

  AdaptiveBridson:
    target_expression: "Omega_h2"
    target_value: 0.12
    outer_half_width: 0.02      # |Ω − 0.12| ≤ 0.02 discovery support
    min_radius: 0.002           # final u-space resolution (~1/500)
```

`core_half_width` / stop threshold become `0.02/8 = 0.0025` automatically.
Wire `Calculators` / `Likelihood` as in the full example below.

---

## 2. Full example (explicit internals)

Use when you need budgets, graph mode, or to document every control. Values
below match the production iDM Vector AdaptiveBridson card defaults.

```yaml
project_name: adaptive-bridson-full

Scan:
  name: levelset-full
  save_dir: "&J/outputs"
  sample_directory:
    limit: 200
    width: 6
    archive_samples: true

EnvReqs:
  V2:
    workers: 8
    batch_size: 32
  # Check_default_dependencies: …   # optional host env merge

Sampling:
  Method: AdaptiveBridson
  Seed: 21

  Variables:                     # length must be 2..5
    - name: MChi
      description: "DM mass [GeV]"
      distribution:
        type: Log
        parameters: { min: 0.1, max: 100.0 }
    - name: Y
      description: "dimensionless y"
      distribution:
        type: Log
        parameters: { min: 1.0e-10, max: 1.0e-3 }
    # - name: third_dim          # optional 3rd–5th axis
    #   distribution: { type: Flat, parameters: { min: 1.0, max: 60.0 } }

  selection: "Y < 9.5e-4"        # optional sympy bool on *physical* params

  AdaptiveBridson:
    # ---- required ----------------------------------------------------------
    target_expression: "Omega_h2"
    target_value: 0.12

    # ---- public tuning (preferred) -----------------------------------------
    outer_half_width: 0.02       # discovery |f−T| ≤ 0.02
    min_radius: 0.002            # u-space floor; no denser than this

    # ---- radii / scale (usually leave defaults) ----------------------------
    initial_radius: 0.10         # r0 gen-0 Bridson + first windows
    refinement_factor: 0.5       # r_g ← max(min_radius, r_g × factor)
    # radius_shrink_mode: on_coverage   # default; alias: on_fill

    # ---- derived / legacy (optional overrides; not needed for new cards) ---
    # core_half_width: 0.0025    # default = outer_half_width / 8
    # threshold: 0.0025          # default = core_half_width; t_max−t_min stop
    # function_tolerance: …      # compat alias of threshold
    # final_half_width: 0.001    # optional tighter export band ≤ core

    # ---- fill / bridge / endpoints -----------------------------------------
    bridge_gaps: true
    bridge_span_factor: 2.5      # max core–core bridge = factor × r_g
    k_ref: 30                    # Bridson trials per densify center
    quiet_fill_passes: 3         # no-progress passes → scale quiet
    # max_fill_passes: 64
    # endpoint_stall_passes: 12
    # endpoint_omni_probes: 16
    # core_spacing_factor: 2.0   # coverage: max core NN gap ≤ factor × r_g
    # min_cores_for_coverage: 4
    # outer_shrink_factor: 0.7    # anneal w_outer → w_core when brackets clear

    # ---- budgets -----------------------------------------------------------
    max_generations: 16          # max r_g shrinks (scale index)
    max_points: 50000
    max_new_per_generation: 4000 # densify budget per fill_pass

    # ---- neighbor graph (brackets / support; not densify engine) -----------
    neighbor_graph: auto         # auto | delaunay | knn
    # knn_k: 8                   # default 4 * d when kNN
    # slice_pairs: [[MChi, Y]]   # d≥4 projections in levelset.json

  # Optional soft preference (AdaptiveBridson still drives on target_expression)
  LogLikelihood:
    - { name: "LogL_Omega", expression: "LogGauss(Omega_h2, 0.1200, 0.0012)" }

Calculators:
  # Modules: …                 # external tools (e.g. microOMEGAs)
  #   execution.input / output with save: true → SAMPLE/

# Operas:
#   Modules: …                 # pure-Python f(obs) operators

# Likelihood:
#   expressions: …             # if target needs LogL products
```

Production reference cards:

- `Jarvis-Examples/iDM/bin/iDM_Vector_AdaptiveBridson_Omega.yaml`
- `Jarvis-Examples/iDM/bin/iDM_Axial_AdaptiveBridson_Omega.yaml`

---

## Sampling key tables

### Shared `Sampling` keys

| Key | Required | Default | Notes |
|-----|----------|---------|--------|
| `Method` | **yes** | — | `AdaptiveBridson` (case-sensitive) |
| `Seed` | no | `0` | alias `seed`; seeds all gens / fill passes |
| `selection` | no | — | physical-param filter before submit |
| `Variables` | **yes** | — | length **2–5** |
| `AdaptiveBridson` | **yes** | — | alias `adaptive_bridson` |
| `LogLikelihood` | no | — | alias of top-level `Likelihood.expressions` |

### `Sampling.AdaptiveBridson` keys

#### Public (prefer these)

| Key | Required | Default | Notes |
|-----|----------|---------|--------|
| `target_expression` | **yes** | — | sympy over observables |
| `target_value` | **yes** | — | level \(T\) |
| `outer_half_width` | no | `0.02` | discovery band in \(f\) |
| `min_radius` | no | `1/200` | u-space Euclidean floor |

#### Common optional

| Key | Default | Notes |
|-----|---------|--------|
| `initial_radius` | `0.10` | \(r_0\) |
| `refinement_factor` | `0.5` | \(r_g\) shrink multiplier |
| `bridge_gaps` | `true` | MST reconnection |
| `bridge_span_factor` | `2.5` | max bridge length factor |
| `max_generations` | `16` | max \(r_g\) shrinks |
| `max_points` | `50000` | hard sample budget |
| `max_new_per_generation` | `4000` | densify budget / fill_pass |
| `k_ref` | `30` | Bridson trials / densify center |
| `neighbor_graph` | `auto` | `auto` / `delaunay` / `knn` |
| `knn_k` | `4 * d` | when graph is kNN |

#### Advanced / legacy

| Key | Default | Notes |
|-----|---------|--------|
| `core_half_width` | `outer/8` | densify-center half-width |
| `threshold` | `= core_half_width` | \(t_{\max}-t_{\min}\) stop |
| `function_tolerance` | — | compat alias of `threshold` |
| `final_half_width` | null | optional tighter export band |
| `radius_shrink_mode` | `on_coverage` | or `every_generation` (legacy) |
| `outer_shrink_factor` | `0.7` | anneal \(w_{\mathrm{outer}}\) |
| `core_spacing_factor` | `2.0` | coverage NN gap |
| `min_cores_for_coverage` | `4` | coverage floor |
| `quiet_fill_passes` | `3` | scale quiescence |
| `max_fill_passes` | `max(64, 4·max_gen)` | safety |
| `endpoint_stall_passes` | `12` | retire a front |
| `endpoint_omni_probes` | `max(16, 4d)` | directions per front / pass |
| `slice_pairs` | all pairs | d≥4 projections |

### Related scheduling keys

| Key | Effect |
|-----|--------|
| `EnvReqs.V2.workers` | Worker process count |
| `EnvReqs.V2.batch_size` | submit-group size |

Workers get `publish_feedback: true` automatically for AdaptiveBridson.

---

## Dimension policy

| \(d=\) `len(Variables)` | Gen-0 | Neighbor graph | Output emphasis |
|-------------------------|-------|----------------|-----------------|
| 2 | Bridson | Delaunay | Polylines + cloud |
| 3 | Bridson | Delaunay | Crossing cloud |
| 4 | Bridson | kNN | Cloud + slice projections |
| 5 | **Sobol** | kNN | Same as d=4 |
| else | — | — | `ValueError` at config |

---

## Outputs

| Artifact | Path |
|----------|------|
| HDF5 archive | `outputs/<scan>/DATABASE/samples.hdf5` |
| CSV | `Jarvis2 convert <task.yaml>` → `samples.csv` |
| Level-set JSON | `outputs/<scan>/levelset.json` |
| SAMPLE | `outputs/<scan>/SAMPLE/…` when `save: true` |

`levelset.json` includes `algorithm: outer_core_root_correction_bridson`,
radii, bands, convergence flags, and (for d=2) polylines for jplot.

---

## CLI

```bash
Jarvis2 run    bin/task.yaml
Jarvis2 check  bin/task.yaml          # fixed-point smoke if configured
Jarvis2 convert bin/task.yaml         # HDF5 → CSV
Jarvis2 convert bin/task.yaml --force # overwrite existing CSV
```
