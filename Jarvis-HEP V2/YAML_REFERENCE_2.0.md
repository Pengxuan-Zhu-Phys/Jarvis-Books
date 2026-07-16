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
7. [`Mapper`](#7-mapper-top-level-optional)
8. [`LibDeps`](#8-libdeps)
9. [`Calculators`](#9-calculators)
10. [`Operas`](#10-operas)
11. [`Likelihood`](#11-likelihood)
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
project_name: my-scan            # str; only used in run_summary
Scan:
  name: scan-01                  # str; drives outputs/<name> and ${Scan:name}
# scan_name: scan-01             # fallback alias when Scan.name is absent
# task_result_dir: /abs/path     # override the outputs/<scan> root
# run_id: my-run-id              # override the auto uuid4 run id

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
  Method: Bridson                # Bridson | Random | Grid | CSV | AdaptiveLevelSet
  # mode: check_modules          # special task type; replaces Method (needs `data`)
  # data: "&J/points.csv"        # check_modules input CSV (alias: points_csv)
  Seed: 42                       # int (alias: seed); default 0
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
  # -- per-method keys --
  Radius: 0.35                   # Bridson: REQUIRED minimum point distance (u-space)
  MaxAttempt: 30                 # Bridson: REQUIRED k candidates per active point
  # MaxWorker: 4                 # Bridson: max in-flight proposals (default EnvReqs.V2.workers)
  # "Point number": 500          # Random: REQUIRED sample count (alias: point_number)
  # CSV:                         # CSV method: REQUIRED block
  #   path: "&J/points.csv"      #   REQUIRED; &J/, absolute, or task-YAML-relative
  #   variables: [x, y]          #   column subset (default: all non-uuid columns)
  #   uuid_column: uuid          #   default "uuid"; missing column -> fresh uuid4 per row
  #   delimiter: ","             #   default ","
  #   encoding: utf-8            #   default utf-8
  # AdaptiveLevelSet:            # AdaptiveLevelSet method: REQUIRED sub-block (§6.9)
  #   target_expression: "LogL"  #   REQUIRED sympy over returned observables
  #   target_value: -2.9957      #   REQUIRED level-set constant
  #   contour_precision: 0.01    #   default 0.01 (u-space edge length)
  #   function_tolerance: 0.05   #   default 0.05
  #   initial_radius: 0.08       #   default 0.08 (gen-0 spacing)
  #   max_generations: 25        #   default 25
  #   max_points: 5000           #   default 5000 (d≤3) / 20000 (d≥4)
  # LogLikelihood:               # alias of Likelihood.expressions (lower precedence)
  #   - name: LogL_Z
  #     expression: z

# ---- u -> x mapper (optional; auto-derived when absent) ---------------------
Mapper:
  type: distribution             # distribution | identity | none | flat(default fallthrough)
  # keys: [x, y]                 # identity: observable names for u components
  # variables: [...]             # distribution/flat: same schema as Sampling.Variables

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
      # samples, Workers, and runs. Each NEW Worker re-runs installation on an
      # already-populated pack directory, so installation commands MUST be
      # idempotent (plain overwrite-copy like the above is fine; V1 cards
      # already assume this).
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

# ---- likelihood ---------------------------------------------------------------
Likelihood:
  expressions:                   # precedence over Sampling.LogLikelihood
    - name: LogL_Z
      expression: LogGauss(z, 1.0, 0.1)   # shared 38-function Expression Core
    # semantics: a term literally named LogL is THE total; otherwise LogL = sum of terms
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
| `Sampling.mode` | [6.7](#67-check_modules-samplingmode-check_modules) | no | — |
| `Sampling.data` / `Sampling.points_csv` | [6.7](#67-check_modules-samplingmode-check_modules) | yes (check_modules) | — |
| `Sampling.Seed` / `Sampling.seed` | [6.1](#61-keys-common-to-bridson--random--grid) | no | `0` |
| `Sampling.selection` | [6.1](#61-keys-common-to-bridson--random--grid) | no | — |
| `Sampling.Variables[].name` | [6.2](#62-variables-entry--distribution-types) | yes | — |
| `Sampling.Variables[].description` | [6.2](#62-variables-entry--distribution-types) | no | — |
| `Sampling.Variables[].distribution.type` | [6.2](#62-variables-entry--distribution-types) | no | `Flat` |
| `Sampling.Variables[].distribution.parameters.{min,max}` | [6.2](#62-variables-entry--distribution-types) | yes (Flat/Log) | — |
| `Sampling.Variables[].distribution.parameters.{mean,stddev}` | [6.2](#62-variables-entry--distribution-types) | yes (Normal/Log-Normal) | — |
| `Sampling.Variables[].distribution.parameters.{location,scale}` | [6.2](#62-variables-entry--distribution-types) | no (Logit) | `0`, `1` |
| `Sampling.Variables[].distribution.parameters.num` | [6.5](#65-grid) | **yes (Grid)** | — |
| `Sampling.Variables[].distribution.parameters.length` | [6.3](#63-bridson) | **yes (Bridson)** | — |
| `Sampling.LogLikelihood` | [6.8](#68-likelihood-expression-semantics) | no | — |
| `Sampling.Radius` | [6.3](#63-bridson) | **yes** | — |
| `Sampling.MaxAttempt` | [6.3](#63-bridson) | **yes** | — |
| `Sampling.MaxWorker` | [6.3](#63-bridson) | no | `EnvReqs.V2.workers` |
| `Sampling."Point number"` / `point_number` | [6.4](#64-random) | **yes** | — |
| `Sampling.CSV.path` | [6.6](#66-csv) | **yes** | — |
| `Sampling.CSV.variables` | [6.6](#66-csv) | no | all non-uuid columns |
| `Sampling.CSV.uuid_column` | [6.6](#66-csv) | no | `uuid` |
| `Sampling.CSV.delimiter` | [6.6](#66-csv) | no | `,` |
| `Sampling.CSV.encoding` | [6.6](#66-csv) | no | `utf-8` |
| `Sampling.AdaptiveLevelSet` / `adaptive_level_set` | [6.9](#69-adaptivelevelset) | **yes** (ALS) | — |
| `Sampling.AdaptiveLevelSet.target_expression` | [6.9](#69-adaptivelevelset) | **yes** | — |
| `Sampling.AdaptiveLevelSet.target_value` | [6.9](#69-adaptivelevelset) | **yes** | — |
| `Sampling.AdaptiveLevelSet.contour_precision` | [6.9](#69-adaptivelevelset) | no | `0.01` |
| `Sampling.AdaptiveLevelSet.function_tolerance` | [6.9](#69-adaptivelevelset) | no | `0.05` |
| `Sampling.AdaptiveLevelSet.initial_radius` | [6.9](#69-adaptivelevelset) | no | `0.08` |
| `Sampling.AdaptiveLevelSet.refinement_factor` | [6.9](#69-adaptivelevelset) | no | `0.5` (d≤3) / `0.65` (d≥4) |
| `Sampling.AdaptiveLevelSet.max_generations` | [6.9](#69-adaptivelevelset) | no | `25` |
| `Sampling.AdaptiveLevelSet.max_points` | [6.9](#69-adaptivelevelset) | no | `5000` (d≤3) / `20000` (d≥4) |
| `Sampling.AdaptiveLevelSet.max_new_per_generation` | [6.9](#69-adaptivelevelset) | no | `max_points // 10` |
| `Sampling.AdaptiveLevelSet.k_ref` | [6.9](#69-adaptivelevelset) | no | `4` |
| `Sampling.AdaptiveLevelSet.neighbor_graph` | [6.9](#69-adaptivelevelset) | no | `auto` |
| `Sampling.AdaptiveLevelSet.knn_k` | [6.9](#69-adaptivelevelset) | no | `4 * d` |
| `Sampling.AdaptiveLevelSet.slice_pairs` | [6.9](#69-adaptivelevelset) | no | all pairs (d≥4) |
| `Sampling.AdaptiveLevelSet.simplify_tolerance` | [6.9](#69-adaptivelevelset) | no | off |

### 3.3 `Mapper` / `LibDeps` / `Calculators` / `Operas` / `Likelihood`

| Path | §  | Required | Default |
|---|---|---|---|
| `Mapper.type` | [7](#7-mapper-top-level-optional) | no | auto-derived |
| `Mapper.keys` | [7](#7-mapper-top-level-optional) | for `identity` | — |
| `Mapper.variables` | [7](#7-mapper-top-level-optional) | for `distribution`/fallthrough | — |
| `LibDeps.path` | [8.1](#81-libdepsmodules--token-path-registry) | no | project root |
| `LibDeps.Modules[].name` | [8.1](#81-libdepsmodules--token-path-registry) | yes | — |
| `LibDeps.Modules[].{path,source,installation}` | [8.1](#81-libdepsmodules--token-path-registry) | no | `<LibDeps.path>/<name>` |
| `LibDeps.registered_executables[].name` | [8.2](#82-libdepsregistered_executables) | yes | — |
| `LibDeps.registered_executables[].source` | [8.2](#82-libdepsregistered_executables) | yes | — |
| `LibDeps.registered_executables[].resolution` | [8.2](#82-libdepsregistered_executables) | no | `direct_path` |
| `Calculators.Pools.<name>` / `pools.<name>` | [9.1](#91-calculatorspools-alias-pools) | no | per-module `make_paraller` or `1` |
| `Calculators.Archiver.*` | [9.2](#92-calculatorsarchiver) | no | see table |
| `Calculators.Cleanup.*` | [9.3](#93-calculatorscleanup) | no | see table |
| `Calculators.Modules[].name` | [9.4](#94-calculatorsmodules--one-external-calculator) | yes | — |
| `Calculators.Modules[].{required_modules,clone_shadow,path,source,symlink_name,timeout,make_paraller,env_setup,installation,initialization}` | [9.4](#94-calculatorsmodules--one-external-calculator) | no | see table |
| `Calculators.Modules[].execution.*` | [9.5](#95-execution-block) | yes in practice | see table |
| `Operas.Modules[].name` | [10](#10-operas) | effectively yes | `Operas<i>` |
| `Operas.Modules[].operator` | [10](#10-operas) | **yes** | — |
| `Operas.Modules[].{call_mode,timeout_sec,timeout,kwargs,input,output}` | [10](#10-operas) | no | see table |
| `Likelihood.expressions[].name` | [11](#11-likelihood) | no | `"LogL"` |
| `Likelihood.expressions[].expression` | [11](#11-likelihood) | **yes** | — |

---

## 4. Top-Level Keys

| Key | Type | Default | Consumed by |
|---|---|---|---|
| `project_name` | str | `Scan.name` | run_summary only (`core.py:646`) |
| `Scan` | map | — | `name` is the only key read |
| `scan_name` | str | `"default"` | fallback when `Scan.name` absent |
| `task_result_dir` | str | `<root>/outputs/<scan>` | output root override |
| `run_id` | str | uuid4 | run identity in info/run_summary |
| `EnvReqs` | map | — | V1 requirements plus V2 scheduling/defaults (§4.1, §5) |
| `Sampling` | map | — | sampler selection + config (§6) |
| `Mapper` | map | auto-derived | Worker u→x mapper (§7) |
| `LibDeps` | map | — | token paths + registered executables (§8) |
| `Calculators` | map | — | modules, pools, archiver, cleanup (§9) |
| `Operas` | map | — | `Modules` list (§10) |
| `Likelihood` | map | — | `expressions` list (§11) |

Reserved (stamped by the loader, will be overwritten): `task_yaml`, `task_root`,
`project_root`, `task_result_dir`, `scan_name`.

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

`Sampling.Method` selects the sampler via `Distributor.set_method`. Implemented methods:
stateless **`Bridson`, `Random`, `Grid`, `CSV`** (resume-capable fixed sets) and stateful
**`AdaptiveLevelSet`** (feedback barrier, `stateless=False`; see §6.9). Any other value
raises `NotImplementedError` — see §13 and Appendix A.9 for exactly which error message
you get and when. Alternatively `Sampling.mode: check_modules` (or `check-modules`) runs the
fixed-point calculator smoke path (§6.7), bypassing `Method` entirely.

### 6.1 Keys Common to Bridson / Random / Grid

| Key | Required | Default | Notes |
|---|---|---|---|
| `Variables` | yes | — | empty list raises `ValueError`; schema in §6.2 |
| `Seed` (alias `seed`) | no | `0` | `0` = unseeded for Random/Bridson |
| `selection` | no | — | sympy boolean over **physical** params; see box below |
| `LogLikelihood` | no | — | alias of `Likelihood.expressions`, lower precedence (§6.8, §11) |

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
- name: x                  # required; empty/missing name -> entry silently dropped
  description: "..."       # informational only
  distribution:
    type: Flat             # default Flat
    parameters: {...}
```

| `type` | Parameters | Mapping u∈[0,1] → x |
|---|---|---|
| `Flat` | `min`, `max` | `min + (max-min)·u` |
| `Log` | `min`, `max` (>0) | `exp(log min + (log max − log min)·u)` |
| `Normal` | `mean`, `stddev` | `mean + stddev·Φ⁻¹(u)` |
| `Log-Normal` | `mean`, `stddev` | `exp(mean + stddev·Φ⁻¹(u))` |
| `Logit` | `location` (0), `scale` (1) | `logit(u)·scale + location` |

Extra per-method parameters on the same `parameters` map: `num` (Grid, **required**, points
per dimension — §6.5), `length` (Bridson, **required** — §6.3, and see the code
inconsistency in Appendix A.11).

Missing required distribution parameters raise raw `KeyError` at sampler startup (not a
schema message).

### 6.3 Bridson

Poisson-disk / blue-noise sampling — good spatial coverage without a regular grid's
axis-aligned artifacts.

| Key | Required | Default | Notes |
|---|---|---|---|
| `Radius` | **yes** (`KeyError`) | — | minimum u-space distance between accepted points |
| `MaxAttempt` | **yes** (`KeyError`) | — | candidates per active point during placement (`k`) |
| `MaxWorker` | no | `EnvReqs.V2.workers` | max in-flight proposals |
| `Variables[].distribution.parameters.length` | **yes** (`KeyError`) | — | u-space box edge for that dimension; see Appendix A.11 for the code inconsistency with the resume/replay coordinate helpers |

Minimal recipe (adapted from `tests/parity_project/bridson_opera.yaml`):

```yaml
Sampling:
  Method: Bridson
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

| Key | Required | Default | Notes |
|---|---|---|---|
| `Point number` (alias `point_number`) | **yes** (`ValueError`) | — | note the space in the primary key name |

Minimal recipe (adapted from `tests/parity_project/random_opera.yaml`):

```yaml
Sampling:
  Method: Random
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
  Seed: 0
  Variables:
    - name: x
      distribution: {type: Flat, parameters: {min: 0.0, max: 1.0, num: 3}}
    - name: y
      distribution: {type: Flat, parameters: {min: 0.0, max: 1.0, num: 3}}
```

### 6.6 CSV

Replay physical points from a file — no u→x mapping is performed (`Mapper` defaults to
`none`; §7).

| Key under `Sampling.CSV` | Required | Default | Notes |
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
  CSV:
    path: "&J/data/check_modules_points.csv"
    variables: [x, y]
```

### 6.7 `check_modules` (`Sampling.mode: check_modules`)

A fixed-point calculator smoke path — runs the calculator/opera/likelihood chain on rows from
a CSV instead of proposing new points. Independent of `Sampling.Method`, but see the ordering
gotcha in Appendix A.12: a stray invalid `Sampling.Method` value still breaks bootstrap even
in this mode.

| Key | Required | Notes |
|---|---|---|
| `data` (alias `points_csv`) | **yes** | CSV with columns `uuid,x,y` — **the x/y pair is hard-coded** (`core.py:157`, Appendix A.7) |

Minimal recipe (adapted from `tests/parity_project/check_modules.yaml`):

```yaml
Sampling:
  mode: check_modules
  data: "&J/data/check_modules_points.csv"
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

### 6.9 AdaptiveLevelSet

Feedback-driven level-set tracer (`jarvishep2/Sampling/adaptive_level_set.py`). Registered
`stateless=False`; runs only on the internal Redis runtime and automatically enables Worker
`publish_feedback` (no YAML flag). Full annotated recipe and tables:
[`YAML-Example/ADAPTIVE_LEVEL_SET.md`](YAML-Example/ADAPTIVE_LEVEL_SET.md).

**Constraints**

| Constraint | Rule |
|---|---|
| Dimension | `2 ≤ len(Variables) ≤ 5` else `ValueError` at `set_config` |
| Sub-block | `Sampling.AdaptiveLevelSet` (alias `adaptive_level_set`) **required** mapping |
| Gen-0 | d ≤ 4 Bridson Poisson-disk on unit cube; d = 5 Sobol |
| Neighbor graph | `auto` → Delaunay (d ≤ 3) / kNN (d ≥ 4); explicit `delaunay` / `knn` allowed |
| Output | `<task_result_dir>/levelset.json` plus normal SAMPLE/DATABASE archive |

**Shared `Sampling` keys used by ALS**

| Key | Required | Default | Notes |
|---|---|---|---|
| `Method` | **yes** | — | must be `AdaptiveLevelSet` |
| `Variables` | **yes** | — | length 2–5; same distribution catalog as §6.2 |
| `Seed` / `seed` | no | `0` | Master `SeedSequence` for all generations |
| `selection` | no | — | physical-param filter before submit (dropped, not failures) |
| `LogLikelihood` | no | — | alias of `Likelihood.expressions` (§6.8) |

**Keys under `Sampling.AdaptiveLevelSet`**

| Key | Required | Default | Notes |
|---|---|---|---|
| `target_expression` | **yes** | — | non-empty sympy string over returned observables (+ variable names) |
| `target_value` | **yes** | — | float level-set constant `f(obs) = target_value` |
| `contour_precision` | no | `0.01` | max ‖u_i − u_j‖ over *known* crossing edges |
| `function_tolerance` | no | `0.05` | max \|f_i − f_j\| over known crossing edges; both must hold to converge |
| `initial_radius` | no | `0.08` | gen-0 spacing scale in u-space |
| `refinement_factor` | no | `0.5` (d≤3) / `0.65` (d≥4) | refine radius = `initial_radius * factor^generation` |
| `max_generations` | no | `25` | int ≥ 1; refine round limit |
| `max_points` | no | `5000` (d≤3) / `20000` (d≥4) | int ≥ 10 hard sample budget |
| `max_new_per_generation` | no | `max_points // 10` | int ≥ 1 per-generation refine budget |
| `k_ref` | no | `4` | candidates drawn per crossing edge |
| `neighbor_graph` | no | `auto` | `auto` \| `delaunay` \| `knn` |
| `knn_k` | no | `4 * d` | kNN degree when graph is kNN |
| `slice_pairs` | no | all unordered pairs | d ≥ 4 only; 2-D projections in `levelset.json` |
| `simplify_tolerance` | no | off | d=2 polish hook (parsed; polyline chaining always runs for d=2) |

Minimal recipe (opera-only circle):

```yaml
EnvReqs:
  V2:
    workers: 2
    batch_size: 8
Sampling:
  Method: AdaptiveLevelSet
  Seed: 7
  Variables:
    - name: x
      distribution: { type: Flat, parameters: { min: 0.0, max: 1.0 } }
    - name: y
      distribution: { type: Flat, parameters: { min: 0.0, max: 1.0 } }
  AdaptiveLevelSet:
    target_expression: "r2"
    target_value: 0.04
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
```

---

## 7. `Mapper` (top-level, optional)

Explicit control of the Worker-held u→x mapper. When absent, derived automatically
(`worker_config._default_mapper`):

- `Sampling.Method: CSV` → `type: none` (params travel in `opera_params`)
- `Sampling.Variables` present → `type: distribution` with those variables
- otherwise → `type: identity, keys: [x, y]` (test fallback)

| `type` | Extra keys | Behavior |
|---|---|---|
| `distribution` | `variables` | V1-compatible distribution mapping (§6.2 formulas) |
| `identity` | `keys` | `u[i]` passed through as `keys[i]` |
| `none` | — | no mapping; Sample must carry `opera_params` |
| *anything else* (`flat`, or omitted `type`) | `variables` | falls through to `FlatUMapper` (linear `min + u·(max-min)`, reading `distribution.parameters.min/max` directly, no `Φ⁻¹` support); **no variables → mapper silently `None`** |

`flat` is only reachable by explicitly writing `Mapper: {type: flat, ...}` (or any
unrecognized `type` string) — auto-derivation never produces it.

---

## 8. `LibDeps`

### 8.1 `LibDeps.Modules` — token path registry

Defines `${LibDeps:<name>}` tokens (Phase 1). Path resolution per entry, **first hit wins**:

```
1. Modules[].installation.path
2. Modules[].installation.source
3. Modules[].path
4. Modules[].source
5. <LibDeps.path>/<name>                (fallback)
```

`LibDeps.path` itself defaults to the project root and supports `&J/`.

Unknown `${LibDeps:name}` references raise `KeyError` at Phase 1. There is **no installation
executor** in V2 — the block only provides paths (V1's install pipeline is not ported).

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

---

## 10. `Operas`

Every dynamically discovered Jarvis-Operas function is callable in shared expressions by its
qualified registered name, for example `math.add(x, y)`. Persisted user functions and installed
`jarvis_operas.core` / `jarvis_operas.user` entry points are discovered independently in each
spawn Worker. There is deliberately no V2-only function-registration or alias YAML block.

`Operas.Modules[]` — in-process Python operators (no subprocess, no staging — runs directly
in the Worker, once imported at startup).

| Key | Required | Default | Notes |
|---|---|---|---|
| `make_paraller` | no | ignored | retained V1 spelling/shape; V2 Worker and workflow-layer concurrency own scheduling |
| `name` | effectively yes | `Operas<i>` | dict key for the execution plan |
| `operator` | **yes** (`KeyError`) | — | importlib dotted callable first, then optional Jarvis-Operas registry name; resolved once per Worker |
| `call_mode` | no | `call` | intended `call` \| `acall`; **unknown values currently fall through to sync call** (A.17) |
| `timeout_sec` (alias `timeout`) | no | none | thread-based timeout; the runaway thread is **not** killed |
| `kwargs` | no | `{}` | static kwargs; `observables` is always injected, and every input observable is also passed as a kwarg |
| `input` | no | pass-through | `"x"` (copy), `{name, expression}` (shared expression language; compiled once per Worker and reused per Sample), or `{name, entry}` (dotted copy) |
| `output` | effectively yes | `[]` | `{name, entry}`; **an empty list discards the entire result** |

The operator must return a `Mapping`, else `TypeError` → Sample fails.

At Worker startup, every Operas input expression is `sympify`/`lambdify` compiled once and the
operator is resolved once. Each Sample evaluates those cached callables and invokes the cached
operator. Likelihood expressions use the same compile-once-per-Worker lifecycle. Calculator
Dump-variable expressions use the same class with a Worker-process cache, compiling on first
use; AdaptiveLevelSet targets and sampler selections use control-process contexts.

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
and AdaptiveLevelSet; the YAML structures remain unchanged. See
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

## 11. `Likelihood`

```yaml
Likelihood:
  expressions:
    - name: LogL_Z
      expression: LogGauss(z, 1.0, 0.1)
```

| Key | Required | Default | Notes |
|---|---|---|---|
| `expressions[].name` | no | `"LogL"` | term name; a term named exactly `LogL` becomes the total (§6.8) |
| `expressions[].expression` | **yes** | — | sympy expression string, evaluated over observables |

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

There is **no schema validation layer** (the designed `ConfigLoader` + jsonschema was dropped;
see `components/config_schema.md`). The small `EnvReqs.V2` surface is intentionally checked
at load time so obsolete Runtime/Redis settings cannot silently change a run.

The planned Agent Bridge `--validate` verb (`DESIGN_AGENT_BRIDGE_2.0.md` §4.2, WP-D8.4) will
surface the "silently coerced" list above as warnings without changing this runtime behavior.

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
   - A **wrong/typo'd** value (e.g. `Method: Dynesty`) never reaches `core.py` at all. It fails
     earlier, inside `bootstrap_distributed_runtime() → init_sampler_from_config()`
     (`core.py:97-100`), via `Distributor.set_method`'s own `NotImplementedError`
     (`"Sampling.Method 'Dynesty' is not implemented in Jarvis-HEP V2 yet"`,
     `distributor.py:43-45`) — accurate, echoes the bad value back.
   - Only a **missing/empty** `Sampling.Method` (with `Sampling.mode` not `check_modules`)
     falls through `init_sampler_from_config`'s falsy-method branch (no exception there) and
     is later caught by `run()`'s own `else:` branch, whose message historically named only
     `Bridson` even though Random/Grid/CSV/AdaptiveLevelSet are equally valid choices.
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
    anything other than Bridson/Random/Grid/CSV/AdaptiveLevelSet, bootstrap fails with
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
