# DESIGN — Physics-Grade Auto Plot Scenes (V2, D15.5)

**Status**: design accepted 2026-07-21; all WPs `todo`
**Date**: 2026-07-21 (maintainer feedback: "自动生成 JarvisPLOT 的 YAML 需要更符合真实物理分析的图")
**Scope**: make the plot YAML that V2 auto-emits after a scan (`plot_scene.py`) produce
figures a physicist would actually put in a talk — using metadata the task YAML
**already contains** and rendering capabilities JarvisPLOT **already ships**. HEP keeps
emitting scenes only; every drawing algorithm stays in JarvisPLOT (D11.5/D12.3 division).

---

## 1. Problem — the evidence is a user's own diff

V2 currently auto-emits one scene shape for every scan: raw `scatter` colored by
`LogL`, plus level-set polylines (AdaptiveBridson) or a dynesty runplot (nested). In
`Jarvis-Examples/iDM/images/` the maintainer's **hand-edited** copy of an auto-generated
scene sits next to the original. The hand edits are exactly the missing physics:

| Hand edit in `iDM-plot.yaml` | Why it was needed | Where V2 already knows this |
|---|---|---|
| `frame.ax.xscale: log`, `yscale: log` | `MChi`/`Y` are **Log-distributed** scan variables; on linear axes the scan collapses into a corner | `Sampling.Variables[].distribution.type: Log` — the generator reads Variables for axis *names* but ignores `distribution.type` |
| `axc.color: {cmap: rainbow, scale: linear, vmin: -3, vmax: 0}` | Without clamping, one failed point at `LogL = -1e300` (or the D13.8 `-inf` convention) flattens the entire colorbar to a single hue | LogL statistics are in the DATABASE the generator itself reads; failed points carry `status: Failed` |
| `style: [a4paper_2x1, rectcmap]` | Colormap style class needed for the colorbar frame | Static — generator emits `a4paper_2x1` only |

Beyond the diff, the auto scenes ignore what JarvisPLOT can already render
(`Figure/method_registry.py`: `contour`/`contourf`/`jpcontour`/`jpcontourf`/`jpfield` +
dedicated `profile_runtime`, `posterior_density_runtime`, `posterior_hpd`,
`interp_2d_runtime` modules):

1. **No profile-likelihood map** — the standard pheno deliverable (2D max-LogL per bin,
   68%/95% CL contours at ΔlnL = −1.15/−3.00) is never emitted, even though PLOT has a
   profile runtime for it.
2. **Nested scans plot dead points as if they were posterior samples** — a raw scatter
   of every dynesty evaluation is *misleading*: density of dead points reflects the
   sampling schedule, not the posterior. The importance weights are already exported
   (`DATABASE/dynesty_result.csv` `log_weight` column) and PLOT has
   `posterior_density`/`posterior_hpd` — nothing connects them.
3. **MCMC scans have no burn-in handling in plots** — `chain_history.csv` (D13.6) has
   `chain_id`/`step`/`accepted`; the scene plots all archive rows including burn-in and
   rejected-proposal evaluations.
4. **No best-fit marker**, no axis limits from `Variables[].parameters.{min,max}`, and
   axis labels are `$<raw column name>$` (`$MChi$` is not LaTeX — physicists write
   `$m_{\chi}$ [GeV]`).

None of this needs new drawing code. It is a **metadata plumbing gap** in
`jarvishep2/plot_scene.py`.

## 2. Goals

1. **Use what the YAML already says.** `distribution.type: Log` → `xscale/yscale: log`;
   `parameters.{min,max}` → axis limits; `Sampling.Variables[].latex` (new **optional**
   key, invariant-1-additive) → axis label, falling back to the raw name.
2. **Robust color scale by default.** Drop `status: Failed` / non-finite LogL rows from
   the plotted CSV view; set `vmin/vmax` from finite-LogL percentiles (default p5/p100),
   overridable via a new optional `Scan.plot` block (§4).
