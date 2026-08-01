# Design — AdaptiveBridson (final)

**Status**: **Finalized** (2026-07-22) after real iDM / microOMEGAs runs.  
**Code**: `jarvishep2/Sampling/adaptive_bridson.py`  
**levelset.json algorithm id**: `outer_core_root_correction_bridson`  
**YAML recipe**: [`YAML-Example/ADAPTIVE_BRIDSON.md`](YAML-Example/ADAPTIVE_BRIDSON.md)  
**Key inventory**: [`YAML_REFERENCE_2.0.md`](YAML_REFERENCE_2.0.md) §6.9  

**Scope**: sampling control only — geometry in **u-space**, densify near
\(f(u)=T\), stop when the local target band is thin enough. Contour polylines
for plotting are **post-hoc** and do **not** drive proposals.

Historical edge-midpoint refine (D10) is retired as a proposal engine. See
[`components/adaptive_voronoi_contour.md`](components/adaptive_voronoi_contour.md)
for the old design.

---

## 1. Goal

Approximate the level set \(f(u)=T\) in the unit cube \(u\in(0,1)^d\)
(\(2\le d\le 5\)):

1. Discover the contour cheaply (global coarse Bridson).
2. Densify only near points that already sit on / near the target.
3. Grow disconnected pieces along the contour until they reconnect.
4. Refine the spatial scale \(r_g\) until the local \(f\)-band around the best
   point is thin enough, or \(r_g\) hits a minimum floor.

Physical parameters are obtained only via `map_u_to_physical` when building
Samples. All Bridson distances, windows, and blue-noise separation live in
**u-space**.

---

## 2. Notation

| Symbol | Meaning |
|--------|---------|
| \(u\in(0,1)^d\) | Unit-cube coordinate (geometry space) |
| \(f(u)\) | Value of `target_expression` after evaluation |
| \(T\) | `target_value` |
| \(r_g\) | Current spatial scale (window / blue-noise radius) |
| \(r_0\) | `initial_radius` (default `0.10`) |
| \(r_{\min}\) | `min_radius` (default `1/200`) — u-space floor |
| \(w_{\mathrm{outer}}\) | `outer_half_width` — discovery support in \(f\) |
| \(w_{\mathrm{core}}\) | `outer_half_width / 8` by default — densify-center band |
| \(\tau\) | Stop threshold on \(t_{\max}-t_{\min}\) (default \(=w_{\mathrm{core}}\)) |
| **generation** | Number of \(r_g\) shrinks completed (scale index) |
| **fill_pass** | Densify rounds at the **same** \(r_g\) (does **not** advance generation) |
| **core** | Point with \(\lvert f-T\rvert\le w_{\mathrm{core}}\) (densify / bridge center) |
| **frontier** | \(w_{\mathrm{core}} < \lvert f-T\rvert \le w_{\mathrm{outer}}\) (support only) |
| **outer** | \(\mathrm{core}\cup\mathrm{frontier}\) |
| **best** | \(\arg\min_i \lvert f_i-T\rvert\) among finite \(f\) |

---

## 3. Two independent masks (do not confuse)

| Mask | Definition | Role |
|------|------------|------|
| **Absolute outer/core** | \(\lvert f-T\rvert\) vs \(w_{\mathrm{outer}}, w_{\mathrm{core}}\) | Who may **open densify windows** / act as bridge centers |
| **\(t_{\min}, t_{\max}\)** | min/max of finite \(f\) among points inside the ball of radius **\(2\,r_g\)** about **best** | **Generation / exit gate only** — not the densify-center mask |

Why \(2\,r_g\)? Under strict blue-noise spacing \(r_g\), a ball of radius \(r_g\)
around best is essentially a singleton, so its \(f\)-width would always be
zero. Using \(2\,r_g\) measures local target variation at the current scale.

A singleton ball (fewer than 2 finite neighbors in \(2\,r_g\)) yields
\(t_{\min}=t_{\max}=\mathrm{null}\) and **never** counts as converged.

---

## 4. Complete control flow

