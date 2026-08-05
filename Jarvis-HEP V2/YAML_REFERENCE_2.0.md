# Jarvis-HEP V2 — Task YAML Reference (As-Built)

**Status**: as-built reference; core inventory originally pinned at `d0de31a`; integration/UI
corrections reviewed at `6b9841e` (2026-07-13); expression unification + `EnvReqs.V2` surface
pinned at `0a5e85e` (2026-07-14, 283 passed / 1 skipped)
**Date**: 2026-07-03 · reviewed and corrected 2026-07-04 (Appendix A.11–A.13) ·
restructured 2026-07-05 · EnvReqs/expression/ALS sync 2026-07-14
**Purpose**: the complete, code-derived inventory of every YAML key the V2 runtime actually
reads — including defaults, valid values, aliases, and validation behavior. Use this document
to review whether the YAML interaction design is complete and coherent.
**Companion docs**: [`DESIGN_2.0_DISTRIBUTED.md`](DESIGN_2.0_DISTRIBUTED.md) (architecture),
[`components/config_schema.md`](components/config_schema.md) (loader internals),
[`DESIGN_PRINCIPLES_REVIEW_2.0.md`](DESIGN_PRINCIPLES_REVIEW_2.0.md) (code-quality review,
including the `Distributor`/IO-type registry work that will change some of the "hard error"
behavior documented in §13).

> Everything below is derived from the code, not from the design sketches. Keys that appear in
> the design docs but are **not** implemented are listed in Appendix A, not in the main body.
> Appendix A/B/C item numbers are cross-referenced from other Workshop docs — **never
> renumber them**; append new findings instead.

## Notation used throughout

- **Bold "yes"** in a Required column = omitting the key raises an exception (type named in
  the same cell). Plain "no" = a default applies silently.
- `Key` in backticks is the literal YAML key spelling, case-sensitive, exactly as the code
  reads it (V2 has real case/spelling inconsistencies — that's documented, not normalized
  away; see Appendix A.5).
- `file.py:123` citations are `jarvishep2/file.py` line numbers at the pinned commit above;
  re-verify against `git blame` if the commit has moved on.
- "Phase 1" / "Phase 2" refer to the two-stage token resolution described in §12.

---

## Table of Contents