3. **Method-aware scene selection** (the sampler knows what its points mean):
   - fixed-set / AdaptiveBridson → scatter + level-set (today's scene, upgraded per 1–2)
     **plus** a profile-likelihood contour figure;
   - `Dynesty`/`MultiNest` → posterior view weighted by `log_weight` from
     `*_result.csv` (PLOT `posterior_density`/HPD layers) + the existing runplot; raw
     dead-point scatter demoted to an `enable: false` layer for debugging;
   - MCMC family → post-burn-in accepted-chain density (via `chain_history.csv` join)
     + best-fit marker; burn-in fraction from `Bounds` (default 0.3) recorded in the
     scene metadata.
4. **Best-fit star** on every 2D figure (max finite LogL row).
5. **Emitted YAML stays stock-jplot** — consumable by `Jarvis2 plot` / `jplot`
   unchanged; `_jarvishep2` metadata block records the choices made (percentiles,
   burn-in, weight column) so users can see and override them.
6. **Hand-edit friendliness**: never overwrite a user-modified scene — if the output
   file exists and its `_jarvishep2.producer_hash` doesn't match, write `<name>.new.yaml`
   beside it instead (the iDM workflow above shows users *do* edit these files).

### Non-goals

- Any rendering/statistics code in HEP beyond selecting columns/rows and computing
  percentiles & the best-fit row (contours, KDE, HPD stay in JarvisPLOT).
- Corner plots / 1D marginal batteries — that is `Jarvis2 analyze` (D15.2), which
  should reuse the same scene-synthesis helpers this WP builds.
- Changing existing scene file names or the `Jarvis2 plot` CLI.

## 3. Design

One new module-internal layer in `plot_scene.py` (or `plot_scene_meta.py`):

```
SceneMeta.from_run(config, task_result_dir)
  .variables      # name, latex?, dist type, min/max  ← Sampling.Variables
  .method_family  # fixed | adaptive | nested | mcmc  ← Distributor registration
  .logl_stats     # finite percentiles, best-fit row  ← DATABASE scan (one pass)
  .weights        # nested: csv + log_weight col; mcmc: chain_history + burn_in
```

Scene builders take `SceneMeta` and emit frames/layers; the three existing emitters
(`emit_jplot_scan_levelset_yaml`, `emit_jplot_dynesty_runplot_yaml`,
`emit_plot_scenes_from_run`) become thin dispatchers. New optional YAML surface:

```yaml
Sampling:
  Variables:
    - name: MChi
      latex: "m_{\\chi}\\,[\\mathrm{GeV}]"   # optional; label fallback = name
      distribution: {type: Log, parameters: {min: 1.0, max: 1000.0}}

Scan:
  plot:                      # optional block, all defaults sane
    color: {key: LogL, cmap: rainbow, percentile: [5, 100]}  # or vmin/vmax
    profile_contours: [0.68, 0.95]
    burn_in: 0.3             # mcmc scenes only
    figures: [scatter, profile, posterior]   # subset selection
```

## 4. Work packages

| WP | Title | Depends on | Accept |
|---|---|---|---|
| D15.5 | `SceneMeta` + axis/scale/label/limit plumbing + robust color clamp + best-fit marker + no-clobber | — | regenerating the iDM scene reproduces the maintainer's hand edits (log axes, clamped colorbar) **without** hand edits; failed/−inf rows excluded; existing scene tests stay green; hand-edited file is never overwritten |
| D15.6 | Method-aware scenes: nested posterior-weighted view + MCMC post-burn-in view + profile-likelihood contour figure | D15.5 | Eggbox Dynesty run auto-emits a weighted posterior figure (dead-point scatter `enable: false`); DRAM run emits post-burn-in density + best-fit; AdaptiveBridson emits scatter+levelset+profile figures; all render through stock `jplot` |
| D15.7 | `Scan.plot` block + `Variables[].latex` in validation contracts + YAML_REFERENCE §§ | D15.5 | `Jarvis2 validate` accepts/checks the new optional keys; docs list every default |

**Rollback**: no `Scan.plot` block + deleting the new keys ⇒ scenes degrade to today's
shape; emitters keep their names and call sites. **Out of scope**: corner plots
(D15.2), new PLOT drawing algorithms, scene regeneration CLI.

## 5. Risks

1. **Method-family detection drift** — derive from the Distributor registration
   (`stateless`, class) rather than string-matching method names.
2. **Weight-column contract** — nested CSV schema is already frozen by D13.6
   (`DATABASE_NESTED_RESULT_COLUMNS`); the posterior scene must read through that
   constant, not re-hardcode column names.
3. **PLOT capability mismatch** — D15.6 must smoke-render every emitted scene via
   `jplot` in tests (render-or-fail), not just YAML-validate it.