```
                    ┌─────────────────────────────┐
                    │  Gen-0: global Bridson (r0) │
                    │  evaluate all points → f_i  │
                    └──────────────┬──────────────┘
                                   │
                                   ▼
                    ┌─────────────────────────────┐
              ┌────►│  Classify bands             │
              │     │  core / frontier / outer    │
              │     │  best, t_min, t_max (2 r_g) │
              │     │  brackets A/B/C/D           │
              │     │  active endpoints           │
              │     │  coverage_ok? fill_needed?  │
              │     └──────────────┬──────────────┘
              │                    │
              │         ┌──────────┼──────────┐
              │         ▼          ▼          ▼
              │   fill_needed   converged   advance gen?
              │         │          │          │
              │         ▼          ▼          ▼
              │   root-corr     EXIT ok    solidify cores
              │   endpoints                r_g ← max(r_min, r_g×ρ)
              │   bridge MST               generation += 1
              │   core densify             fill_pass = 0
              │   fill_pass++              reset endpoints
              │   wait feedback            (if r_g already
              │         │                   at floor → stop)
              │         └──────────┐
              └────────────────────┘
```

### 4.1 Generation 0 (global coarse Bridson)

1. Fill the unit cube once with Bridson spacing \(r_0\) (`initial_radius`).
   - \(d\le 4\): Bridson.
   - \(d=5\): Sobol (or equivalent QMC) for gen-0 only.
2. Submit all gen-0 points; wait for feedback; assign finite \(f_i\).
3. Non-finite / failed \(f\) never become best, core, or live.

### 4.2 Classify absolute bands

\[
\begin{aligned}
\mathrm{core} &= \{ i : \lvert f_i - T\rvert \le w_{\mathrm{core}} \}, \\
\mathrm{frontier} &= \{ i : w_{\mathrm{core}} < \lvert f_i - T\rvert \le w_{\mathrm{outer}} \}, \\
\mathrm{outer} &= \mathrm{core}\cup\mathrm{frontier}.
\end{aligned}
\]

- **Only cores** open Bridson densify windows and MST bridge centers.
- **Frontier** is support: brackets, exclusion seeds — never densify centers.
- Empty core after gen-0 is **not** failure: straddling brackets seed the first
  cores via root-correction.

Cores **accumulate** within a fixed \(r_g\); they are **solidified** into a
permanent set only when \(r_g\) shrinks.

### 4.3 Compute \(t_{\min}, t_{\max}\) (exit / next-gen gate)

1. `best = argmin |f−T|`.
2. Collect all finite \(f\) with \(\lVert u - u_{\mathrm{best}}\rVert \le 2\,r_g\).
3. If fewer than 2 values → no band (not converged).
4. Else \(t_{\min}=\min f\), \(t_{\max}=\max f\).
5. Band is “thin” when \(t_{\max}-t_{\min} < \tau\) (default \(\tau=w_{\mathrm{core}}\)).

### 4.4 Brackets and edge classes

On the neighbor graph (Delaunay \(d\le 3\), kNN \(d\ge 4\)) plus a **nearest
opposite-side** query (local sign-change pairs within \(\sim 2.5\,r_g\)):

| Class | Meaning | Action |
|-------|---------|--------|
| **A** | Neither endpoint is core | Root-correction / expand toward \(T\) |
| **B** | Exactly one endpoint is core | Expand from the core along the edge |
| **C** | Both cores, u-gap large | Spatial bridge between cores |
| **D** | Both cores, already close | No fill on this edge |

**Root-correction** places a secant sample on a straddling edge:

\[
u = u_i + \alpha\,(u_j-u_i),\quad
\alpha=\mathrm{clip}_{[0.05,0.95]}\!\left(\frac{T-f_i}{f_j-f_i}\right)
\]

(midpoint fallback if spacing rejects). New points near \(T\) upgrade to core
on the next classify — old points are not “moved.”

### 4.5 Active endpoints (contour tips)

Disconnected or open contour pieces need **directional extension**, not only
local densify around existing cores:

1. Detect geometric **fronts** from straddle / bridge geometry at the current
   scale.
2. Maintain a persistent **active endpoint** set at fixed \(r_g\).
3. Each fill pass probes **full directions** on the hypersphere around each
   active front (omni probes; default \(\ge 16\)).
4. Retire a front only when it is superseded by a farther core, hits the cube
   boundary, or exhausts several distinct attempts (`endpoint_stall_passes`).
5. On \(r_g\) shrink, **rebuild** the endpoint set from the new core geometry
   (do not keep stale fronts from a coarser scale).

### 4.6 Core densify + gap bridge (same \(r_g\))

