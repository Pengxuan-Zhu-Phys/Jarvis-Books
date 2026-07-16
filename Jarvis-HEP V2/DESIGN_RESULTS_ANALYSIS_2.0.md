# DESIGN — Result Reuse & Analysis Closure (V2, D15)

**Status**: design accepted 2026-07-16; no code yet — all WPs `todo`
**Date**: 2026-07-16
**Scope**: stop paying for repeated calculator evaluations (warm-start / point cache),
close the post-run analysis loop (corner plots, best-fit tables, posterior weighting via
JarvisPLOT), and finish the Portal format surface (File / HepMC / LHE).
**Maintainer constraint**: **D8 stays parked**.

---

## 1. Problem

Every V2 sample pays full calculator cost even when an identical point was already
computed in a previous scan (slow regime: minutes per point). And a finished scan ends
at `samples.hdf5` + flowchart — users leave the toolchain for corner plots, best-fit
extraction, and posterior work, while D13's MCMC/nested outputs will make those the
default asks. Portal still lacks the File/HepMC/LHE outputs V1 exposed.

## 2. Goals

1. **Warm-start**: `Jarvis2 run task.yaml --warm-start <old DATABASE|scan dir>` — a
   proposed point whose cache key matches a prior record adopts its observables/LogL
   without touching calculators; the record is marked `cached_from` for traceability.
2. **Honest cache keys**: hash of (physical param values rounded to a declared
   precision, per-module config fingerprint: commands + inputs + env_setup sources +
   calculator `source` version note). Any module fingerprint mismatch ⇒ cache miss.
   No silent reuse across changed physics.
3. **`Jarvis2 analyze <scan>`**: corner plot (all pairs or `--vars`), 1D marginals,
   best-fit / MAP table (txt + json), posterior weights (nested: importance weights;
   MCMC: post-burn-in chain) — all **rendered by JarvisPLOT** (V2 emits scene YAML +
   data, PLOT owns every drawing algorithm, same division as D11.5/D12.3).
4. **Portal completion**: expose File / HepMC / LHE adapters through
   `jarvis_portal.v2` (Portal-side release, like the 1.4.0 SLHA work), then fixture
   them in V2. Adding a format stays a Portal upgrade, never a HEP release.
5. Everything additive: no output-contract changes; new DATABASE columns/attrs only.

### Non-goals

- Surrogate/emulator-based active learning (candidate for a later milestone, pairs
  with D13's `dnn` heritage).
- Web dashboard (revisit after D14; terminal monitor stays the only live view).
- Cross-scan DATABASE *merging* tools (warm-start reads, never rewrites, old scans).

## 3. Design notes

- **Cache lookup lives in the control process** at proposal time (before Redis push):
  hits short-circuit into `submit_result`-equivalent records so stats/op_count stay
  honest (`cached` counted separately in `run_summary` — never inflates `completed`
  throughput silently).
- **Precision contract**: `Sampling.Variables[].cache_decimals` (default: exact float
  repr) — documented in YAML_REFERENCE; float-noise duplicates are the user's explicit
  opt-in, not a heuristic.
- **`analyze` reads only the DATABASE** (+ levelset/chain attrs) — it never touches
  Redis or a live scan; safe on finished/archived scans and inside `Jarvis2 project`
  packs.
- D13 coupling: posterior weighting needs the chain/weight columns D13.6 defines;
  `analyze` ships a reduced mode (profiles + best-fit only) when they are absent, so
  D15.2 does not block on D13.

## 4. Work packages

| WP | Title | Depends on | Accept |
|---|---|---|---|
| D15.1 | Point cache + `--warm-start` | — | re-running a finished Eggbox scan with `--warm-start` completes with 0 calculator subprocess launches; changing one module command ⇒ full re-compute; `cached` metric in run_summary |
| D15.2 | `Jarvis2 analyze`: corner/marginals/best-fit via JarvisPLOT scenes | — | analyze on the Eggbox golden produces corner PNG + best-fit table; reduced mode without chain columns; no drawing code in HEP |
| D15.3 | Posterior weighting mode | D13.6, D15.2 | nested-run evidence/weights and MCMC post-burn-in posteriors reproduce V1 golden analysis numbers |
| D15.4 | Portal File/HepMC/LHE exposure + V2 fixtures | Portal release | `Jarvis2 portal formats` lists them; one real fixture per format in `tests/` |

**Rollback**: no `--warm-start`/`analyze` invocation ⇒ zero behavior change.
**Out of scope**: emulators, web UI, DATABASE merging, D8 surfaces.

## 5. Risks

1. **Stale-cache physics** is the only dangerous failure mode here — hence the
   fingerprint-miss-by-default rule and the visible `cached_from` provenance column.
2. **Analyze scope creep** — bounded by "JarvisPLOT owns rendering; HEP emits scenes";
   any new plot type is a PLOT feature request, not a HEP WP.