1. [How the YAML Is Loaded](#1-how-the-yaml-is-loaded)
2. [Full Annotated Skeleton](#2-full-annotated-skeleton)
3. [Key Index](#3-key-index) — flat lookup, every key path in one table
4. [Top-Level Keys](#4-top-level-keys)
5. [`EnvReqs.V2` runtime settings](#5-envreqsv2-runtime-settings)
6. [`Sampling`](#6-sampling)
7. [The u→x mapper (`Sampling.Mapper` + auto-derived)](#7-the-ux-mapper-samplingmapper--auto-derived)
8. [`LibDeps`](#8-libdeps)
9. [`Calculators`](#9-calculators)
10. [`Operas`](#10-operas)
11. [`Likelihood` → `Sampling.LogLikelihood`](#11-likelihood--samplingloglikelihood)
12. [Token Reference](#12-token-reference)
13. [Validation & Failure Behavior](#13-validation--failure-behavior)
- [Appendix A — Known Gaps & Design Warts](#appendix-a--known-gaps--design-warts-input-for-the-yaml-design-review)
- [Appendix B — Internal Worker-Config Keys (not YAML)](#appendix-b--internal-worker-config-keys-not-yaml)
- [Appendix C — CLI Surface](#appendix-c--cli-surface-for-completeness)

---

## 1. How the YAML Is Loaded

`Jarvis2 <task>.yaml` → `task_config.load_task_yaml()` runs, in order:

1. `yaml.safe_load` — the top level **must** be a mapping (else `ValueError`).
2. Project root uses `JARVIS_HEP_TASK_ROOT` / `JHEP_TASK_ROOT` when set; otherwise it is
   inferred by walking up from the YAML's directory looking for a
   `.jarvis-project.json` or `jarvis.project.yaml` marker (falls back to a `bin/` parent, then
   the YAML directory itself).
3. Scan name = `Scan.name` → `scan_name` → `"default"`.
4. Output root = `task_result_dir` (top-level key) → `<project_root>/outputs/<scan_name>`.
5. `EnvReqs.V2` is merged with the optional shared default file, then normalized. Redis,
   sample-artifact policy, and Worker recovery are internal defaults rather than YAML knobs.
6. These derived keys are **stamped into the config dict** (reserved names; do not use them as
   your own keys): `task_yaml`, `task_root`, `project_root`, `task_result_dir`, `scan_name`.

```
task.yaml
   │  yaml.safe_load
   ▼
raw dict ──► infer_project_root() ──► scan_name resolution ──► task_result_dir
   │                                                                  │
   ▼                                                                  ▼
normalize_runtime_block()                          config["task_result_dir"] = …
   │                                                    (+ task_yaml/task_root/
   ▼                                                     project_root/scan_name stamped)
config["Runtime"] = {mode=redis, workers, batch_size,
                      internal artifact/recovery defaults}
```

Phase-1 static token resolution (`&J/`, `${LibDeps:…}`, `${Scan:…}`, registered executable
names) runs once in the control process; per-Sample tokens (`@SampleID`, `@Sdir`, `@PackID`)
resolve inside the Worker (Phase 2). See §12.

---

## 2. Full Annotated Skeleton

Every key the runtime reads, with defaults in comments. All blocks except `Sampling` are
optional.

```yaml
# ---- identity -------------------------------------------------------------
# The ONLY accepted top-level keys are:
#   Scan, EnvReqs, Sampling, LibDeps, Calculators, Operas
# project_name / scan_name / task_result_dir / run_id / Mapper / Likelihood are
# REJECTED by the schema (JV2-SCH-001) — see §4.
Scan:
  name: scan-01                  # str; drives outputs/<name> and ${Scan:name}

# ---- V1-compatible default-settings entry point -------------------------
EnvReqs:
  Check_default_dependencies:
    required: true
    default_yaml_path: "&J/deps/environment_default.yaml"

# ---- V2 scheduling + SAMPLE/archive policy --------------------------------
# `worker` is accepted as a singular alias; use `workers` in new YAML.
# Nested sample_directory / cleanup / archiver may also live under EnvReqs.V2
# (merged from default_yaml) and are applied into Scan / Calculators.*.
EnvReqs:
  V2:
    workers: 4                   # int >= 0; default 0 (factory uses 1 when <= 0)
    batch_size: 256               # int > 0; default 256, sampler submit-group size
    sample_directory:            # optional; also Scan.sample_directory
      enabled: true
      limit: 200
      width: 6
      pack: true
      start_bucket: 1
    cleanup:
      strategy: direct           # direct (default) | mv_to_staging
    archiver:
      mode: process              # process (default) | thread
      handoff: direct
      pack_buckets: true
      batch_size: 200
      flush_interval_sec: 1.0
      delete_after_archive: true

# ---- sampling (required for a runnable task) --------------------------------
Sampling:
  Method: Bridson                # Bridson | Random | Grid | CSV | AdaptiveBridson
  # mode: check_modules          # special task type; replaces Method (needs `data`)
  # data: "&J/points.csv"        # check_modules input CSV (alias: points_csv)
  selection: "x + y < 1.5"       # optional sympy bool expr over physical params
  Variables:                     # required by Bridson/Random/Grid/ALS (not CSV)
    - name: x
      description: flat x        # informational only
      distribution:
        type: Flat               # Flat | Log | Normal | Log-Normal | Logit
        parameters:              # per type:
          min: 0.0               #   Flat/Log: min, max
          max: 1.0               #   Normal/Log-Normal: mean, stddev
                                 #   Logit: location (0), scale (1)
          # num: 20              #   Grid only: REQUIRED points per dimension
          # length: 1.0          #   Bridson only: u-space box edge — REQUIRED here
                                 #   (KeyError if absent; inconsistent elsewhere, see A.11)
  # -- per-method knobs: ALL of them live under Bounds (see §6.1) --
  Bounds:                        # REQUIRED for Bridson / Random / CSV / AdaptiveBridson
    Radius: 0.35                 #   Bridson: REQUIRED minimum point distance (u-space)
    MaxAttempt: 30               #   Bridson: REQUIRED k candidates per active point
    # MaxWorker: 4               #   Bridson: max in-flight proposals (default EnvReqs.V2.workers)
    # Seed: 0                    #   Bridson / Random / Grid (alias: seed)
    # "Point number": 500        #   Random: REQUIRED sample count (alias: point_number)
    # path: "&J/points.csv"      #   CSV: REQUIRED; &J/, absolute, or task-YAML-relative
    # variables: [x, y]          #   CSV: column subset (default: all non-uuid columns)
    # uuid_column: uuid          #   CSV: default "uuid"; missing column -> fresh uuid4 per row
    # delimiter: ","             #   CSV: default ","
    # encoding: utf-8            #   CSV: default utf-8
    # target_expression: "LogL"  #   AdaptiveBridson: REQUIRED sympy over returned observables
    # target_value: -2.9957      #   AdaptiveBridson: REQUIRED level-set constant
    # initial_radius: 0.08       #   AdaptiveBridson: gen-0 Bridson r0 + first live windows
    # refinement_factor: 0.5     #   AdaptiveBridson: spacing r' = r_g * factor; next r_g *= factor
    # threshold: 0.05            #   AdaptiveBridson: stop when (t_max - t_min) < threshold
    # max_generations: 25        #   AdaptiveBridson: optional gen cap (with/instead of threshold)
    # max_points: 5000           #   AdaptiveBridson: hard sample budget
  # LogLikelihood:               # alias of Likelihood.expressions (lower precedence)
  #   - name: LogL_Z
  #     expression: z

# ---- u -> x mapper (§7) ----------------------------------------------------
# Default: auto-derived from Sampling.Variables (distribution → params).
# Optional explicit reparameterization (list of {name, expression}):
#   Sampling:
#     Mapper:
#       - name: x
#         expression: "cos(t)"
#       - name: y
#         expression: "sin(t)"
# Top-level `Mapper:` (outside Sampling) is still rejected.

# ---- external tool registry -------------------------------------------------
LibDeps:
  path: "&J/External"            # base dir for module paths (default project root)
  Modules:
    - name: SPheno               # defines ${LibDeps:SPheno}
      path: "&J/External/SPheno" # or source:, or installation: {path:|source:}
                                 # fallback: <LibDeps.path>/<name>
  registered_executables:
    - name: eggboxlk             # bare name becomes callable in any command string
      source: "${LibDeps:SPheno}/bin/eggbox"   # REQUIRED; must exist at load time
      resolution: direct_path    # direct_path | symlink (default direct_path)

# ---- calculators (external programs) ----------------------------------------
Calculators:
  Pools:                         # global concurrency slots per calculator (alias: pools)
    EggBox: 4                    # Redis calc:free:<name>; floor 1
                                 # fallback per-module `make_paraller`, else 1
  Archiver:
    mode: process                # process (default) | thread
    handoff: direct              # direct (default) | staging
    pack_buckets: true           # tar SAMPLE/<bucket> after archived==assigned
    batch_size: 200              # default 200, must be > 0
    flush_interval_sec: 1.0      # default 1.0, floor 0.05
    strategy: move               # move | copy (default move) — used if staging hop on
    delete_after_archive: true   # default true
  Cleanup:
    strategy: direct             # direct (default) | mv_to_staging
    staging_dir: null            # only for mv_to_staging; default <task_result_dir>/staging
  Modules:
    - name: EggBox               # REQUIRED (nameless entries are dropped)
      required_modules: []       # dependency names -> execution layers (same layer = concurrent)
      clone_shadow: false        # true = physical per-pack copy (non-thread-safe tools)
      path: "&J/calc/eggbox"     # runtime dir; may contain @PackID when clone_shadow
      # source: "&J/src/eggbox"  # clone_shadow: copytree source (when no installation)
                                 # non-shadow: symlinked into the Sample dir
      # symlink_name: eggbox     # link name in the Sample dir (default: module name)
      # timeout: 600             # seconds; whole-execute deadline (default: none)
      # make_paraller: 2         # per-module pool fallback (sic — V1 typo, see A.4)
      env_setup:
        - source: "&J/External/rivet_env.sh"   # sourced once per Worker, env cached
      installation:              # clone_shadow install commands (once per Worker per pack)
        - cmd: "cp -r ${source}/* ${path}/"    # ${source}/${path} tokens available
          cwd: "${path}"
      # NOTE: PackIDs are stable slots (001…N) whose directories persist across
      # samples, Workers, and runs, so installation commands MUST be idempotent
      # (plain overwrite-copy like the above is fine; V1 cards already assume this).
      #
      # SINCE f106b65 — install REUSE: a pack keeps `.jarvis_install_stamp.json`
      # and `installation` is **skipped** when the stored fingerprint still
      # matches (module name, basepath, source path + root stat, command list).
      # The fingerprint intentionally does NOT scan the source tree for content
      # edits. After editing source, set `reinstall: true` once in
      #     <calculator folder>/jarvis_install.json     (outside PackID dirs)
      # and rerun; the control process bumps a monotone reinstall_epoch and
      # every pack reinstalls exactly once. The control file is control-process
      # owned; Workers write only `.jarvis_install_stamp.json` inside their pack.
      # The compatible all-module emergency switch is:
      #     JARVIS_FORCE_CALC_INSTALL=1 Jarvis2 run card.yaml
      # Design: `DESIGN_CALC_INSTALL_CONTROL_2.0.md`.
      initialization:            # post-install commands (once per pack)
        - cmd: "./configure"
          cwd: "${path}"
      execution:
        path: "&J/calc/eggbox"   # fallback for module path
        commands:
          - cmd: python3 eggboxlk        # registered names + all tokens resolve here
            cwd: "@Sdir"
        input:
          - name: inpjson
            path: "@Sdir/input.json"
            type: JSON           # via Jarvis-Portal; also CSV/TSV/DAT/Wolfram (unknown types hard-fail)
            actions:
              - type: Dump       # ONLY "Dump" is implemented
                variables:
                  - name: xx
                    expression: x * Pi   # sympy over params; Pi/pi available
                  - name: yy             # no expression -> copy param `yy` verbatim
        output:
          - name: oupjson
            path: "@Sdir/output.json"
            type: JSON
            variables:
              - name: z
                entry: z         # dotted path into the JSON (default: name)

# ---- operas (in-process Python operators) -----------------------------------
Operas:
  # make_paraller: 16           # accepted V1 key; V2 Worker concurrency does not use it
  Modules:
    - name: TrivialEggbox
      operator: helper.eggbox2d  # REQUIRED; importlib dotted callable or Jarvis-Operas registry name
      call_mode: call            # intended: call | acall; currently not strictly validated
      # timeout_sec: 30          # optional (alias: timeout); thread-based timeout
      # kwargs: {shift: 0.5}     # static extra kwargs
      input:                     # optional; default = pass all observables through
        - x                      # str form: pass observable x
        - name: r                # mapping form: computed input
          expression: x + y      #   shared V1-compatible expression language; catalog below
        - name: alias
          entry: some.dotted.key #   or copy via dotted path
      output:                    # REQUIRED to capture anything (unlisted keys dropped)
        - name: z
          entry: z               # dotted path into the returned dict

# ---- likelihood -----------------------------------------------------------
# Top-level `Likelihood:` is REJECTED. Likelihood terms live under Sampling:
#
#   Sampling:
#     LogLikelihood:
#       - name: LogL_Z
#         expression: LogGauss(z, 1.0, 0.1)   # shared 38-function Expression Core
#     # a term literally named LogL is THE total; otherwise LogL = sum of terms
```

---

## 3. Key Index

Flat lookup — every key path documented in this file, with its section and required/default
at a glance. Use this table to jump straight to a key; use §§4–11 for the full behavioral
detail (error types, aliases, code citations).

### 3.1 `EnvReqs.V2`

| Path | §  | Required | Default |
|---|---|---|---|
| `EnvReqs.V2.workers` / `worker` | [5](#5-envreqsv2-runtime-settings) | no | `0` |
| `EnvReqs.V2.batch_size` | [5](#5-envreqsv2-runtime-settings) | no | `256` |
| `EnvReqs.V2.sample_directory` | [5.1](#51-sample_directory-also-scansample_directory) | no | limit 200 / pack true |
| `EnvReqs.V2.cleanup` | [9.3](#93-calculatorscleanup) | no | `direct` |
| `EnvReqs.V2.archiver` | [9.2](#92-calculatorsarchiver) | no | process + pack |

### 3.2 `Sampling`

| Path | §  | Required | Default |
|---|---|---|---|
| `Sampling.Method` | [6](#6-sampling) | for a live scan | — |
| `Sampling.mode` | [6.7](#67-check_modules-jarvis2-check--samplingmode-check_modules) | no | — |
| `Sampling.data` / `Sampling.points_csv` | [6.7](#67-check_modules-jarvis2-check--samplingmode-check_modules) | yes (check_modules) | — |
| `Sampling.Bounds.Seed` / `Sampling.Bounds.seed` | [6.1](#61-keys-common-to-bridson--random--grid) | no | `0` |
| `Sampling.selection` | [6.1](#61-keys-common-to-bridson--random--grid) | no | — |
| `Sampling.Mapper[].name` / `expression` | [7](#7-the-ux-mapper-samplingmapper--auto-derived) | no | — |
| `Sampling.Variables[].name` | [6.2](#62-variables-entry--distribution-types) | yes | — |
| `Sampling.Variables[].description` | [6.2](#62-variables-entry--distribution-types) | no | — |
| `Sampling.Variables[].distribution.type` | [6.2](#62-variables-entry--distribution-types) | yes | — |
| `Sampling.Variables[].distribution.parameters.{min,max}` | [6.2](#62-variables-entry--distribution-types) | yes (Flat/Log) | — |
| `Sampling.Variables[].distribution.parameters.{mean,stddev}` | [6.2](#62-variables-entry--distribution-types) | yes (Normal/Log-Normal) | — |
| `Sampling.Variables[].distribution.parameters.{location,scale}` | [6.2](#62-variables-entry--distribution-types) | yes (Logit) | — |
| `Sampling.Variables[].distribution.parameters.{n,p}` | [6.2](#62-variables-entry--distribution-types) | yes (Binomial) | — |
| `Sampling.Variables[].distribution.parameters.lambda` | [6.2](#62-variables-entry--distribution-types) | yes (Poisson) | — |
| `Sampling.Variables[].distribution.parameters.{alpha,beta}` | [6.2](#62-variables-entry--distribution-types) | yes (Beta) | — |
| `Sampling.Variables[].distribution.parameters.rate` | [6.2](#62-variables-entry--distribution-types) | yes (Exponential) | — |
| `Sampling.Variables[].distribution.parameters.{shape,scale}` | [6.2](#62-variables-entry--distribution-types) | yes (Gamma) | — |
| `Sampling.Variables[].distribution.parameters.num` | [6.5](#65-grid) | **yes (Grid)** | — |
| `Sampling.Variables[].distribution.parameters.length` | [6.3](#63-bridson) | **yes (Bridson)** | — |
| `Sampling.LogLikelihood` | [6.8](#68-likelihood-expression-semantics) | no | — |
| `Sampling.Bounds.Radius` | [6.3](#63-bridson) | **yes** | — |
| `Sampling.Bounds.MaxAttempt` | [6.3](#63-bridson) | **yes** | — |
| `Sampling.Bounds.MaxWorker` | [6.3](#63-bridson) | no | `EnvReqs.V2.workers` |
| `Sampling.Bounds."Point number"` / `point_number` | [6.4](#64-random) | **yes** | — |
| `Sampling.Bounds.path` | [6.6](#66-csv) | **yes** | — |
| `Sampling.Bounds.variables` | [6.6](#66-csv) | no | all non-uuid columns |
| `Sampling.Bounds.uuid_column` | [6.6](#66-csv) | no | `uuid` |
| `Sampling.Bounds.delimiter` | [6.6](#66-csv) | no | `,` |
| `Sampling.Bounds.encoding` | [6.6](#66-csv) | no | `utf-8` |
| `Sampling.Bounds` (AdaptiveBridson knobs) | [6.9](#69-adaptivebridson) | **yes** (ALS) | — |
| `Sampling.Bounds.target_expression` | [6.9](#69-adaptivebridson) | **yes** | — |
| `Sampling.Bounds.target_value` | [6.9](#69-adaptivebridson) | **yes** | — |
| `Sampling.Bounds.outer_half_width` | [6.9](#69-adaptivebridson) | no | `0.02` (public) |
| `Sampling.Bounds.min_radius` | [6.9](#69-adaptivebridson) | no | `1/200` (public) |
| `Sampling.Bounds.initial_radius` | [6.9](#69-adaptivebridson) | no | `0.10` |
| `Sampling.Bounds.refinement_factor` | [6.9](#69-adaptivebridson) | no | `0.5` |
| `Sampling.Bounds.threshold` | [6.9](#69-adaptivebridson) | no | default `outer/8` (legacy) |
| `Sampling.Bounds.core_half_width` | [6.9](#69-adaptivebridson) | no | default `outer/8` (legacy) |
| `Sampling.Bounds.max_generations` | [6.9](#69-adaptivebridson) | no | `16` (max \(r_g\) shrinks) |
| `Sampling.Bounds.max_points` | [6.9](#69-adaptivebridson) | no | `50000` |
| `Sampling.Bounds.max_new_per_generation` | [6.9](#69-adaptivebridson) | no | `4000` |
| `Sampling.Bounds.k_ref` | [6.9](#69-adaptivebridson) | no | `30` |
| `Sampling.Bounds.neighbor_graph` | [6.9](#69-adaptivebridson) | no | `auto` |
| `Sampling.Bounds.knn_k` | [6.9](#69-adaptivebridson) | no | `4 * d` |
| `Sampling.Bounds.function_tolerance` | [6.9](#69-adaptivebridson) | no | alias of `threshold` (compat) |

### 3.3 `LibDeps` / `Calculators` / `Operas` / `Likelihood`

> **Top-level** `Mapper` is rejected. Optional **`Sampling.Mapper`** is a flat
> list of `{name, expression}` (D22) — see [§7](#7-the-ux-mapper-samplingmapper--auto-derived).
> Omitted → distribution-only auto mapper (backward compatible).

| Path | §  | Required | Default |
|---|---|---|---|
| `LibDeps.path` | [8.1](#81-libdepsmodules--shared-install-once-libraries) | no | project root |
| `LibDeps.make_paraller` | [8.1](#81-libdepsmodules--shared-install-once-libraries) | no | `1` |
| `LibDeps.Modules[].name` | [8.1](#81-libdepsmodules--shared-install-once-libraries) | yes | — |
| `LibDeps.Modules[].{required_modules,installed,path,source,installation}` | [8.1](#81-libdepsmodules--shared-install-once-libraries) | no | `<LibDeps.path>/<name>` |
| `LibDeps.registered_executables[].name` | [8.2](#82-libdepsregistered_executables) | yes | — |
| `LibDeps.registered_executables[].source` | [8.2](#82-libdepsregistered_executables) | yes | — |
| `LibDeps.registered_executables[].resolution` | [8.2](#82-libdepsregistered_executables) | no | `direct_path` |
| `Calculators.Pools.<name>` / `pools.<name>` | [9.1](#91-calculatorspools-alias-pools) | no | per-module `make_paraller` or `1` |
| `Calculators.Archiver.*` | [9.2](#92-calculatorsarchiver) | no | see table |
| `Calculators.Cleanup.*` | [9.3](#93-calculatorscleanup) | no | see table |
| `Calculators.Modules[].name` | [9.4](#94-calculatorsmodules--one-external-calculator) | yes | — |
| `Calculators.Modules[].{required_modules,clone_shadow,path,source,symlink_name,timeout,make_paraller,env_setup,installation,initialization}` | [9.4](#94-calculatorsmodules--one-external-calculator) | no | see table |
| `Calculators.Modules[].execution.*` | [9.5](#95-execution-block) | yes in practice | see table |
| `Calculators.Modules[].modes[]` | [9.6](#96-multi-mode-calculator-shared-packid) | no | shared-only logical modes |
| `Operas.Modules[].name` | [10](#10-operas) | effectively yes | `Operas<i>` |
| `Operas.Modules[].operator` | [10](#10-operas) | **yes** | — |
| `Operas.Modules[].{call_mode,timeout_sec,timeout,kwargs,input,output}` | [10](#10-operas) | no | see table |
| `Sampling.LogLikelihood[].name` | [11](#11-likelihood--samplingloglikelihood) | no | `"LogL"` |
| `Sampling.LogLikelihood[].expression` | [11](#11-likelihood--samplingloglikelihood) | **yes** | — |

---

## 4. Top-Level Keys

**Exactly six top-level keys are accepted.** Anything else is a hard `JV2-SCH-001`:

```
Allowed keys: Calculators, EnvReqs, LibDeps, Operas, Sampling, Scan
```

| Key | Type | Default | Consumed by |
|---|---|---|---|
| `Scan` | map | — | `name` is the only key read |
| `EnvReqs` | map | — | V1 requirements plus V2 scheduling/defaults (§4.1, §5) |
| `Sampling` | map | — | sampler selection + config (§6) |
| `LibDeps` | map | — | token paths + registered executables (§8) |
| `Calculators` | map | — | modules, pools, archiver, cleanup (§9) |
| `Operas` | map | — | `Modules` list (§10) |

**Rejected at the top level** (measured with `Jarvis2 validate` on `da33371`; earlier
revisions of this section wrongly documented all six as supported):

| Key | Diagnostic suggestion | Where it went |
|---|---|---|
| `Likelihood` | "Move its expressions to `Sampling.LogLikelihood`; top-level `Likelihood` is not a V2 interface." | [§11](#11-likelihood--samplingloglikelihood) |
| `Mapper` (top-level) | "Remove top-level `Mapper`: it is not a V2 task-card interface." | Optional **`Sampling.Mapper`** only — [§7](#7-the-ux-mapper-samplingmapper--auto-derived) |
| `project_name` | "Remove top-level `project_name`: it is not a V2 task-card interface." | `Scan.name` is the scan identity |
| `scan_name` | generic unexpected-key error | `Scan.name` |
| `task_result_dir` | generic unexpected-key error | derived by the loader (`task_config.py:483`) from the project root + scan name |
| `run_id` | generic unexpected-key error | uuid4 per run (`core.py:420`); not authorable |
| `Utils` | "…`Utils.interpolations_1D` moved to Jarvis-Operas: use `interp1.*` …" | `Operas` |

Reserved (stamped by the loader onto the in-memory config, never authored in the card):
`task_yaml`, `task_root`, `project_root`, `task_result_dir`, `scan_name`, `run_id`.

---

### 4.1 `EnvReqs`: V1-compatible V2 default settings

V1 projects already use this exact entry point:

```yaml
EnvReqs:
  Check_default_dependencies:
    required: true
    default_yaml_path: "&J/deps/environment_default.yaml"
```

V2 retains it unchanged. When `required` is true, the loader resolves
`default_yaml_path` with the normal `&J` rules, loads the referenced YAML, and reads only
`EnvReqs.V2` from it. Those defaults are recursively merged with the task's own
`EnvReqs.V2`, so the task value always wins. `required: false` skips the file.

For example, a shared `deps/environment_default.yaml` can keep the V2 runtime profile beside
the existing V1 dependency declarations:

```yaml
EnvReqs:
  # Existing V1 dependency requirements remain here unchanged.
  V2:
    workers: 1
    batch_size: 256
```

`EnvReqs.V2` must be a mapping. A required but missing or malformed default file fails during
YAML load with a focused error. V1 environment requirement keys such as `OS`, `Python`, and
`CERN_ROOT` continue to be preserved as compatibility metadata; V2 does not validate them at
load time.

---

## 5. `EnvReqs.V2` runtime settings

Public V2 scheduling + SAMPLE/archive policy knobs (also mergeable from
`EnvReqs.Check_default_dependencies.default_yaml_path`). Nested maps are applied into
`Scan.sample_directory` / `Calculators.Cleanup` / `Calculators.Archiver` (task values win).

| Key | Values | Default | Notes |
|---|---|---|---|
| `workers` | int ≥ 0 | `0` | number of Worker processes; the factory uses one Worker when the value is ≤ 0 |
| `worker` | int ≥ 0 **or** mapping | — | scalar = compatibility alias for `workers` (do not specify both); mapping = per-Worker knobs `{force_serial_layers: bool, sample_artifacts: auto\|always\|never}` |
| `batch_size` | int > 0 | `256` | number of samples submitted in one scheduler group |
| `sample_directory` | mapping | see §5.1 | SAMPLE bucket layout + tar packing (V1 parity) |
| `cleanup` | mapping | `{strategy: direct}` | handoff policy; see §9.3 |
| `archiver` | mapping | process + pack | Layer-2 policy; see §9.2 |
| `redis` | mapping | internal `127.0.0.1:6379/0` | optional broker override `{host, port, db}` overlaid on the internal zero-config default (multi-host / non-default-port deployments); other keys ignored |
| `factory` | mapping | `monitor_hz: 120`, watchdog defaults | control-process knobs: `monitor_hz` (or `monitor: {hz}`), `watchdog: {enabled, stale_sec, poll_interval_sec, max_sample_retries}` |
| `checkpoint_heartbeat_sec` | number ≥ 1 | `30.0` | **the only** checkpoint-cadence knob (all methods, nested included). It also bounds how much work a `--resume` may redo: worst case one heartbeat of wall-clock. Lower it for expensive samples. |

Unknown `EnvReqs.V2` keys are a hard error listing the supported set. Keys outside
`EnvReqs.V2` (V1's `Python`, `CERN_ROOT`, …) are tolerated and ignored; legacy
`EnvReqs.Runtime` default files are filtered to the supported V2 keys instead of failing.

### 5.1 `sample_directory` (also `Scan.sample_directory`)

| Key | Values | Default | Notes |
|---|---|---|---|
| `enabled` | bool | `true` | Redis bucket allocator on pull |
| `limit` | int ≥ 1 | `200` | samples per `SAMPLE/<bucket>/` |
| `width` | int ≥ 1 | `6` | zero-pad width (`000001`) |
| `pack` | bool | `true` | emit `<bucket>.tar.gz` after archive |
| `start_bucket` | int ≥ 1 | `1` | first bucket id |

Layout: `SAMPLE/000001/<uuid>/…` → (after pack) `SAMPLE/000001.tar.gz`.

```yaml
EnvReqs:
  V2:
    workers: 4
    batch_size: 128
    sample_directory: { limit: 200, pack: true }
    cleanup: { strategy: direct }
    archiver: { mode: process, pack_buckets: true }
    # optional groups (D12.4):
    # redis: { host: 127.0.0.1, port: 6380, db: 1 }
    # factory: { monitor_hz: 120, watchdog: { enabled: true, stale_sec: 30 } }
    # worker: { force_serial_layers: false, sample_artifacts: auto }
```

Top-level `Runtime` is rejected by the YAML loader. It remains an internal normalized object
used by the executor, not a supported task interface.

---

## 6. `Sampling`

`Sampling.Method` selects the sampler via `Distributor.set_method`. Implemented methods
(as-built):

| Family | Methods |
|---|---|
| Stateless | `Bridson`, `Random`, `Grid`, `CSV` |
| Contour | `AdaptiveBridson` (§6.9) |
| MCMC | `MCMC`, `AMMCMC`, `AM`, `DRAM` |
| Ensemble / PT | `EnsembleMCMC`, `Ensemble`, `DEMCMC`, `PTMCMC`, `PT`, `PTEnsemble` |
| Nested | `Dynesty`, `MultiNest` (§6.10) |

Unknown methods raise via Distributor (supported-list message). Alternatively
`Sampling.mode: check_modules` (or `check-modules`) runs the fixed-point calculator smoke
path (§6.7), bypassing `Method` for the scan loop (bootstrap still resolves Method first —
see Appendix A.12).

### 6.1 Keys Common to Bridson / Random / Grid

> **Bounds-only layout (breaking change, 2026-08-03).** Every **method knob** —
> `Radius`, `MaxAttempt`, `MaxWorker`, `Point number`, `Seed`, the CSV path, and the whole
> AdaptiveBridson knob set — now lives **exclusively under `Sampling.Bounds`**.
> The old layout (these keys directly under `Sampling`) is **rejected**, not silently
> accepted, with a diagnostic that names the destination:
>
> ```
> JV2-SCH-001  $.Sampling      'Bounds' is a required property
> JV2-SCH-001  $.Sampling      Additional properties are not allowed ('Point number', 'Seed' …)
> JV2-MTH-020  Sampling.Bounds Random requires Sampling.Bounds with 'Point number' (or point_number)
> JV2-MTH-001  Sampling.Point number  is not a V2 setting; place it under Sampling.Bounds
> ```
>
> **What did *not* move**: `Method`, `Variables`, `LogLikelihood`, `selection`,
> `FeedbackReturn`, `mode` / `data` / `points_csv` (check-modules) all stay directly
> under `Sampling`. The rule is simply: *if it tunes the method, it goes in `Bounds`.*

| Key | Where | Required | Default | Notes |
|---|---|---|---|---|
| `Variables` | `Sampling` | yes | — | empty list raises `ValueError`; schema in §6.2 |
| `Seed` (alias `seed`) | **`Sampling.Bounds`** | no | `0` | `0` = unseeded for Random/Bridson |
| `selection` | `Sampling` | no | — | sympy boolean over **physical** params; see box below |
| `LogLikelihood` | `Sampling` | no | — | alias of `Likelihood.expressions`, lower precedence (§6.8, §11) |

`Bounds` is a **closed** block per method: an unknown key inside it is an error, so a typo
cannot be silently ignored. `Grid` is the one method for which `Bounds` is optional (its
only knob is `Seed`; the point count comes from each variable's `num`).

**`selection` semantics (identical across all three samplers).** Bridson/Random/Grid each
pre-generate a fixed candidate/draw set once (`Point number` draws, the full grid, or the
Bridson blue-noise point count), and a rejected point consumes a slot from that fixed set
without incrementing the accepted count. **A restrictive `selection` yields fewer accepted
samples than the nominal budget** for all three — there is no regeneration/retry-on-reject
anywhere. `MaxAttempt` is unrelated to `selection`: it only bounds retries inside Bridson's own
blue-noise point-placement algorithm at `initialize()` time.

The control process's shared `ExpressionContext` caches a `CompiledExpression` by the exact
expression text and available variable-name set. Thus each distinct selection shape pays
`sympify` + `lambdify` once per process; candidate filtering performs only the cached numerical
call. This is an execution optimization and does not change the V1-compatible YAML or rejection
semantics.

### 6.2 `Variables[]` Entry & Distribution Types

```yaml
- name: x                  # required; empty/missing name is an error
  description: "..."       # informational only
  distribution:
    type: Flat             # required; names are case-sensitive
    parameters: {...}
```

| `type` | Parameters | Mapping u∈[0,1] → x |
|---|---|---|
| `Flat` | `min`, `max` | `min + (max-min)·u` |
| `Log` | `min`, `max` (>0) | `exp(log min + (log max − log min)·u)` |
| `Normal` | `mean`, `stddev` | `mean + stddev·Φ⁻¹(u)` |
| `Log-Normal` | `mean`, `stddev` | `exp(mean + stddev·Φ⁻¹(u))` |
| `Logit` | `location` (0), `scale` (1) | `logit(u)·scale + location` |
| `Binomial` | integer `n` ≥ 0, `p` ∈ [0,1] | `Binomial(n,p)` inverse CDF; yields an integer |
| `Poisson` | `lambda` ≥ 0 | `Poisson(lambda)` inverse CDF; yields an integer |
| `Beta` | `alpha`, `beta` > 0 | `Beta(alpha,beta)` inverse CDF |
| `Exponential` | `rate` > 0 | `-log(1-u) / rate` |
| `Gamma` | `shape`, `scale` > 0 | `scale·P⁻¹(shape,u)` |

Extra per-method parameters on the same `parameters` map: `num` (Grid, **required**, points
per dimension — §6.5), `length` (Bridson, **required** — §6.3, and see the code
inconsistency in Appendix A.11).

All required parameters and their numerical domains are checked by `Jarvis2 validate` before
Redis, Workers, or output directories are created.

### 6.3 Bridson

Poisson-disk / blue-noise sampling — good spatial coverage without a regular grid's
axis-aligned artifacts.

All knobs live under `Sampling.Bounds` (§6.1).

| Key (under `Bounds`) | Required | Default | Notes |
|---|---|---|---|
| `Radius` | **yes** | — | minimum u-space distance between accepted points |
| `MaxAttempt` | **yes** | — | candidates per active point during placement (`k`) |
| `MaxWorker` | no | `EnvReqs.V2.workers` | max in-flight proposals |
| `Seed` (alias `seed`) | no | `0` | |
| `Variables[].distribution.parameters.length` | **yes** (`KeyError`) | — | u-space box edge for that dimension; see Appendix A.11 for the code inconsistency with the resume/replay coordinate helpers |

Minimal recipe (adapted from `tests/parity_project/bridson_opera.yaml`):

```yaml
Sampling:
  Method: Bridson
  Bounds:
    Radius: 0.35
    MaxAttempt: 30
    Seed: 42
  Variables:
    - name: x
      distribution: {type: Flat, parameters: {min: 0.0, max: 1.0, length: 1.0}}
    - name: y
      distribution: {type: Flat, parameters: {min: 0.0, max: 1.0, length: 1.0}}
```

### 6.4 Random

Uniform i.i.d. sampling in u-space.

All knobs live under `Sampling.Bounds` (§6.1).

| Key (under `Bounds`) | Required | Default | Notes |
|---|---|---|---|
| `Point number` (alias `point_number`) | **yes** | — | note the space in the primary key name |
| `Seed` (alias `seed`) | no | `0` | |

Minimal recipe (adapted from `tests/parity_project/random_opera.yaml`):

```yaml
Sampling:
  Method: Random
  Bounds:
    "Point number": 6
    Seed: 11
  Variables:
    - name: x
      distribution: {type: Flat, parameters: {min: 0.0, max: 1.0}}
    - name: y
      distribution: {type: Flat, parameters: {min: 0.0, max: 1.0}}
```

### 6.5 Grid

Cartesian product — deterministic and exhaustive, not a target sample count.

| Key | Required | Default | Notes |
|---|---|---|---|
| `Variables[].distribution.parameters.num` | **yes** (`ValueError`) | — | points along that dimension; total samples = product across all variables |

Minimal recipe (adapted from `tests/parity_project/grid_opera.yaml`):

```yaml
Sampling:
  Method: Grid
  Bounds:            # optional for Grid — Seed is its only knob
    Seed: 0
  Variables:
    - name: x
      distribution: {type: Flat, parameters: {min: 0.0, max: 1.0, num: 3}}
    - name: y
      distribution: {type: Flat, parameters: {min: 0.0, max: 1.0, num: 3}}
```

### 6.6 CSV

Replay physical points from a file — no u→x mapping is performed (the derived mapper is
`type: none` and params travel as CSV columns; [§7](#7-the-ux-mapper-samplingmapper--auto-derived)).

| Key under `Sampling.Bounds` | Required | Default | Notes |
|---|---|---|---|
| `path` | **yes** | — | `&J/`, absolute, or task-YAML-relative |
| `variables` | no | all non-uuid columns | must be a list |
| `uuid_column` | no | `uuid` | column absent → fresh uuid4 per row |
| `delimiter` | no | `,` | |
| `encoding` | no | `utf-8` | |

Minimal recipe (adapted from `tests/parity_project/csv_opera.yaml`):

```yaml
Sampling:
  Method: CSV
  Bounds:
    path: "&J/data/check_modules_points.csv"
    variables: [x, y]
```

### 6.7 `check_modules` (`Jarvis2 check` / `Sampling.mode: check_modules`)

Smoke path for the calculator/opera/likelihood chain. **Two sources of points** (in order):

1. **CSV fixed-point replay** when the resolved file exists.
2. Else **sampler-drawn smoke points** (`n_samples`, default **10**) from `Sampling.Method`
   (V1-like assembly-line: Bridson/Random/Grid use `propose_next`; Nested/MCMC draw unit-cube
   `u` and run the Worker plan).

CSV path resolution:

| Priority | Source |
|---|---|
| 1 | `Sampling.data` / `Sampling.points_csv` (task YAML) |
| 2 | `EnvReqs.V2.check_modules.data` (default environment YAML) |
| 3 | Built-in default `&J/data/check_modules_points.csv` |

| Key | Required | Notes |
|---|---|---|
| `Sampling.data` (alias `points_csv`) | no | Overrides default CSV; if the file is missing → sampler fallback |
| `EnvReqs.V2.check_modules.data` | no | Default CSV path from `deps/environment_default.yaml` |
| `EnvReqs.V2.check_modules.n_samples` | no | Default `10` — used only when CSV is missing |

At start the control log prints which path was used, e.g.  
`check-modules: using fixed points from CSV -> …` or  
`check-modules: CSV not found … drawing 10 smoke point(s) from Sampling.Method=…`.

**Always-on runtime policy for `Jarvis2 check` / `--check-modules` (not YAML):**

| Setting | Value | Why |
|---|---|---|
| Workers | **1** | simple, inspectable smoke |
| Calculator PackID pool | **1** (`make_paraller` / `Pools` / module `make_paraller` pinned) | avoid free-list rotation installing a new `clone_shadow` pack per sample |
| Sample root | **`outputs/<scan>/SAMPLE/test/<uuid>/`** | flat uuid dirs (no `00000N/` buckets) |
| Numbered buckets | **off** (`sample_directory.enabled=false`) | direct inspect under `test/` |
| Bucket pack (tar.gz) | **off** | never tar during check |

Full `Jarvis2 run` is unchanged (multi-worker, `SAMPLE/00000N/<uuid>/`, optional pack).

Coordinate columns for CSV rows follow `Sampling.Variables` names when present (not hard-coded
`x,y` only). Minimal dedicated check card:

```yaml
Sampling:
  mode: check_modules
  data: "&J/data/check_modules_points.csv"
```

Or run a normal scan card with smoke points only:

```bash
Jarvis2 check bin/Example_MultiNest_Operas_Quick.yaml
# → CSV if present under EnvReqs default path; else 10 MultiNest-dim unit-cube points
```

### 6.8 Likelihood Expression Semantics

Shared wording with §11; kept here because `Sampling.LogLikelihood` is the lower-precedence
alias of `Likelihood.expressions`. Expressions are compiled once via the shared
`ExpressionContext` / `CompiledExpression` runtime (`jarvishep2/expression.py`) and evaluated
over the observables dict; earlier terms' results are visible to later terms. If a term is
literally named `LogL` it becomes the total; otherwise `LogL = Σ terms`. All 38 V1 lightweight
functions listed in the shared-language catalog are available through
`inner_func.NUMERIC_MODULES`. Missing observables raise a structured
`MissingExpressionVariablesError` (the Sample fails). See Appendix A.6 for the remaining
undeclared-symbol collision risk.

### 6.9 AdaptiveBridson

Feedback-driven **outer/core Bridson** densifier with root-correction and
endpoint extension (`jarvishep2/Sampling/adaptive_bridson.py`). Registered
`stateless=False`; Redis runtime enables Worker `publish_feedback`.

The control API names its wait setting `generation_timeout` (default `3600` seconds);
it is applied independently to each generation/barrier, not as a total run budget.
The legacy `timeout` keyword remains accepted as an alias for compatibility.

| Doc | Role |
|---|---|
| Binding algorithm | [`DESIGN_ADAPTIVE_BRIDSON_LIVE_BAND.md`](DESIGN_ADAPTIVE_BRIDSON_LIVE_BAND.md) |
| YAML recipes (minimal + full) | [`YAML-Example/ADAPTIVE_BRIDSON.md`](YAML-Example/ADAPTIVE_BRIDSON.md) |

**Control loop (summary)** — full flow in the design doc:

1. **Gen-0**: global Bridson (d≤4) / Sobol (d=5) with `initial_radius` \(r_0\).
2. **Classify** absolute bands: core \(\lvert f-T\rvert\le w_{\mathrm{core}}\),
   frontier up to \(w_{\mathrm{outer}}\). Default \(w_{\mathrm{core}}=w_{\mathrm{outer}}/8\).
3. **Gate** \(t_{\min},t_{\max}\) from finite \(f\) in a **\(2\,r_g\)** ball about
   best (\(\arg\min\lvert f-T\rvert\)) — exit / next-gen only, not densify mask.
4. **Same-\(r_g\) fill** (`fill_pass++`, generation does **not** advance):  
   root-correction on straddles → active endpoints (omni probes) → MST bridge →
   local Bridson densify in each core’s \(r_g\) ball (blue-noise sep \(=r_g\)).
5. When fill is quiet: if band thin **and** cores cover contour → **converged**;
   else \(r_g\leftarrow\max(r_{\min}, r_g\times\rho)\), `generation+=1`, rebuild
   endpoints.
6. Hard stops: `min_radius`, `max_generations` (max \(r_g\) shrinks), `max_points`.

**Constraints**

| Constraint | Rule |
|---|---|
| Dimension | `2 ≤ len(Variables) ≤ 5` else `ValueError` at `set_config` |
| Sub-block | `Sampling.Bounds` **required** mapping (§6.1) |
| Gen-0 | d ≤ 4 Bridson; d = 5 Sobol |
| Neighbor graph | Delaunay (d≤3) / kNN (d≥4) for brackets; densify is local Bridson |
| Output | `<task_result_dir>/levelset.json` + SAMPLE/DATABASE archive |

**Keys under `Sampling.Bounds`**

Public (new cards):

| Key | Required | Default | Notes |
|---|---|---|---|
| `target_expression` | **yes** | — | sympy over returned observables (+ variable names) |
| `target_value` | **yes** | — | float \(T\) |
| `outer_half_width` | no | `0.02` | discovery \(\lvert f-T\rvert\le w_{\mathrm{outer}}\) |
| `min_radius` | no | `1/200` | u-space Euclidean floor for \(r_g\) |

Derived: `core_half_width = outer_half_width/8`, `threshold = core_half_width`
(\(t_{\max}-t_{\min}\) stop).

Common optional:

| Key | Default | Notes |
|---|---|---|
| `initial_radius` | `0.10` | \(r_0\): gen-0 + first windows |
| `refinement_factor` | `0.5` | \(r_g\) shrink multiplier |
| `bridge_gaps` | `true` | MST reconnection |
| `bridge_span_factor` | `2.5` | max bridge = factor × \(r_g\) |
| `max_generations` | `16` | max \(r_g\) shrinks (scale index) |
| `max_points` | `50000` | hard sample budget |
| `max_new_per_generation` | `4000` | densify budget per fill_pass |
| `k_ref` | `30` | Bridson trials per densify center |
| `neighbor_graph` | `auto` | `auto` \| `delaunay` \| `knn` |
| `knn_k` | `4 * d` | when graph is kNN |

Legacy / advanced (still readable): `core_half_width`, `threshold`,
`function_tolerance` (alias of threshold), `final_half_width`,
`radius_shrink_mode`, `outer_shrink_factor`, `core_spacing_factor`,
`min_cores_for_coverage`, `quiet_fill_passes`, `max_fill_passes`,
`endpoint_stall_passes`, `endpoint_omni_probes`, `slice_pairs`.

Minimal recipe (opera-only circle):

```yaml
EnvReqs:
  V2:
    workers: 2
    batch_size: 8
Sampling:
  Method: AdaptiveBridson
  Bounds:
    Seed: 7
    target_expression: "r2"
    target_value: 0.04
    outer_half_width: 0.05
    min_radius: 0.005
  Variables:
    - name: x
      distribution: { type: Flat, parameters: { min: 0.0, max: 1.0 } }
    - name: y
      distribution: { type: Flat, parameters: { min: 0.0, max: 1.0 } }
Operas:
  Modules:
    - name: Circle
      operator: jarvishep2.testing.eggbox.circle_r2
      call_mode: call
      input:
        - { name: x, expression: x }
        - { name: y, expression: y }
```

### 6.10 Nested sampling: `Dynesty` / `MultiNest`

Both methods use the **vendored dynesty 3.x** stack + Redis evaluation pool (UUID channel).
They are **not** Fortran MultiNest / pymultinest — V1 already used that naming for static
`NestedSampler`.

| Method | Engine (fixed — no YAML switch) |
|---|---|
| `Dynesty` | always `DynamicNestedSampler` |
| `MultiNest` | always static `NestedSampler` |

**Do not set `Bounds.dynamic` / `Bounds.Dynamic`** — validation rejects those keys
(`JV2-BND-012`). Choose the Method instead of a flag.

YAML surface is aligned with the [official dynesty API](https://dynesty.readthedocs.io/en/stable/api.html):

```yaml
Sampling:
  Method: Dynesty          # or MultiNest (static NestedSampler)
  selection: "x > 0"       # optional; rejects before physics when false
  Variables: [...]
  Bounds:
    nlive: 500             # required in practice; min 2
    rseed: 21              # RNG seed (also Sampling.Seed)
    dlogz: 0.1             # evidence threshold alias:
                           #   MultiNest → run_nested.dlogz
                           #   Dynesty   → run_nested.dlogz_init

    # --- NestedSampler / DynamicNestedSampler constructor (official keys) ---
    # Prefer nested block; flat Bounds.bound / Bounds.sample / … also accepted.
    sampler:
      bound: multi           # none | single | multi | balls | cubes
      sample: auto           # auto | unif | rwalk | slice | rslice
      walks: 25              # rwalk
      facc: 0.5
      slices: null           # slice / rslice
      bootstrap: null
      enlarge: null
      update_interval: null  # int or float×nlive
      first_update: { min_ncall: 1000, min_eff: 10.0 }
      periodic: null         # list of dim indices
      reflective: null
      ncdim: null
      blob: false
      queue_size: null       # default: EnvReqs.V2.batch_size
      use_pool: null
      # history_filename / save_evaluation_history also accepted

    # --- run_nested (official keys; filtered by Method engine) ---
    run_nested:
      print_progress: true   # default true → Jarvis logger
      maxcall: 40000
      maxiter: null
      # MultiNest (static) only:
      # dlogz: 0.1
      # logl_max: .inf
      # add_live: true
      # Dynesty (dynamic) only:
      # dlogz_init: 0.01
      # nlive_init / nlive_batch / maxbatch / n_effective / use_stop / …
```

**Checkpoint/resume knobs are HEP-owned, not yours.** `checkpoint_file`,
`checkpoint_every` and `resume` are **rejected** inside `Bounds` (`JV2-BND-001`) — the
engine's native save/restore is driven by HEP:

- cadence comes from `EnvReqs.V2.checkpoint_heartbeat_sec` (§5) — one knob for every method;
- the checkpoint path is derived from the scan, and `--resume` picks it up automatically;
- the Redis evaluation pool is stripped when the engine state is pickled and re-attached
  on restore, so a nested run resumes with its live points intact.

**Pass-through rules**

1. Constructor kwargs: union of flat official keys + `Bounds.sampler` / `Bounds.constructor`.
   HEP always injects `loglikelihood`, `prior_transform`, `ndim`, `pool`, `rstate` (stripped
   if a user pastes them). Unknown keys are **ignored with a warning**, never crash.
2. `run_nested` kwargs: filtered to the live `run_nested` signature. MultiNest uses
   `dlogz`; Dynesty uses `dlogz_init` (a top-level `Bounds.dlogz` is remapped to
   `dlogz_init` for Dynesty).
3. Outputs: `DATABASE/dynesty_result.csv` or `DATABASE/multinest_result.csv` + stock
   Jarvis-PLOT `dynesty_runplot` jplot under `images/<scan>/`.

**Minimal everyday card (recommended):**

```yaml
Sampling:
  Method: Dynesty          # or MultiNest (static NestedSampler)
  Variables: [...]
  Bounds:
    Seed: 21
    nlive: 100
    dlogz: 0.5             # Dynesty → dlogz_init; MultiNest → static dlogz
    run_nested:
      print_progress: true
  LogLikelihood: [...]
```

Scaffold templates: `project_template/bin/sampling/Sampling_{Dynesty,MultiNest}_{Simple,Full}.yaml`.

### 6.11 MCMC family: `MCMC` / `AMMCMC` / `AM` / `DRAM`

Feedback-driven Metropolis on Redis. One generation ≈ one proposal stage across chains;
Workers evaluate `LogL`; control absorbs accept/reject.

| Key (`Sampling.Bounds`) | Aliases | Default | Notes |
|---|---|---|---|
| `num_chains` | `chains` | `1` | Prefer `chains ≥ workers` |
| `num_iters` | `steps` | required-ish | Iterations per chain |
| `proposal_scale` | — | `0.1` | Gaussian scale (unit cube) |
| `on_failure` | — | `reject` | `reject` \| `halt` |
| `adapt.enabled` | `adapt_enabled` | `true` (AM/DRAM) | Adaptive covariance |
| `adapt.start_iter` / `adapt.window` / `adapt.eps` / `adapt.scale` | dotted or flat | V1 defaults | AM/DRAM |
| `dr.steps` | `dr_steps` | `2` | DRAM delayed-rejection stages |
| `dr.scale_factors` | `dr_scale_factors` | `[1.0, 0.5]` | DRAM |

**Outputs:** `DATABASE/samples.hdf5` rows include `chain_id`, `step`, `stage`;
`DATABASE/chain_history.csv` has `accepted`/`weight`; `DATABASE/sampler_summary.json`
has accept rates + Gelman–Rubin `rhat_logl`. See [datarecorder.md](components/datarecorder.md) §2.1.

### 6.12 Ensemble / DE / PT: `EnsembleMCMC` / `Ensemble` / `DEMCMC` / `PT*` 

Same `MCMCSampler` class, different engines:

| Method | Engine notes |
|---|---|
| `EnsembleMCMC` / `Ensemble` | Stretch move; half-ensemble barriers; `stretch_a` (default 2.0) |
| `DEMCMC` | Differential evolution; `de.gamma` / `de.noise` / `de.crossover` |
| `PTMCMC` / `PT` / `PTEnsemble` | Parallel tempering; temperature ladder + control-side swaps |

Common Bounds keys from §6.11 still apply (`num_chains`, `num_iters`, …). PT extras:
temperature ladder from Bounds (see engine config contract). Summary includes
`swap_attempts` / `swap_accepts` when PT is active.

### 6.12b `Sampling.FeedbackReturn` (optional, D13.8)

Controls the **flat** `hep:feedback` payload Workers publish for barrier samplers.
Archive / DataRecorder still receives the full nested sample (including `status`).

Default for Dynesty / MultiNest / MCMC family:

```json
{ "uuid": "…", "logL": -12.34 }
```

Unusable points: likelihood writes `LogL = -inf`; wire carries `"logL": "-inf"`
(decoded to IEEE −∞). **No nested `observables`, no `status` on feedback.**

| Key | Default | Notes |
|-----|---------|--------|
| `mode` | `minimal` | `minimal` \| `fields` \| `all_flat` |
| `fields` | `[]` | top-level sibling keys when `mode: fields` |
| `include_logl` | `true` | always emit `logL` (or −∞) |

```yaml
Sampling:
  Method: Dynesty
  # optional — already minimal
  FeedbackReturn:
    mode: minimal

  Method: AdaptiveBridson
  Bounds:
    target_expression: "delta_chi2"
  # default auto: mode=fields, fields from target symbols
  FeedbackReturn:
    mode: fields
    fields: [delta_chi2]
```

Alias: `feedback_return`. See [`DESIGN_FEEDBACK_RETURN_2.0.md`](DESIGN_FEEDBACK_RETURN_2.0.md).

### 6.13 `nuisance_optimize` (Worker step, not a Sampling.Method)

Declared under `Sampling.Nuisance` (or top-level `Nuisance`); inserted as an
execution-plan step **before** likelihood. Not selected via `Sampling.Method`.
Implementation: `jarvishep2/Module/nuisance.py` + `profile1d.py` (Profile1D
golden-section).

| Key | Required | Default | Notes |
|---|---|---|---|
| `Method` | no | `Profile1D` | Only Profile1D ships in D13 |
| `Variables` | yes | — | First entry used (1-D profile); `name` + `distribution.parameters.{min,max}` |
| `LogLikelihood` | yes | — | `[{name, expression}, …]` compile-once via `ExpressionContext` |
| `PassCondition` | no | `[]` | Soft-fail sample when any term is false at best probe |
| `TargetMode` | no | `min` | `min` / `max` |
| `MaxAttempt` | no | `50` | Golden-section probe budget |
| `re_run_physics` | no | **`true`** | Alias: `rerun_physics` |

**`re_run_physics` (D13.7c decision):** default **`true`** for V1 parity. V1
Profile1D re-executed each probe as a full sample (`NAttempt` card → new
Worker pipeline). V2 folds that into one Worker step; when true, every probe
re-runs calculators + Operas with the free nuisance injected. Set
`re_run_physics: false` when the nuisance is pure expression-only (e.g.
Gaussian constraint that never feeds an external calculator) — then probes
only re-evaluate compiled LogL/pass expressions on cached observables.

```yaml
Sampling:
  Nuisance:
    Method: Profile1D
    Variables:
      - name: ratio
        distribution: {type: Flat, parameters: {min: -3, max: -2}}
    LogLikelihood:
      - {name: L_nu, expression: "Abs(log(WN2) + 40.3)"}
    TargetMode: min
    MaxAttempt: 100
    re_run_physics: true   # default; false = expression-only probes
    PassCondition: []
```

---

## 7. The u→x mapper (`Sampling.Mapper` + auto-derived)

The mapper turns a sampler's normalized draw (`u_coords ∈ [0,1]^d`) into physical parameters
that seed `Sample.params` / observables and feed `Sampling.selection`.

### 7.1 Default (no `Sampling.Mapper`)

When `Sampling.Mapper` is omitted, behavior is unchanged: a distribution-only pipeline is
auto-built from `Sampling.Variables` (D22 / `MapperPipeline`).

| Card shape | Behavior |
|---|---|
| `Sampling.Method: CSV` | no u→x mapping; params travel in `opera_params` (CSV columns). **`Sampling.Mapper` is rejected** (`JV2-MAP-010`) |
| `Sampling.Variables` present | V1-compatible distribution mapping (§6.2), one parameter per variable, declaration order |
| neither | identity fallback `keys: [x, y]` — legacy Eggbox / test only |

### 7.2 Optional `Sampling.Mapper` — flat name → expression

`Sampling.Mapper` is an **ordered list** of fixed-key mappings
`{name, expression}`. Schema keys stay fixed so user symbols never become
object property names (JSON Schema–friendly):

```yaml
Sampling:
  Method: Bridson
  Variables:
    - name: t
      distribution:
        type: Flat
        parameters: {min: 0.0, max: 6.283185307, length: 1.0}
  Mapper:
    - name: x
      expression: "cos(t)"
    - name: y
      expression: "sin(t)"
  selection: "y > 0"
```

The free-form object form `Mapper: {x: "cos(t)"}` is **rejected** (`JV2-MAP-001` / schema).

Rules (enforced at `Jarvis2 validate` / load — prefix `JV2-MAP`):

- **Stacking, not replacement.** `Variables[].distribution` still defines the prior /
  sampling transform; Mapper expressions run **after** that (reparameterization only).
- **Closed namespace.** Free symbols ⊆ `Variables` names ∪ other Mapper names ∪ constants
  (`Pi`/`E`/`Inf` …). No observables, no `LogL`/`uuid`, no Operas functions, no RNG/I/O.
- **DAG.** Expressions may reference earlier-defined Mapper names; load-time topological
  sort; cycles → `JV2-MAP-004`.
- **Names.** Mapper keys must not collide with `Variables[].name`, reserved constants, or
  DATABASE columns (`uuid`, `sample_index`, `status`, `LogL`).
- **Output.** Sampling variables **and** Mapper outputs all appear in `Sample.params`
  (variables are never dropped).
- **Arity of `u`.** `len(u_coords) == len(Variables)` (strict; no silent truncation).
- **Plot axes.** Auto axis preference: `levelset.variable_names` → **`Mapper` write order**
  → `Variables` order → archive columns.
- **Resume.** Checkpoint card fingerprint includes a Mapper text hash; changing Mapper
  (or Variables used by it) refuses `--resume` (`mapper_hash` / D22.5).

Design: [`DESIGN_SAMPLING_MAPPER_2.0.md`](DESIGN_SAMPLING_MAPPER_2.0.md).

### 7.3 Implementation notes

- **Single pipeline.** Control process and Workers share `MapperPipeline` /
  picklable `MapperSpec` (no dual u→x implementations on the hot path).
- **Top-level `Mapper:` is still rejected** (`JV2-SCH-001 $`) — only
  `Sampling.Mapper` is legal.
- Historical note: older docs described a top-level `Mapper` with `type`/`keys`/`variables`.
  That surface never shipped; do not revive it.

---

## 8. `LibDeps`

### 8.1 `LibDeps.Modules` — shared install-once libraries

Defines shared native-library builds and `${LibDeps:<name>}` tokens. The control process
handles this block once, **after environment preflight and before Redis/Workers start**.
A failed build therefore stops the run before any scan process is spawned. Modules are
ordered by `required_modules`; independent modules may build concurrently up to
`LibDeps.make_paraller`.

Path resolution per entry, **first hit wins**:

```
1. Modules[].installation.path
2. Modules[].installation.source
3. Modules[].path
4. Modules[].source
5. <LibDeps.path>/<name>                (fallback)
```

`LibDeps.path` itself defaults to the project root and supports `&J/`.

The token vocabulary is `${LibDeps:<name>}`, `${LibDeps:path}` (the base directory),
`${LibDeps:make_paraller}`, `${path}` / `${source}` within a module's own installation,
and `@{ROOT path}`. The latter comes from `EnvReqs.CERN_ROOT.path` or
`get_path_command`; using it without that configuration fails before the build with a
message naming `EnvReqs.CERN_ROOT`.

`installation.commands` accepts V1's plain string list. Commands preserve V1's sequential
standalone `cd DIR` behaviour: the following command starts in that directory. Mapping
commands (`{cmd: ..., cwd: ...}`) are also accepted.

Each successful module writes `<installation.path>/.jarvis_install_stamp.json`; the shared
control file is `<LibDeps.path>/jarvis_install.json`. A matching fingerprint (module name,
installation path/source/commands, and source-root stat) reuses the existing build. The
deliberate limitation is the same as calculator installs: editing a file **inside** a source
directory does not request a rebuild. Set `"reinstall": true` in the control file before
the next run to rebuild all modules; V2 never asks an interactive `Reinstall? (y/n)` prompt.
V1's `installed:` key is accepted but ignored — the stamp is authoritative.

`Jarvis2 run TASK.yaml --skip-library-installation` (and `check`) does no build, emits a
warning, and verifies that every declared installation path already exists. Per-module
build output goes to `logs/<scan>/library-<module>.log`.

### 8.2 `LibDeps.registered_executables`

Register a tool once; its bare name then resolves to an absolute path inside every command
string (word-boundary replacement, longest name first — so a registered name that is a
substring of another registered name is still resolved correctly).

| Key | Required | Values | Notes |
|---|---|---|---|
| `name` | yes | — | empty → `ValueError` |
| `source` | yes | path (all static tokens OK) | must exist → `FileNotFoundError` |
| `resolution` | no | `direct_path` (default) \| `symlink` | symlink materialized under `<root>/deps/registered/<name>` |

---

## 9. `Calculators`

Owns the external-program execution model: concurrency pools, the archiving pipeline, and the
per-module install/init/execute lifecycle. See §9.5 for the `execution` sub-block detail and
§12 for the token language used inside `path`/`cmd`/`cwd` fields.

### 9.1 `Calculators.Pools` (alias `pools`)

`{name: slots}` — global concurrency caps, seeded into Redis `calc:free:<name>`. Values are
coerced with floor 1. When absent, per-module `make_paraller` (default 1) is used
(`calculator_pools.resolve_calculator_pools`). A Worker blocks up to `calc_acquire_timeout`
(30 s, internal — Appendix B) waiting for a slot, then the Sample fails with `TimeoutError`.
For a multi-mode parent, the pool key is the parent name (`Prospino`, not
`Prospino.ng`). Prefer at least `mode count + Worker count` slots when resources
allow; fewer slots are legal but can increase mode rebuilds. A pool smaller
than the mode count cannot keep every mode resident and must thrash.

### 9.2 `Calculators.Archiver`

Controls Layer-2 persistence (`DATABASE/` writes + optional SAMPLE bucket tar).

| Key | Values | Default | Notes |
|---|---|---|---|
| `mode` | `thread` \| `process` | **`process`** | dedicated Archiver process (default) vs control-process thread |
| `handoff` | `direct` \| `staging` | **`direct`** | mirrors Cleanup; staging hop only when set |
| `pack_buckets` | bool | **`true`** | pack sealed SAMPLE buckets after `archived>=assigned` |
| `batch_size` | int > 0 | `200` | Samples buffered before a DATABASE batch flush |
| `flush_interval_sec` | float ≥ 0.05 | `1.0` | time-based flush trigger for partial batches |
| `strategy` | `move` \| `copy` | `move` | transfer method when a staging hop is used |
| `delete_after_archive` | bool | `true` | delete staging residue when staging hop is used |

### 9.3 `Calculators.Cleanup`

Controls optional staging handoff. **Default is direct** (products already under
`SAMPLE/<bucket>/<uuid>/`).

| Key | Values | Default | Notes |
|---|---|---|---|
| `strategy` | `direct` \| `mv_to_staging` | **`direct`** | `mv_to_staging` enables the optional buffer hop |
| `staging_dir` | path | `<task_result_dir>/staging` | only used when strategy is `mv_to_staging` |

### 9.4 `Calculators.Modules[]` — one external calculator

| Key | Required | Default | Notes |
|---|---|---|---|
| `name` | **yes** | — | nameless entries silently dropped |
| `required_modules` | no | `[]` | names → layer assignment; same layer runs concurrently in one Sample; unknown deps do not error (cycle fallback lumps remainder) |
| `clone_shadow` | no | `false` | `true` = physical per-pack copy for non-thread-safe tools |
| `path` | no | `execution.path` → `"."` | runtime dir; `@PackID` allowed with clone_shadow |
| `source` | no | — | clone_shadow: `copytree` source when no `installation`; non-shadow: symlinked into the Sample dir |
| `symlink_name` | no | module name | link name for the non-shadow symlink |
| `timeout` | no | none | seconds; deadline across **all** execution commands |
| `make_paraller` | no | 1 | pool-slot fallback (V1 spelling kept — Appendix A.4) |
| `env_setup` | no | `[]` | `[{source: script.sh}]`; `source <script> && env` captured **once per Worker**, cached, merged over `os.environ` |
| `installation` | no | `[]` | `[{cmd, cwd}]`, clone_shadow only, once per pack; `${source}`/`${path}` tokens |
| `initialization` | no | `[]` | same shape, runs after install |
| `execution` | yes in practice | `{}` | see §9.5 |

### 9.5 `execution` block

| Key | Notes |
|---|---|
| `path` | fallback for the module `path` |
| `commands` | `[{cmd, cwd}]`; `cwd` default `"."`; run sequentially via the per-Worker scheduler; non-zero rc or timeout → Sample fails |
| `input` | file specs written **before** commands |
| `output` | file specs read **after** commands |

**`input[]` spec**: `{name, path, type, actions}`. **`type` is resolved via Jarvis-Portal**
(JSON/CSV/TSV/DAT/Wolfram and any format the installed Portal exposes). Unknown types
**hard-fail** with the registered list (no silent skip). Only `actions[].type: Dump` is implemented;
each `variables` entry writes `name:` either from `expression:` (sympy over the
params/observables, `Pi`/`pi` available, non-numeric params → `ValueError`) or the raw param
value (missing → `"MISSING_VALUE"` string). If the file already exists it is merged, not
replaced.

**`output[]` spec**: `{name, path, type, variables}`. Same Portal registry as input; each `variables`
entry maps `name:` from `entry:` (dotted path, default = name; missing → `None`).

A `save:` key appears in V1-style specs and the parity YAML; **V2 never reads it**
(Appendix A.3).

### 9.6 Multi-mode calculator (shared PackID)

Use `modes` only when a package's build states are mutually exclusive: switching
to mode B replaces mode A in the same physical installation. If both programs
or build directories can coexist, declare two ordinary modules instead.

```yaml
Calculators:
  Pools: {Prospino: 8}
  Modules:
    - name: Prospino
      path: "&J/calculators/Prospino/@PackID"
      source: "&J/src/Prospino"
      clone_shadow: true
      installation: ["cp -R ${source}/. ${path}"]
      modes:
        - name: ng
          installation: ["python configure.py ng", "make"]
          initialization: ["rm -f ng.out"]
          execution:
            commands: ["./prospino"]
            output: [{name: xsec_ng, path: "@Sdir/ng.json", type: JSON}]
        - name: ns
          installation: ["python configure.py ns", "make"]
          execution:
            commands: ["./prospino"]
            output: [{name: xsec_ns, path: "@Sdir/ns.json", type: JSON}]
```

The parent owns one physical `@PackID` pool and one `jarvis_install.json`.
Parent `installation` runs only for a new/invalidated base. A mode switch runs
only that mode's optional `installation`; it does not replay parent installation.
Parent initialization and mode initialization are optional and run on every
selected call, in that order. `selection: false` is evaluated before Redis
acquisition, so a skipped mode neither rebuilds nor relabels a pack.

A mode may set `path` and `execution.path` for its command working directory:

```text
mode.execution.path > mode.path > parent.path
```

The parent path remains the physical installation anchor. There is no `@Mode`,
`${mode_dir}`, `mode_packs`, or per-mode physical directory.

Logical names use a dot: `Prospino.ng`. A bare dependency `Prospino` means all
sibling modes; a dotted dependency selects one or more explicit modes. Bare
dependencies may also refer to LibDeps, Operas, or `Parameters`. Modes of one
parent run serially within a sample; different calculator parents can run in
parallel. Redis prefers a warm target pack, waits briefly when that exact mode
is in flight, then borrows and rebuilds only when necessary.

---

## 10. `Operas`

Every dynamically discovered Jarvis-Operas **function** is callable in shared expressions by
its qualified registered name, for example `math.add(x, y)`. **Namespace constants** (D23)
use the same dotted syntax **without parentheses**, for example `pdg.mZ` or
`(mz - pdg.mZ) / pdg.mZ`. Constants are folded at parse time into literal floats (they do
not appear in free symbols or the lambdify parameter list). A name cannot be both a bare
constant and a callable form: `pdg.mZ()` is an error.

Persisted user functions and installed `jarvis_operas.core` / `jarvis_operas.user` entry
points are discovered independently in each spawn Worker. There is deliberately no V2-only
function-registration or alias YAML block.

`Operas.Modules[]` — in-process Python operators (no subprocess, no staging — runs directly
in the Worker, once imported at startup).

| Key | Required | Default | Notes |
|---|---|---|---|
| `make_paraller` | no | ignored | retained V1 spelling/shape; V2 Worker and workflow-layer concurrency own scheduling |
| `name` | effectively yes | `Operas<i>` | dict key for the execution plan |
| `operator` | **yes** (`KeyError`) | — | importlib dotted callable first, then optional Jarvis-Operas registry name; resolved once per Worker. **Must not** be a namespace constant (`pdg.mZ` etc.) — that is rejected with `JV2-OPR-002`; write the constant in an expression instead |
| `call_mode` | no | `call` | intended `call` \| `acall`; **unknown values currently fall through to sync call** (A.17) |
| `timeout_sec` (alias `timeout`) | no | none | thread-based timeout; the runaway thread is **not** killed |
| `kwargs` | no | `{}` | static kwargs; `observables` is always injected, and every input observable is also passed as a kwarg |
| `input` | no | pass-through | `{name, expression}` (shared expression language; compiled once per Worker and reused per Sample) or `{name, entry}` (dotted copy). The bare-string copy form `"x"` is **implemented but currently unusable** — see the box below |
| `output` | effectively yes | `[]` | `{name, entry}`; **an empty list discards the entire result** |

> **Schema is stricter than the runtime (2026-08-04, D22.8).** `operas.py:319-321` explicitly
> handles a bare string in `input` (`payload[item] = observables.get(item)` — copy that
> observable through), but the schema requires every `input[]` entry to be an object:
>
> ```
> JV2-SCH-001  $.Operas.Modules[0].input[0]   'x' is not of type 'object'
> ```
>
> So `input: ["x"]` is rejected at `Jarvis2 validate` even though the Worker would execute it.
> Until that is fixed, write `- {name: x, entry: x}`.

The operator must return a `Mapping`, else `TypeError` → Sample fails.

At Worker startup, every Operas input expression is `sympify`/`lambdify` compiled once and the
operator is resolved once. Each Sample evaluates those cached callables and invokes the cached
operator. Likelihood expressions use the same compile-once-per-Worker lifecycle. Calculator
Dump-variable expressions use the same class with a Worker-process cache, compiling on first
use; AdaptiveBridson targets and sampler selections use control-process contexts.

**Shared expression language:**

| Group | Names |
|---|---|
| Constants | `Pi`, `pi`, `PI`, `E`, `Inf` |
| Log/exponential | `log`, `exp`, `ln` |
| Trigonometric | `sin`, `cos`, `tan`, `sec`, `csc`, `cot`, `sinc` |
| Inverse trigonometric | `asin`, `acos`, `atan`, `asec`, `acsc`, `acot`, `atan2` |
| Hyperbolic | `sinh`, `cosh`, `tanh`, `sech`, `csch`, `coth` |
| Inverse hyperbolic | `asinh`, `acosh`, `atanh`, `acoth`, `asech`, `acsch` |
| General math | `sqrt`, `Min`, `Max`, `root`, `Abs`, `Heaviside` |
| Probability | `Gauss`, `Normal`, `LogGauss` |

`Gauss` is the V1 unnormalised Gaussian kernel; `Normal` includes `1/(σ√(2π))`;
`LogGauss` omits the normalisation term; `Heaviside(0) = 0.5`. The mechanism is
`ExpressionContext → CompiledExpression` for Operas, Likelihood, Calculator/Portal, Selection,
and AdaptiveBridson; the YAML structures remain unchanged. See
[`V1_LIGHTWEIGHT_FUNCTION_MIGRATION_2026-07-13.md`](V1_LIGHTWEIGHT_FUNCTION_MIGRATION_2026-07-13.md)
for the source audit and
[`OPERAS_DYNAMIC_FUNCTION_DISCOVERY_2026-07-13.md`](OPERAS_DYNAMIC_FUNCTION_DISCOVERY_2026-07-13.md)
for the external extension lifecycle.

`call_mode: acall` recipe (the operator is an `async def`, awaited on a private event loop —
useful when the operator itself does async I/O):

```yaml
Operas:
  Modules:
    - name: AsyncModel
      operator: mypackage.models.evaluate_async
      call_mode: acall
      timeout_sec: 30
      output:
        - name: z
          entry: z
```

---

## 11. `Likelihood` → `Sampling.LogLikelihood`

> **Correction (2026-08-04).** A **top-level `Likelihood:` block is rejected**
> (`JV2-SCH-001 $`, suggestion: *"Move its expressions to `Sampling.LogLikelihood`; top-level
> `Likelihood` is not a V2 interface."*). Authorable likelihood terms live under `Sampling`.
> The `Likelihood.expressions` → `Sampling.LogLikelihood` precedence described below is an
> **internal** config precedence (both shapes exist in the normalized in-memory config); it is
> not a choice a card can make.

```yaml
Sampling:
  LogLikelihood:
    - name: LogL_Z
      expression: LogGauss(z, 1.0, 0.1)
```

| Key | Required | Default | Notes |
|---|---|---|---|
| `Sampling.LogLikelihood[].name` | no | `"LogL"` | term name; a term named exactly `LogL` becomes the total (§6.8) |
| `Sampling.LogLikelihood[].expression` | **yes** | — | sympy expression string, evaluated over observables |

Precedence: `Likelihood.expressions` → `Sampling.LogLikelihood` (first non-empty wins; they
are never merged). Semantics in §6.8. When neither exists the plan carries no likelihood step
and `LogL` is absent from the output.

---

## 12. Token Reference

| Token | Phase | Meaning |
|---|---|---|
| `&J/` (or bare `&J`) | 1 | project root |
| `${Scan:name}` | 1 | scan name |
| `${Scan:root}` / `output` / `outputs` (any other suffix too) | 1 | `<root>/outputs/<scan>` |
| `${LibDeps:<name>}` | 1 | LibDeps module path (§8.1); unknown → `KeyError` |
| *registered executable name* | 1 | absolute tool path (§8.2) |
| `~` | 1 | `os.path.expanduser` |
| `${source}`, `${path}` | install/init only | calculator `source` / runtime path (§9.4) |
| `@SampleID` | 2 (Worker) | Sample uuid |
| `@Sdir` | 2 (Worker) | Sample save dir (forces materialization) |
| `@PackID` | 2 (Worker) | calculator slot id (traceability / clone_shadow) |

`@Mode` and `${mode_dir}` are deliberately unsupported. Multi-mode calculators
share the parent physical `@PackID` directory; mode is a logical execution name,
not a path component.

Phase-1 leftovers (`&…`, `${LibDeps:…}`, `${Scan:…}`) surviving into Phase 2 raise
`ValueError`. Sample tokens in a string are preserved through Phase 1 and the static parts
around them are still resolved — e.g. `"${LibDeps:SPheno}/@SampleID.slha"` resolves the
`LibDeps` half in the control process and leaves `@SampleID` for the Worker.

---

## 13. Validation & Failure Behavior

Two very different regimes coexist:

- **Silently coerced to defaults** (typos vanish): `EnvReqs.V2.workers`,
  `EnvReqs.V2.batch_size`, `Archiver.mode/strategy/batch_size/flush_interval_sec`,
  `Cleanup.strategy`, and `FileOperation.delete_method`,
  malformed `registered_executables`/`LibDeps`/module list entries (non-mapping or nameless
  items are dropped).
- **Hard errors, mostly raw exceptions**: missing YAML file / non-mapping top level,
  `Sampling.Variables` empty, `Radius`/`MaxAttempt`/`Point number`/`num`/`CSV.path` missing,
  unknown `Sampling.Method`, invalid `selection`, `registered_executables` name/source
  problems, unknown `${LibDeps:…}`, opera `operator` missing/not callable, top-level
  `Runtime`, and unsupported `EnvReqs.V2` keys.

**D13.9 config gate (as-built):** after `load_task_yaml`, every scan/check path runs
`jarvishep2.task_validation.validate_task_config` **before Redis/Workers**. Failures are
coded (`JV2-*`) with YAML paths and abort with exit 2. Closed surfaces include
`Sampling.Method` / `Variables` (distribution types case-sensitive), Nested
`Sampling.Bounds` allow-lists (Dynesty/MultiNest), `EnvReqs.V2` + Archiver/Cleanup
unknown keys and illegal present values, Bridson `length`/`Radius`/`MaxAttempt`,
Random `Point number`, CSV `path`. Dead keys (`execution.*.save`, …) are **warnings**
(promoted by `--strict`). Offline: `Jarvis2 validate TASK.yaml [--strict] [--json]`.

Design: [`DESIGN_YAML_VALIDATION_2.0.md`](DESIGN_YAML_VALIDATION_2.0.md).
Loader notes: `components/config_schema.md`.

Remaining soft coercions (if any still exist inside `normalize_*` for **omitted**
keys only) apply defaults; **present-but-illegal** operational knobs hard-fail at the gate.
Full Agent JSON envelope remains D8 (parked); D13.9 ships human + minimal `--json` issues list.

---

## Appendix A — Known Gaps & Design Warts (input for the YAML design review)

1. **Internal Redis is local-only.** V2 uses `127.0.0.1:6379` and now fails early if that
   service is unavailable. Remote/cluster Redis is not yet a supported V2 user interface.
2. **`Runtime.Subprocess` is an internal dead key** — normalized (`runtime_config.py:74`) and
   never read by anything downstream. Either wire it to `SubprocessRuntimeConfig` or delete it.
3. **`execution.input[].save` / `output[].save` are dead keys** — present in the parity YAML
   and V1 heritage, never read by `io_json`/`calculator`.
4. **`make_paraller` typo** (should be `parallel`) survives from V1 in module configs and the
   internal `calculator_make_paraller`. V2 is a new CLI with a new package name — the cheap
   moment to fix the spelling (optionally keeping the old key as an alias) is now.
5. **Naming-convention chaos on one surface:** `Point number` (space) vs `MaxAttempt`
   (CamelCase) vs `batch_size` (snake) vs `Seed`/`seed` both accepted; `Pools`/`pools`;
   `timeout` vs `timeout_sec`; `Sampling.LogLikelihood` vs `Likelihood.expressions`;
   `data` vs `points_csv`. Each alias pair is one more thing the docs must explain.
6. **Incomplete symbol contracts can still collide with SymPy built-ins.** All consumers now use
   the shared `ExpressionContext`, and the Operas/Likelihood function-language inconsistency is
   closed. Explicitly supplied symbol names are protected, but an observable that is not in a
   consumer's declared contract can still collide with `I`, `N`, `beta`, `gamma`, `lambda`,
   etc. The next improvement is boot-time compilation against the complete workflow observable
   schema. `Pi/pi/PI`, `E`, and `Inf` are intentionally reserved V1-compatible constants and
   take precedence over identically named observables.
7. **check_modules is x/y-only.** `core._build_check_module_samples` hard-codes
   `row["x"], row["y"]`, so the smoke path cannot exercise a real model's parameter names.
8. **Design-doc keys that do not exist in code** (do not use them): `Archiver.async_io`,
   `Archiver.format`, `Archiver.nas_optimize`, `Cleanup.staging_dir` default sketch
   `&J/outputs/${Scan:name}/staging` (actual default is `<task_result_dir>/staging`),
   `env_setup` at the *Calculators* block level (it is per-module only). Calculator
   `input`/`output` `type` is delegated to Jarvis-Portal (unknown types hard-fail; SLHA
   appears when Portal exposes it — see `DESIGN_PORTAL_IO_2.0.md`).
9. **Two different, easy-to-conflate errors for "bad `Sampling.Method`"** — which one you hit
   depends on *what kind* of mistake you made, and the more common mistake gets the better
   message:
   - A **wrong/typo'd** value (e.g. `Method: Dynestyy`) never reaches `core.py` at all. It fails
     earlier, inside `bootstrap_distributed_runtime() → init_sampler_from_config()`, via
     `Distributor.set_method`'s own `NotImplementedError` with the supported-method list —
     accurate, echoes the bad value back. (`Dynesty` / `MultiNest` **are** implemented.)
   - Only a **missing/empty** `Sampling.Method` (with `Sampling.mode` not `check_modules`)
     falls through `init_sampler_from_config`'s falsy-method branch (no exception there) and
     is later caught by `run()`'s own `else:` branch, whose message historically named only
     `Bridson` even though Random/Grid/CSV/AdaptiveBridson are equally valid choices.
   - Net effect: the *rarer* mistake (forgetting `Sampling.Method` entirely) gets the *worse*
     message; the *common* mistake (a typo) already gets a fine one.
10. **Silent-default normalization hides typos** (§13). At minimum, log a warning naming the
    rejected value; ideally reject unknown keys per block ("did you mean …").
11. **Bridson `length` is required in one code path, defaulted in another.** `bridson.py`'s
    `initialize()` does `var.parameters["length"]` — no fallback, `KeyError` if a Bridson
    variable omits it. But `sampling_utils.py`'s `map_row_to_physical`/`row_to_u_coords`
    (used for resume/replay coordinate conversion) do `var.parameters.get("length", 1.0) or
    1.0`. In practice `initialize()` always runs first, so the `KeyError` fires before the
    lenient path is ever reached — but the two functions disagree about whether `1.0` is a
    real default, which is exactly the kind of silent-inconsistency a schema layer would catch.
    Fix: either give `initialize()` the same `.get(..., 1.0)` fallback, or make `length`
    explicitly required everywhere (drop the lenient default in `sampling_utils.py`).
12. **`Sampling.mode: check_modules` does not shield you from `Sampling.Method` errors.**
    `init_sampler_from_config()` runs unconditionally during `bootstrap_distributed_runtime()`
    and resolves `Sampling.Method` via `Distributor.set_method` *before* `run()` ever checks
    `Sampling.mode`/`--check-modules`. A check-modules task doesn't use `Sampling.Method` at
    all, but if one is present (e.g. left over from copy-pasting a real scan YAML) and set to
    anything other than Bridson/Random/Grid/CSV/AdaptiveBridson, bootstrap fails with
    `Distributor.set_method`'s `NotImplementedError` before the check-modules path is ever
    reached. Two independent-looking knobs (`mode` and `Method`) have a hidden ordering
    coupling.
13. **`--pid` is a dead CLI flag.** `client.py`'s `build_parser()` defines
    `--pid` ("Attach to a running scan by control PID") but `args.pid` is never read anywhere —
    passing it silently does nothing. Also note: `V2_DISTRIBUTED_PLAN.md`'s "Verification
    Commands" section shows `Jarvis2 <task>.yaml --benchmark 30` and `Jarvis2 <task>.yaml
    --convert`, but neither flag exists in the current `build_parser()` — those are
    planned/aspirational, not implemented.
14. **CLI modes are not mutually exclusive.** `--monitor`, `--plot`, and `--check-modules`
    can be supplied together. `dispatch()` silently follows its internal precedence instead of
    rejecting the conflicting intent. D11.2 replaces this with explicit subcommands/usage errors.
15. **Redis is no longer a CLI/YAML user surface (as of `0a5e85e`).** `--redis-host/port/db`
    and top-level `Runtime.redis` were removed; the broker is always the internal local service.
    Multi-host / non-default ports need D12.4's planned optional `EnvReqs.V2.redis` override.
    The older "default-valued flag does not override YAML redis" wart is retired with those
    flags.
16. **CLI success does not mean samples succeeded.** The run path returns submitted/archived
    counts, and failed records are archived. An all-failed run can therefore return exit 0.
    D11.1 introduces a single `RunOutcome` and explicit partial-failure policy.
17. **`Operas.Modules[].call_mode` is not validated.** Values other than `acall` currently
    fall through to the synchronous call path. Invalid values must fail during validation/preload.
    Separately, failed sample envelopes lack stable `error_type`/`error` fields (D8/D11.1).

## Appendix B — Internal Worker-Config Keys (not YAML)

`build_worker_config(extra=…)` / tests can inject these; they are **not** read from the task
YAML today. Listed so the review can decide whether any deserve promotion to YAML:

| Key | Default | Meaning |
|---|---|---|
| `pull_timeout` | 1 (builder) / 5 (Worker fallback) | Redis `blpop` timeout per loop |
| `calc_acquire_timeout` | 30 | seconds to wait for a calculator slot |
| `subprocess_max_concurrency` | auto (max layer width) | per-Worker scheduler cap |
| `force_serial_layers` | false | rollback switch: serialize same-layer calculators |
| `calculator_pools` | from `Calculators.Pools` | explicit slot map |
| `calculator_make_paraller` | — | global fallback slot count |
| `test_process_delay_sec` | 0 | test-only artificial delay |

## Appendix C — CLI Surface (for completeness)

```
Jarvis2 <task>.yaml            # run (uses internal local Redis)
Jarvis2 <task>.yaml --check-modules
Jarvis2 <task>.yaml --resume   # skip the 30 s resume prompt
Jarvis2 --monitor [--redis-host H --redis-port P --redis-db N]
Jarvis2 <plot-scene>.yaml --plot  # render-only; positional is NOT a scan task
Jarvis2 <task>.yaml --pid N    # accepted by argparse but dead — see Appendix A.13
```

Exit codes: 0 ok, 1 run/IO failure or nothing archived, 2 usage / unsupported task.

`--benchmark`/`--convert` (referenced in `V2_DISTRIBUTED_PLAN.md`'s verification commands)
are **not implemented** in the current `client.py` — see Appendix A.13. Portal/Operas
discovery subcommands are also absent; use `jportal`/`jopera` as temporary package-level tools.