When cores exist and fill is still needed:

1. **Thin** densify centers if cores are over-dense (proposal centers only;
   all cores still participate in exclusion).
2. For each center \(p\), propose only inside the ball of radius \(r_g\) about
   \(p\), with Bridson-style trials (`k_ref` per center).
3. **Blue-noise separation** \(r_{\mathrm{sep}} = r_g\): reject candidates too
   close to any already accepted / submitted site at this scale.
4. Optional **MST gap bridge** between nearby cores (default on):
   span \(\le\) `bridge_span_factor` \(\times r_g\) (default `2.5`).
5. Mid-pass accepts **never** open new windows until the next classify.
6. Duplicate \(u\) keys are never resubmitted (history is sticky even if a
   point leaves the control band).

### 4.7 Same-scale fill vs generation advance

| Concept | Meaning |
|---------|---------|
| `fill_pass` | One densify / root-corr / endpoint / bridge round at **fixed** \(r_g\) |
| `generation` | One completed \(r_g\) shrink (scale index) |

**Fill continues** while structural progress is still possible:

- new cores appear (until coverage exists),
- best \(\lvert f-T\rvert\) improves,
- contour extent / core NN gap improves,
- actionable A/B/C edges or active endpoints remain.

**Scale quiescence**: after `quiet_fill_passes` (default 3) without structural
progress, treat the scale as filled even if some A/B straddles persist
(straddles are geometry, not a hard terminator).

**Coverage at scale** (`_core_coverage_ok`):

- enough cores (`min_cores_for_coverage`, default 4),
- one connected core component,
- max core–core nearest-neighbor gap \(\le\) `core_spacing_factor` \(\times r_g\)
  (default 2.0).

Until coverage holds, **do not shrink** \(r_g\) — keep appending cores.

### 4.8 Convergence (primary exit)

All of the following must hold:

1. Fill is **not** needed (scale quiet / no actionable fill).
2. Best point is a **core**.
3. Core **coverage** is OK at the current \(r_g\).
4. \(t_{\max}-t_{\min} < \tau\) on the \(2\,r_g\) ball about best.
5. If `final_half_width` is configured, at least one point satisfies
   \(\lvert f-T\rvert \le w_{\mathrm{final}}\).

Then: `stop_reason = converged`.

### 4.9 Radius shrink (next generation)

If fill is done, band is **not** thin yet, and \(r_g > r_{\min}\):

1. **Solidify** current cores into the permanent set.
2. \(r_g \leftarrow \max(r_{\min},\, r_g \times\) `refinement_factor`\()\) (default ×0.5).
3. `generation += 1`, `fill_pass = 0`, reset active endpoints / virtual cores.
4. Continue densify at the finer scale.

If \(r_g\) is already at `min_radius` and the band is still not thin enough,
stop with a resolution-limited reason (do not invent finer samples).

### 4.10 Outer-band anneal

When cores exist and **local nearest brackets = 0**:

\[
w_{\mathrm{outer}} \leftarrow \max\bigl(w_{\mathrm{core}},\,
\rho\, w_{\mathrm{outer}}\bigr)
\quad(\rho=\texttt{outer\_shrink\_factor},\;\mathrm{default}\;0.7).
\]

This tightens discovery support as the contour becomes well sampled. It does
**not** replace the \(t_{\min}/t_{\max}\) exit gate.

### 4.11 Hard safety stops

| Condition | Typical `stop_reason` |
|-----------|------------------------|
| `generation ≥ max_generations` | `max_generations` |
| total points ≥ `max_points` | `max_points` |
| `fill_pass ≥ max_fill_passes` | scale force-close then gate |
| no finite \(f\) at all | hard fail |
| local windows saturated / no candidates | `no_refinement_candidates` |

---

## 5. Public YAML surface (two knobs + target)

New cards should set **only**:

| Key | Role |
|-----|------|
| `target_expression` | sympy over returned observables (+ variable names) |
| `target_value` | level \(T\) |
| `outer_half_width` | discovery band \(\lvert f-T\rvert\le w_{\mathrm{outer}}\) |
| `min_radius` | final u-space resolution floor (Euclidean) |

**Derived** (no YAML required):

- `core_half_width = outer_half_width / 8`
- `threshold = core_half_width`  (\(t_{\max}-t_{\min}\) stop)

**Internal defaults** (override only if you know why):

