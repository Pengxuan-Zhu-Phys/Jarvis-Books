# Jarvis-HEP V2 — Task YAML Reference (As-Built)

**Status**: as-built reference @ `jarvis2` `d0de31a` (Jarvis-HEP-v2, 207 tests passing)
**Date**: 2026-07-03 · reviewed and corrected 2026-07-04 (Appendix A.11–A.13) ·
restructured 2026-07-05 (this pass: table of contents, key index, per-block normalization,
worked recipes — no factual changes)
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
5. [`Runtime`](#5-runtime)
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
2. Project root inferred by walking up from the YAML's directory looking for a
   `.jarvis-project.json` or `jarvis.project.yaml` marker (falls back to a `bin/` parent, then
   the YAML directory itself). Env overrides `JARVIS_HEP_TASK_ROOT` / `JHEP_TASK_ROOT` exist for
   helpers that call `env_task_root()`.
3. Scan name = `Scan.name` → `scan_name` → `"default"`.
4. Output root = `task_result_dir` (top-level key) → `<project_root>/outputs/<scan_name>`.
5. `Runtime` is normalized with defaults (silent coercion — see §13).
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
config["Runtime"] = {mode, workers, batch_size,
                      sample_artifacts, redis?,
                      FileOperation?, Watchdog?}
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

# ---- runtime --------------------------------------------------------------
Runtime:
  mode: redis                    # auto | redis   (default auto; a distributed run REQUIRES redis)
  workers: 4                     # int >= 0       (default 0; <=0 is coerced to 1 at factory start)
  batch_size: 256                # int > 0        (default 256; sampler submit-group size)
  sample_artifacts: auto         # auto | always | never  (default auto; per-sample dir/log policy)
  redis:                         # OMITTED/EMPTY -> in-process fakeredis (see Appendix A.1 trap)
    host: 127.0.0.1              #   default localhost
    port: 6379                   #   default 6379
    db: 0                        #   default 0
    # url: redis://…             #   takes precedence over host/port/db
    # codec: json                #   json | msgpack (default json)
  FileOperation:
    delete_method: shutil        # shutil | rm    (default shutil)
  Watchdog:
    enabled: true                # default true
    stale_sec: 30.0              # default 30.0, floor 1.0
    poll_interval_sec: 1.0       # default 1.0, floor 0.1
    max_sample_retries: 3        # default 3, floor 0
  # Subprocess: {...}            # parsed but NEVER consumed (dead key — Appendix A.2)

# ---- sampling (required for a runnable task) --------------------------------
Sampling:
  Method: Bridson                # Bridson | Random | Grid | CSV  (anything else -> NotImplementedError)
  # mode: check_modules          # special task type; replaces Method (needs `data`)
  # data: "&J/points.csv"        # check_modules input CSV (alias: points_csv)
  Seed: 42                       # int (alias: seed); default 0
  selection: "x + y < 1.5"       # optional sympy bool expr over physical params
  Variables:                     # required by Bridson/Random/Grid (not CSV)
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
  # MaxWorker: 4                 # Bridson: max in-flight proposals (default Runtime.workers)
  # "Point number": 500          # Random: REQUIRED sample count (alias: point_number)
  # CSV:                         # CSV method: REQUIRED block
  #   path: "&J/points.csv"      #   REQUIRED; &J/, absolute, or task-YAML-relative
  #   variables: [x, y]          #   column subset (default: all non-uuid columns)
  #   uuid_column: uuid          #   default "uuid"; missing column -> fresh uuid4 per row
  #   delimiter: ","             #   default ","
  #   encoding: utf-8            #   default utf-8
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
    mode: thread                 # thread | process (default thread)
    batch_size: 200              # default 200, must be > 0
    flush_interval_sec: 1.0      # default 1.0, floor 0.05
    strategy: move               # move | copy (default move)
    delete_after_archive: true   # default true
  Cleanup:
    strategy: mv_to_staging      # mv_to_staging | direct (default mv_to_staging)
    staging_dir: null            # default <task_result_dir>/staging
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
      installation:              # clone_shadow install commands (once per pack)
        - cmd: "cp -r ${source}/* ${path}/"    # ${source}/${path} tokens available
          cwd: "${path}"
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
  Modules:
    - name: TrivialEggbox
      operator: jarvishep2.testing.eggbox.eggbox2d_numpy  # REQUIRED dotted callable
      call_mode: call            # call | acall (async) (default call)
      # timeout_sec: 30          # optional (alias: timeout); thread-based timeout
      # kwargs: {shift: 0.5}     # static extra kwargs
      input:                     # optional; default = pass all observables through
        - x                      # str form: pass observable x
        - name: r                # mapping form: computed input
          expression: x + y      #   sympy over observables (sin/cos/exp/log only)
        - name: alias
          entry: some.dotted.key #   or copy via dotted path
      output:                    # REQUIRED to capture anything (unlisted keys dropped)
        - name: z
          entry: z               # dotted path into the returned dict

# ---- likelihood ---------------------------------------------------------------
Likelihood:
  expressions:                   # precedence over Sampling.LogLikelihood
    - name: LogL_Z
      expression: LogGauss(z, 1.0, 0.1)   # sympy; builtins: sin cos exp log sqrt LogGauss
    # semantics: a term literally named LogL is THE total; otherwise LogL = sum of terms
```

---

## 3. Key Index

Flat lookup — every key path documented in this file, with its section and required/default
at a glance. Use this table to jump straight to a key; use §§4–11 for the full behavioral
detail (error types, aliases, code citations).

### 3.1 `Runtime`

| Path | §  | Required | Default |
|---|---|---|---|
| `Runtime.mode` | [5](#5-runtime) | no | `auto` |
| `Runtime.workers` | [5](#5-runtime) | no | `0` |
| `Runtime.batch_size` | [5](#5-runtime) | no | `256` |
| `Runtime.sample_artifacts` | [5](#5-runtime) | no | `auto` |
| `Runtime.redis.url` | [5.1](#51-runtimeredis) | no | — |
| `Runtime.redis.host` | [5.1](#51-runtimeredis) | no | `localhost` |
| `Runtime.redis.port` | [5.1](#51-runtimeredis) | no | `6379` |
| `Runtime.redis.db` | [5.1](#51-runtimeredis) | no | `0` |
| `Runtime.redis.codec` | [5.1](#51-runtimeredis) | no | `json` |
| `Runtime.FileOperation.delete_method` | [5.2](#52-runtimefileoperation) | no | `shutil` |
| `Runtime.Watchdog.enabled` | [5.3](#53-runtimewatchdog) | no | `true` |
| `Runtime.Watchdog.stale_sec` | [5.3](#53-runtimewatchdog) | no | `30.0` |
| `Runtime.Watchdog.poll_interval_sec` | [5.3](#53-runtimewatchdog) | no | `1.0` |
| `Runtime.Watchdog.max_sample_retries` | [5.3](#53-runtimewatchdog) | no | `3` |
| `Runtime.Subprocess.*` | [5.4](#54-runtimesubprocess-dead-key) | no | *(dead key, A.2)* |

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
| `Sampling.MaxWorker` | [6.3](#63-bridson) | no | `Runtime.workers` |
| `Sampling."Point number"` / `point_number` | [6.4](#64-random) | **yes** | — |
| `Sampling.CSV.path` | [6.6](#66-csv) | **yes** | — |
| `Sampling.CSV.variables` | [6.6](#66-csv) | no | all non-uuid columns |
| `Sampling.CSV.uuid_column` | [6.6](#66-csv) | no | `uuid` |
| `Sampling.CSV.delimiter` | [6.6](#66-csv) | no | `,` |
| `Sampling.CSV.encoding` | [6.6](#66-csv) | no | `utf-8` |

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
| `Runtime` | map | all defaults | runtime normalization (§5) |
| `Sampling` | map | — | sampler selection + config (§6) |
| `Mapper` | map | auto-derived | Worker u→x mapper (§7) |
| `LibDeps` | map | — | token paths + registered executables (§8) |
| `Calculators` | map | — | modules, pools, archiver, cleanup (§9) |
| `Operas` | map | — | `Modules` list (§10) |
| `Likelihood` | map | — | `expressions` list (§11) |

Reserved (stamped by the loader, will be overwritten): `task_yaml`, `task_root`,
`project_root`, `task_result_dir`, `scan_name`.

---

## 5. `Runtime`

Normalized by `runtime_config.normalize_runtime_block`. **Invalid values never fail — they
silently fall back to defaults** (see §13).

| Key | Values | Default | Notes |
|---|---|---|---|
| `mode` | `auto` \| `redis` | `auto` | `Jarvis2Core.run` **requires** `redis`; `auto` aborts with `RuntimeError` |
| `workers` | int ≥ 0 | `0` | `init_factory` coerces ≤ 0 to 1 |
| `batch_size` | int > 0 | `256` | sampler submit-group size (`_submit_group`) |
| `sample_artifacts` | `auto` \| `always` \| `never` | `auto` | `auto` materializes per-sample dirs only when a calculator exists or `@Sdir` is referenced; `never` also skips failure artifacts |
| `redis` | map | *absent* | see §5.1 |
| `FileOperation` | map | see below | see §5.2 |
| `Watchdog` | map | see below | see §5.3 |
| `Subprocess` | map | *absent* | see §5.4 — **dead key** |

### 5.1 `Runtime.redis`

| Key | Default | Notes |
|---|---|---|
| `url` | — | `redis.Redis.from_url`; wins over host/port/db |
| `host` | `localhost` | |
| `port` | `6379` | |
| `db` | `0` | |
| `codec` | `json` | `json` \| `msgpack` (msgpack needs the extra dependency) |

**Absent or empty block → the control process builds an in-process `fakeredis` client.** This
is a test convenience with a distributed-run trap — see Appendix A.1.

CLI flags `--redis-host/--redis-port/--redis-db` override the YAML block, but only when at
least one flag differs from its default.

### 5.2 `Runtime.FileOperation`

| Key | Values | Default | Notes |
|---|---|---|---|
| `delete_method` | `shutil` \| `rm` | `shutil` | backend used to delete staging directories after archiving and to clean up transient cleanup/staging paths; invalid value silently falls back to `shutil` |

### 5.3 `Runtime.Watchdog`

Worker liveness (WP-D6.1): a background thread on the control process detects dead/stale
Workers and respawns them, requeuing or failing the in-flight Sample.

| Key | Default | Floor |
|---|---|---|
| `enabled` | `true` | — |
| `stale_sec` | `30.0` | `1.0` |
| `poll_interval_sec` | `1.0` | `0.1` |
| `max_sample_retries` | `3` | `0` |

`enabled: false` disables the watchdog thread entirely (no polling overhead). `stale_sec` is
how long a Worker's heartbeat may go unrefreshed while `status` is `busy`/`starting` before
it's treated as dead; `max_sample_retries` bounds how many times an in-flight Sample is
requeued before being marked `Failed` outright.

### 5.4 `Runtime.Subprocess` (dead key)

Parsed and copied into the normalized `Runtime` block (`runtime_config.py:74-76`) but **never
read by anything downstream** — see Appendix A.2. Do not rely on any sub-key here having an
effect.

---

## 6. `Sampling`

`Sampling.Method` selects the sampler via `Distributor.set_method`. Implemented methods
(= `STATELESS_METHODS`, all resume-capable): **`Bridson`, `Random`, `Grid`, `CSV`**. Any other
value raises `NotImplementedError` — see §13 and Appendix A.9 for exactly which error message
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
| `MaxWorker` | no | `Runtime.workers` | max in-flight proposals |
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
alias of `Likelihood.expressions`. Expressions are compiled with sympy and evaluated over the
observables dict; earlier terms' results are visible to later terms. If a term is literally
named `LogL` it becomes the total; otherwise `LogL = Σ terms`. Available builtins: `sin, cos,
exp, log, sqrt, LogGauss(x, mean, err)` (`inner_func.NUMERIC_MODULES`). Missing observables
raise `KeyError` (the Sample fails). See Appendix A.6 for the sympy-symbol-collision risk.

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

Controls Layer-2 persistence (staging → `SAMPLE/<uuid>/` + `DATABASE/`).

| Key | Values | Default | Notes |
|---|---|---|---|
| `mode` | `thread` \| `process` | `thread` | in-process thread vs a dedicated spawned Archiver process |
| `batch_size` | int > 0 | `200` | Samples buffered before a batch flush |
| `flush_interval_sec` | float ≥ 0.05 | `1.0` | time-based flush trigger for partial batches |
| `strategy` | `move` \| `copy` | `move` | staging → SAMPLE transfer method |
| `delete_after_archive` | bool | `true` | delete the staging directory once archived |

### 9.3 `Calculators.Cleanup`

Controls how a finished Sample's working directory hands off to the Archiver.

| Key | Values | Default | Notes |
|---|---|---|---|
| `strategy` | `mv_to_staging` \| `direct` | `mv_to_staging` | `direct` skips the staging hop |
| `staging_dir` | path | `<task_result_dir>/staging` | *not* the design-doc default sketch — see Appendix A.8 |

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

`Operas.Modules[]` — in-process Python operators (no subprocess, no staging — runs directly
in the Worker, once imported at startup).

| Key | Required | Default | Notes |
|---|---|---|---|
| `name` | effectively yes | `Operas<i>` | dict key for the execution plan |
| `operator` | **yes** (`KeyError`) | — | dotted `module.func`; imported once per Worker |
| `call_mode` | no | `call` | `call` \| `acall` (coroutine) |
| `timeout_sec` (alias `timeout`) | no | none | thread-based timeout; the runaway thread is **not** killed |
| `kwargs` | no | `{}` | static kwargs; `observables` is always injected, and every input observable is also passed as a kwarg |
| `input` | no | pass-through | `"x"` (copy), `{name, expression}` (sympy — only `sin/cos/exp/log`), or `{name, entry}` (dotted copy) |
| `output` | effectively yes | `[]` | `{name, entry}`; **an empty list discards the entire result** |

The operator must return a `Mapping`, else `TypeError` → Sample fails.

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

- **Silently coerced to defaults** (typos vanish): `Runtime.mode`, `sample_artifacts`,
  `Archiver.mode/strategy/batch_size/flush_interval_sec`, `Cleanup.strategy`,
  `FileOperation.delete_method`, all `Watchdog` fields, `workers`, `batch_size`,
  malformed `registered_executables`/`LibDeps`/module list entries (non-mapping or nameless
  items are dropped).
- **Hard errors, mostly raw exceptions**: missing YAML file / non-mapping top level,
  `Sampling.Variables` empty, `Radius`/`MaxAttempt`/`Point number`/`num`/`CSV.path` missing,
  unknown `Sampling.Method`, invalid `selection`, `registered_executables` name/source
  problems, unknown `${LibDeps:…}`, opera `operator` missing/not callable.

There is **no schema validation layer** (the designed `ConfigLoader` + jsonschema was dropped;
see `components/config_schema.md`). A typo like `mode: redsi` degrades to `auto` and only
surfaces later as *"distributed runtime requires Runtime.mode == 'redis'"*.

The planned Agent Bridge `--validate` verb (`DESIGN_AGENT_BRIDGE_2.0.md` §4.2, WP-D8.4) will
surface the "silently coerced" list above as warnings without changing this runtime behavior.

---

## Appendix A — Known Gaps & Design Warts (input for the YAML design review)

1. **`Runtime.redis` omitted ⇒ split-brain.** The control process silently builds an
   *in-process* fakeredis (`core.init_redis`), but spawned Workers re-connect from the same
   (empty) config and reach **real Redis at localhost:6379** (`redis_queue.connect` defaults).
   Tasks are pushed where no Worker looks; the run hangs until `wait_for_results` times out.
   All shipped tests inject an explicit TCP-fakeredis config, so nothing covers the bare-YAML
   path. Recommendation: make `Runtime.redis` required when `mode: redis` + `workers > 0`, or
   fail fast.
2. **`Runtime.Subprocess` is a dead key** — normalized (`runtime_config.py:74`) and never read
   anywhere. Either wire it to `SubprocessRuntimeConfig` (concurrency, log policy) or delete it.
3. **`execution.input[].save` / `output[].save` are dead keys** — present in the parity YAML
   and V1 heritage, never read by `io_json`/`calculator`.
4. **`make_paraller` typo** (should be `parallel`) survives from V1 in module configs and the
   internal `calculator_make_paraller`. V2 is a new CLI with a new package name — the cheap
   moment to fix the spelling (optionally keeping the old key as an alias) is now.
5. **Naming-convention chaos on one surface:** `Point number` (space) vs `MaxAttempt`
   (CamelCase) vs `batch_size` (snake) vs `Seed`/`seed` both accepted; `Pools`/`pools`;
   `timeout` vs `timeout_sec`; `Sampling.LogLikelihood` vs `Likelihood.expressions`;
   `data` vs `points_csv`. Each alias pair is one more thing the docs must explain.
6. **Hard-coded symbol lists / sympy collisions.** Likelihood and opera expression parsing
   pre-declares only `{x, y, z, shift, calc_z, LogL, LogL_Z}`; any *other* observable name is
   sympified freely, so a variable named `E`, `I`, `N`, `pi`, `beta`, `gamma`, `lambda` is
   captured by sympy's built-in constants/functions and produces silently wrong numbers.
   Opera input expressions additionally see only `sin/cos/exp/log` (no `sqrt`/`LogGauss`) —
   inconsistent with the likelihood context.
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
     is later caught by `run()`'s own `else:` branch (`core.py:261-263`), whose message names
     only `Bridson` even though Random/Grid/CSV are equally valid choices.
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
    anything other than Bridson/Random/Grid/CSV, bootstrap fails with `Distributor.set_method`'s
    `NotImplementedError` before the check-modules path is ever reached. Two independent-looking
    knobs (`mode` and `Method`) have a hidden ordering coupling.
13. **`--pid` is a dead CLI flag.** `client.py`'s `build_parser()` defines
    `--pid` ("Attach to a running scan by control PID") but `args.pid` is never read anywhere —
    passing it silently does nothing. Also note: `V2_DISTRIBUTED_PLAN.md`'s "Verification
    Commands" section shows `Jarvis2 <task>.yaml --benchmark 30` and `Jarvis2 <task>.yaml
    --convert`, but neither flag exists in the current `build_parser()` — those are
    planned/aspirational, not implemented.

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
Jarvis2 <task>.yaml            # run (requires Runtime.mode: redis)
Jarvis2 <task>.yaml --check-modules
Jarvis2 <task>.yaml --resume   # skip the 30 s resume prompt
Jarvis2 --monitor [--redis-host H --redis-port P --redis-db N]
Jarvis2 <task>.yaml --pid N    # accepted by argparse but dead — see Appendix A.13
```

Exit codes: 0 ok, 1 run/IO failure or nothing archived, 2 usage / unsupported task.

`--benchmark`/`--convert` (referenced in `V2_DISTRIBUTED_PLAN.md`'s verification commands)
are **not implemented** in the current `client.py` — see Appendix A.13.