| Key | Default | Notes |
|-----|---------|--------|
| `initial_radius` | `0.10` | gen-0 + first \(r_g\) |
| `refinement_factor` | `0.5` | \(r_g\) shrink multiplier |
| `bridge_gaps` | `true` | MST reconnection |
| `bridge_span_factor` | `2.5` | max bridge = factor × \(r_g\) |
| `max_generations` | `16` | max \(r_g\) shrinks |
| `max_points` | `50000` | hard budget |
| `max_new_per_generation` | `4000` | densify budget per fill pass |
| `k_ref` | `30` | Bridson trials per densify center |
| `quiet_fill_passes` | `3` | no-progress passes before scale quiet |
| `max_fill_passes` | `max(64, 4·max_generations)` | safety |
| `neighbor_graph` | `auto` | Delaunay (d≤3) / kNN (d≥4) |

Legacy keys (`threshold`, `core_half_width`, `function_tolerance`, …) remain
readable for old cards and checkpoints, but are **not** recommended for new
YAML.

---

## 6. Explicit non-goals

1. **Not** full-domain Bridson every generation with half radius.
2. **Not** crossing-edge midpoint refine as the densify engine.
3. **Not** promoting mid-fill accepts to new densify centers mid-pass.
4. **Not** driving proposals from plot polylines.
5. **Not** treating persistent A/B straddles alone as “never stop filling”
   once the scale is quiet and coverage holds.

---

## 7. Runtime contracts

| Concern | Policy |
|---------|--------|
| Space | Geometry in **u-space**; physical map only for Sample / \(f\) / selection |
| Feedback | `hep:feedback` + wait-all barrier before next classify |
| Checkpoint | Points, \(f\), cores, \(r_g\), generation, fill_pass, seed, pending UUIDs |
| Determinism | Fixed RNG streams per `(generation, fill_pass)`; fixed core order |
| Dimension | \(2\le d\le 5\) |
| Logging (INFO) | start line, \(r_g\) transitions, final summary |
| Logging (DEBUG) | fill_pass band diagnostics, brackets, endpoints, densify timing |

---

## 8. Outputs

| Artifact | Path |
|----------|------|
| Full evaluation archive | `<task_result_dir>/DATABASE/samples.hdf5` |
| CSV export | `Jarvis2 convert <task.yaml>` → `DATABASE/samples.csv` |
| Level-set payload | `<task_result_dir>/levelset.json` |
| SAMPLE shadows | `<task_result_dir>/SAMPLE/…` when calculator `save: true` |

Typical `levelset.json` fields: `algorithm`, `dim`, `target_*`, `r_g`,
`t_min` / `t_max`, `outer_half_width` / `core_half_width`, `n_points_total`,
`generation`, `fill_pass`, `converged`, `stop_reason`, polylines (d=2),
crossing cloud, fidelity.

---

## 9. Decision log (final)

| Decision | Choice |
|----------|--------|
| Geometry | u-space only for Bridson / windows / distances |
| Public knobs | `outer_half_width` + `min_radius` (+ required target) |
| Densify centers | absolute \(\lvert f-T\rvert\le w_{\mathrm{core}}\) |
| Exit gate | \(t_{\max}-t_{\min}\) on **2 \(r_g\)** ball about best |
| Same-scale work | fill_pass++ without advancing generation |
| Generation | one \(r_g\) shrink |
| Mid-pass accepts | constrain exclusion only; not centers until next classify |
| Contour tips | persistent active endpoints + omni probes |
| Gap fill | MST bridge between nearby cores (default on) |
| Blue noise | separation \(= r_g\) on new proposals |
| Radius floor | `min_radius` (default \(1/200\)) |
| Old Voronoi \(t_{\min}/t_{\max}\) live mark | **retired** (inflated fat tubes) |
| Old edge-midpoint refine | **retired** as proposal engine |

---

## 10. Validation notes

Real iDM Vector / Axial AdaptiveBridson scans (microOMEGAs, \(\Omega h^2=0.12\))
confirmed:

- root-correction minting first cores from gen-0 straddles,
- endpoint extension reconnecting broken contour pieces,
- same-\(r_g\) multi-pass fill before shrinking,
- clean stop when the \(2\,r_g\) band is thin and cores cover the contour.

Unit coverage: `tests/test_adaptive_bridson.py`.
