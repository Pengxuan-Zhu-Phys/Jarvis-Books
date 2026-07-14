# Jarvis-HEP V2 — Distributed Runtime Architecture

**Version target**: 2.0.0
**Status**: Active design — single source of truth for the V2 runtime core
**Date**: 2026-06-24
**Baseline**: v1.6.11 + committed M0/M1 (frozen V1 line)
**Supersedes (data plane only)**: the retired throughput-core design `DESIGN_2.0_CORE.md` (archived in the Jarvis-HEP repo under `docs/archive/2026-06_v2_throughput_core/`).

Audience: maintainers and AI coding agents implementing the V2 runtime.
Purpose: consolidate the June 2026 design discussions into one coherent,
implementable architecture, and define what is **new**, what is **reused**, and what is
**retired**.

---

## 0. Consolidation Notice

This document absorbs the following discussion notes (the raw June 2026 design discussions, kept
in the Jarvis-HEP repo); this document is the authority when they disagree.

| Source note | Contributes |
|-------------|-------------|
| `discussions/Jarvis-HEP_Discussion_Summary_2026-06-21.md` | YAML review, two-layer I/O, Worker model, command/env resolution, clone_shadow policy |
| `discussions/Jarvis-HEP_v2_架构升级_新增与重构计划.md` | New vs refactored component inventory, new `Sample`, migration strategy |
| `discussions/Jarvis_HEP_High_Concurrency_Design_Blueprint.md` | `pack_id` traceability, resource-controlled calculators, independent monitor process |
| `discussions/factory_design.md` | TaskFactory = process manager + read-only `op_count`-driven 60 Hz monitor |
| `discussions/worker_design.md` | Long-lived multiprocess Worker, calculator reuse, Opera preload, worker-local Sample lifecycle |
| `discussions/command_execution_design.md` | `delete_method` (shutil vs rm) and a general command/file-op switch |
| `discussions/jarvis_hep_logging_design.md` | Drop loguru → std-`logging`, two-layer logger (top-level + per-Sample file) |

**Three binding decisions** (maintainer, 2026-06-24) set the architecture:

1. **Transport = Redis from the start.** Sampler → Workers communicate through a Redis
   task queue; monitoring is read-only over Redis with an `op_count` change-detection
   mechanism. (A single-machine `multiprocessing.Queue` fallback is *not* the initial
   target, but the queue interface is kept thin enough to swap.)
2. **Primary target = the slow regime.** Acceptance is gated on external-calculator
   throughput, Worker horizontal scaling (toward ~256), NAS archive latency, and
   crash-recovery robustness. In-process Operas throughput must stay reasonable but is
   no longer the headline KPI.
3. **Worker = a long-lived process; Redis is the sole cross-process resource broker.**
   Each Worker is a `multiprocessing.Process` subclass that lives for the whole run (it is
   *not* spawned per task and *not* a thread-pool thread). Workers pull tasks and acquire
   shared resources (calculator concurrency slots, etc.) only through Redis. The
   single-process `mode: thread` path is retained as a zero-dependency fallback and parity
   oracle, not as the production target.

User-facing contracts — the frozen behavior contracts kept in the Jarvis-HEP repo
(`docs/specs/`): YAML schema, CLI surface, HDF5/CSV output, `run_summary`, checkpoint UX —
remain **frozen**; every new key below is optional.

---

## 0.1 Project Lines — V1 Frozen, V2 Independent

**Decision (maintainer, 2026-06-24).** V1 and V2 are two separate product lines that do not
interfere. The existing thread-based runtime is **V1's design**, now finalized.

| | **V1** | **V2** |
|---|---|---|
| Runtime | single-process thread pool (`WorkerFactory` + `ThreadPoolExecutor`) + the M0/M1 work | Redis + long-lived process Workers + async Archiver |
| Version | **frozen at 1.7.4** | 2.0.0-dev |
| CLI entry | `Jarvis` | **`Jarvis2`** (tentative name) |
| Maintenance | **bug-fix only** — no new features, no V2 backports | active development |
| Working tree | current checkout | **a new independent branch, checked out as a git worktree** |

Consequences (binding for the plan):

- The thread-pool runtime is **V1, design-complete**. V2 does **not** ship it as a product
  mode. Earlier drafts called `mode: thread` a "fallback / parity oracle"; **that is
  dropped** — V2 carries no thread runtime.
- V2 is developed on a **completely independent branch** in a **git worktree**, so V1
  maintenance and V2 development never share a working tree or a CLI.
- `Jarvis2` is a distinct CLI entry point so V1 and V2 can be installed side by side without
  collision.
- **Parity oracle changes.** Output parity is verified against **captured V1 golden outputs**
  (DATABASE/SAMPLE/CSV fixtures produced by frozen `Jarvis` 1.7.4), not against a co-resident
  thread runtime. CI runs V2's Redis path against `fakeredis` (no server required).

**Setup (proposed — confirm names/paths before running):**

```bash
# from the V1 checkout
git branch jarvis2                       # new V2 line  (name TBD: jarvis2 | v2-distributed)
git worktree add ../Jarvis-HEP-v2 jarvis2
# → develop V2 in ../Jarvis-HEP-v2 ; fix V1 bugs in the original checkout
```

**Open packaging decisions** (resolve in WP-D0.2 / D1.1):

- `Jarvis2` console-script on the V2 branch: `[project.scripts] Jarvis2 = "jarvishep.client:main"`.
- To install both lines in one environment without pip clobbering, give V2 a distinct
  **distribution name** (e.g. `Jarvis-HEP2`) and/or a distinct **import package**
  (e.g. `jarvishep2`). Recommended: rename the V2 distribution.

> **`docs/` is gitignored** — these design docs do **not** travel to the new worktree via
> git. Copy `docs/design/DESIGN_2.0_DISTRIBUTED.md`, `doV2_DISTRIBUTED_PLAN.md`,
> and `docs/v2/discussions/` into the V2 worktree (or start tracking them on the V2 branch).

> **Branch-name note.** The current git branch is literally named `v2` but is the **V1**
> (thread) line; the real V2 must use a different branch name to avoid confusion.

---

## 1. Motivation — Why The Pivot

The retired throughput-core design (`DESIGN_2.0_CORE.md`) optimized framework overhead in the
**fast regime** (one in-process Operas module, ~800 samples/s). Its SoA-batch + shared-memory-ring
data plane was built, found not to help calculator/NAS-bound workloads, and retired (archived in
the Jarvis-HEP repo, `docs/archive/2026-06_v2_throughput_core/ARCHIVE_NOTE.md`).

The real cost center is the **slow regime**:

- External calculators (MadGraph, SPheno, Rivet, …) take seconds–minutes; the GIL is
  already released during subprocess waits.
- File I/O lands on NAS, where `move`/`rename` (~10–30 ms / 5 MB) vastly beats `copy`.
- Non-thread-safe tools (e.g. MadGraph generators) need **physical directory isolation**
  per Sample (`clone_shadow`).
- The framework's job is to keep **many** calculator slots busy across **many** Worker
  processes, persist results cheaply, and stay observable — not to shave microseconds.

The model the maintainer confirmed:

> **One Worker process handles one Sample at a time. Inside a Sample, same-layer
> calculators (no data dependency) run concurrently.**

This is incompatible with cross-sample SoA batching, so that data plane is gone. The unit
of work is again **one Sample**; "batch" survives only as the **Archiver's** persistence
batch (≈200 finished Samples flushed together).

---

## 2. Architecture Overview

```
            Sampler  (control process; the "brain")
               │  builds lightweight task_dict (uuid, u_coords, execution_plan)
               │  rpush hep:task_queue ; incr hep:task:op_count
               ▼
        ┌──────────────────────── Redis ────────────────────────┐
        │  hep:task_queue (List)        single source of truth   │
        │  calc:free:<name> (List)      calculator free pools     │
        │  hep:worker:status:<id>       worker heartbeats         │
        │  hep:sample:stats             completed/failed counters │
        │  hep:results:<uuid>           result handoff            │
        │  hep:*:op_count               change counters (monitor) │
        └────────────────────────────────────────────────────────┘
           ▲ blpop           ▲ read-only            ▲ results
           │                 │                      │
   ┌───────┴────────┐  ┌─────┴──────────┐    ┌──────┴───────────┐
   │  Worker pool   │  │  TaskFactory   │    │  Async Archiver  │
   │ N×Process      │  │ process mgr +  │    │  process(es)     │
   │  • Calculators │  │ 60 Hz monitor  │    │  • mv staging→   │
   │    (held)      │  │ snapshot       │    │    SAMPLE/, DB   │
   │  • Opera cache │  └───────┬────────┘    │  • xlha/slha     │
   │  • per-worker  │          │             │  • delete staging│
   │    Subprocess  │          ▼             └──────┬───────────┘
   │    Scheduler   │   get_monitor_snapshot()      ▼
   └───────┬────────┘    (←— --monitor dashboard)  HDF5 / CSV / DATABASE
           │ one Sample at a time
           │  Layer 1 calculators run concurrently inside the Sample
           ▼
     mv work_dir → staging   →   handoff to Archiver
```

Component responsibilities at a glance:

| Component | Owns | Does NOT own |
|-----------|------|--------------|
| **Sampler** | parameter proposals, task submission, `op_count` incr | execution, files, calculators |
| **Redis** | the only shared truth (queue, pools, stats, heartbeats) | large objects (only IDs + light dicts) |
| **TaskFactory** | Worker process lifecycle, Redis init, read-only monitor snapshot | task execution, calculator pools, Sample objects |
| **Worker** | one Sample's full lifecycle: materialize → execute → stage → submit result; long-held calculators + Opera funcs | proposing samples, final archiving |
| **Archiver** | batched final persistence (staging → SAMPLE/ + DATABASE), NAS-optimized file ops, format conversion | running calculators, computing likelihood |

---

## 3. Core Data Model — The New `Sample`

The current `jarvishep/sample.py` `Sample(Base)` already has lazy materialization
(`materialize()`, `create_info()`, `logger_name` metadata) from M1. V2 extends it for
Redis serialization and multiprocess IPC; it does **not** rewrite from scratch.

Required additions:

| Field / method | Purpose |
|----------------|---------|
| `uuid` | UUID4 primary key (already present as `info["uuid"]`) |
| `u_coords` (a.k.a. `u_space`) | normalized sampler vector; the only heavy field shipped over Redis |
| lazy `params` | `u → x` mapping deferred to the Worker (global `UMapper`), not serialized |
| `to_task_dict()` | light dict for Redis: `{uuid, u_coords, execution_plan, opera_params, sample_artifacts...}` |
| `from_task_dict(d)` | reconstruct inside the Worker |
| `materialize(worker_id=…)` | Worker-side: create dirs + open per-Sample logger (extends current `materialize`) |
| `to_info_dict()` | result/monitor projection (no logger, no live handles) |

**Invariants**

- The Sampler ships only `u_coords` + a tiny `execution_plan`. The `u → x` map, calculator
  context, paths, and logger are all rebuilt **in the Worker**. (worker_design §1, §2.)
- `logger` is **never** serialized across the process boundary (set to `None` on the wire);
  Workers create and close their own per-Sample logger (logging §3.2, §5).
- The M1 contract holds: lazy Samples expose `info["logger_name"]` so child loggers bind
  without filesystem materialization; opera-only success leaves `SAMPLE/` empty.
- `@SampleID` / `@Sdir` / `@PackID` tokens keep working via an adapter on `info`.

---

## 4. Worker Model

`jarvishep/worker.py` (new). A `multiprocessing.Process` subclass, **long-lived**, started
and managed by TaskFactory. Design source: `discussions/worker_design.md`.

> **Locked decision (maintainer, 2026-06-24).** This is *the* worker model, not one option
> among several. Every Worker is a dedicated, long-lived process (a `multiprocessing.Process`
> subclass) that persists for the entire scan — never per-task, never a pool thread. All
> cross-process coordination (pull the next Sample, acquire a calculator slot) goes through
> Redis. Inside each Worker process the long-lived units are its main loop (one persistent
> execution context for the run) plus its own `AsyncSubprocessScheduler` event-loop thread.
>
> **Shared queue, not static assignment.** Workers compete on **one shared** `hep:task_queue`
> via `blpop`; "dedicated + long-lived" means the *process* lives for the whole run, not that
> Samples are pinned to a fixed Worker. This keeps load balanced while each Worker still
> amortizes calculator-instance/template loading across every Sample it happens to handle.

**Held for the whole process lifetime** (initialized once in `run()`):

- one Redis client (`blpop` to pull tasks),
- a global `UMapper` (`u → x`),
- **one instance per Calculator type** (`CalculatorModule`, templates pre-loaded) — reused
  across all Samples this Worker sees; avoids re-initialization / template reload,
- an **Opera function cache** (`importlib` once at startup),
- a **per-Worker `AsyncSubprocessScheduler`** (`jarvishep/async_subprocess.py`, reused as-is)
  for same-layer calculator concurrency.
- process-local **`ExpressionContext` caches**: Operas inputs and Likelihood terms compile at
  Worker initialization; Calculator/Portal Dump expressions compile on first use. Every later
  Sample evaluates immutable `CompiledExpression` objects.

**Main loop** (one Sample at a time):

```
while running:
    raw = redis.blpop("hep:task_queue", timeout=5)
    if raw is None: continue
    sample = Sample.from_task_dict(json.loads(raw[1]))
    sample.materialize(worker_id=self.worker_id)          # dirs + per-Sample logger
    try:
        for layer in sample.execution_plan:               # workflow DAG, layer by layer
            run all calculators in this layer concurrently # via per-Worker scheduler
            wait for the layer to finish
        compute LogL / assemble info_dict
        status = "Completed"
    except Exception as e:
        status = "Failed"; materialize_failure_artifacts(...)
    finally:
        mv work_dir → staging                              # fast metadata move
        submit_result(sample.to_info_dict(), staging_path) # → Archiver queue / Redis
        sample.close()                                     # close logger
        incr hep:sample:op_count
```

**Calculator concurrency control.** A Worker acquires a calculator "slot" from a Redis
free-pool (`calc:free:<name>`, `blpop`/`rpush`) so that **global** concurrency for a
resource-heavy calculator is capped across *all* Workers — this is the Redis form of the
Blueprint's semaphore + `pack_id` model. `pack_id` is preserved for traceability
(Blueprint §3, §9).

**Isolation** (Discussion §4):

- **Concurrency-safe tools** → register as `LibDeps` and call via symlink/`registered_executables`
  in the Sample dir (no copy).
- **Non-safe tools** (MadGraph generator, …) → `clone_shadow: true`: each Sample gets a
  physical directory copy. Unavoidable cost; declared explicitly in YAML.

**Compatibility hooks** the calculator side must add (worker_design §10):

- `CalculatorModule.preload_templates()` — load templates once at Worker startup.
- `CalculatorModule.execute(sample_info)` — thin synchronous convenience entry reusing the
  existing `run_command` async machinery.

### 4.1 Shared expression runtime

All small YAML formulas use `jarvishep2.expression.ExpressionContext`; no component owns a
second `sympify`/`lambdify` implementation. `ExpressionContext.compile(text, symbols=...)`
returns an immutable `CompiledExpression` containing its sorted dependencies and numerical
callable. The context cache key is expression text plus the explicit symbol contract.

The same standard namespace applies to Operas inputs, Likelihood, Calculator/Portal Dump
variables, sampler `selection`, and AdaptiveLevelSet targets. It contains the complete frozen
V1 lightweight surface: 38 log/exponential, trigonometric, hyperbolic, general-math, and
Gaussian/Heaviside functions plus `Pi/E/Inf` (and V2 `pi/PI` aliases). Domain wrappers retain
their public errors and result coercion. The authoritative inventory is
[`V1_LIGHTWEIGHT_FUNCTION_MIGRATION_2026-07-13.md`](V1_LIGHTWEIGHT_FUNCTION_MIGRATION_2026-07-13.md).
Contexts and compiled callables are process-local runtime state: they are never placed in Redis
or checkpoints and are rebuilt from YAML/config after `spawn` or resume.

---

## 5. Two-Layer I/O and the Async Archiver

The discussion drew a hard line (Discussion §2, §3) between two I/O layers:

- **Layer 1 — inside the Worker/Sample.** A calculator chain's own file traffic
  (`input.json → run → output.json`) is **per-Sample, per-Module, synchronous**, and must
  complete *for the whole chain* before the Sample is done. Work cwd is usually the
  calculator **`path`** (often with `@PackID`); `@Sdir` binds I/O to the sample save dir
  when configured. **Unchanged from v1.**
- **Layer 2 — the Archiver.** After calculators finish and LogL is computed, products
  needed for persistence are finalized **off the critical path**. This is *not* part of
  the calculator chain.

**As-built Layer-2 flow** (defaults as of `64d7486`; staging is optional, not required):

1. On pull, Worker allocates a Redis SAMPLE **bucket** → materialize under
   `SAMPLE/<bucket_id>/<uuid>/` (V1 `sample_directory` parity; default limit 200).
2. Calculator work stays on `@PackID` (or `@Sdir` when requested). Portal `save: true`
   copies products into the sample dir.
3. Worker **`submit_result`** → `hep:archive_queue`, then **`finish_sample_bucket`**
   (active--). **Default handoff is `direct`** — no `staging/` hop.
4. Archiver (default **`mode: process`**) batches DATABASE writes (`SimpleHDF5Writer`).
5. Redis tracks per-bucket `assigned` / `completed` / **`archived`**. A bucket is packed
   to `SAMPLE/<bucket>.tar.gz` **only when** `sealed && active==0 && archived>=assigned`
   (never pack before DATABASE rows are written — early prune was a production hang).
6. Optional **`Cleanup.strategy: mv_to_staging`** restores the old Worker→staging→SAMPLE
   hop when a heavy/out-of-process path needs a buffer.

**Config (optional; EnvReqs.V2 / Calculators / Scan):**

```yaml
EnvReqs:
  V2:
    sample_directory: { enabled: true, limit: 200, width: 6, pack: true }
    cleanup: { strategy: direct }          # or mv_to_staging
    archiver:
      mode: process                        # default process (thread still supported)
      handoff: direct
      pack_buckets: true
      batch_size: 200
      flush_interval_sec: 1.0
      delete_after_archive: true

Calculators:
  Archiver: { ... }                        # same keys; task overrides EnvReqs.V2
  Cleanup: { strategy: direct, staging_dir: null }

Runtime:                                   # internal only after load_task_yaml
  FileOperation:
    delete_method: "shutil"                # shutil | rm
```

`delete_method` is the only piece of `command_execution_design.md` adopted now; a general
command backend is deferred. Final DATABASE contract is frozen (JSON rows in HDF5);
SAMPLE layout is V1-compatible **bucket dirs + optional tar.gz**, not only flat
`SAMPLE/<uuid>/`.

---

## 6. TaskFactory — Process Manager + Read-Only Monitor

`jarvishep/factory.py` (heavy refactor). Source: `discussions/factory_design.md`. The current
singleton thread-pool `WorkerFactory` becomes the **thread-mode fallback**; the new
`TaskFactory` is the process/Redis manager.

New role (factory_design §2):

> **TaskFactory = Worker process manager + Redis initializer + read-only status snapshot
> provider + monitor center.** It does **not** execute tasks, hold calculators, or own
> Sample objects.

**Read/write separation.** During normal operation Workers and the Sampler are the only
**writers** to Redis; TaskFactory only **reads**. This removes the factory from the hot
path entirely.

**`op_count` change-detection** (factory_design §5) — the key monitoring optimization:

- Each writer (`incr hep:worker:op_count`, `hep:calculator:op_count`, `hep:sample:op_count`,
  `hep:task:op_count`) bumps a monotonic counter on every meaningful state change.
- A background thread in TaskFactory compares `current > last_seen` per subsystem and only
  re-fetches a subsystem's full status when it changed. Idle subsystems cost one `GET`.
- `get_monitor_snapshot()` is a pure in-memory `deepcopy` (<0.5 ms target), safe to call at
  **60 Hz** from `--monitor`.

**Lifecycle:** Jarvis start → create `TaskFactory(redis_config)` → init Redis + keys →
start background snapshot thread (~100–200 Hz, headroom over 60 Hz consumers) →
`start_workers(n)` → Workers self-register and `blpop`.

---

## 7. Redis Schema

Single source of truth (factory_design §4, worker_design §8):

```
hep:task_queue                List   # Sampler rpush, Worker blpop
calc:free:<calc_name>         List   # global calculator concurrency pool (blpop/rpush)
hep:results:<uuid>            Hash   # per-Sample result handoff (or via Archiver queue)
hep:worker:status:<id>        Hash   # {status, current_sample, last_heartbeat, pid}
hep:calculator:status         Hash   # {<name>:free, <name>:busy, ...}
hep:sample:stats              Hash   # {running, completed, failed}
hep:worker:op_count           Str    # INCR on worker state change
hep:calculator:op_count       Str    # INCR on calc free/busy change
hep:sample:op_count           Str    # INCR on sample status change
hep:task:op_count             Str    # INCR on task submit
```

**Task schema** (light; the Worker rebuilds everything else):

```json
{
  "uuid": "…",
  "u_coords": [0.1, 0.2, …],
  "execution_plan": [
    {"type": "calculator", "name": "DemoCalc", "layer": 0},
    {"type": "opera", "module": "…", "func": "compute", "layer": 1},
    {"type": "nuisance_optimize", "strategy": "profile_likelihood"}  // future
  ],
  "opera_params": {…},
  "sample_artifacts": "auto",
  "priority": 0,
  "created_at": "…"
}
```

**Serialization.** JSON for v1; `msgpack`/`ormsgpack` is the upgrade path for numpy/UUID
density. **No large objects in Redis** — only IDs and light dicts; products live on disk and
are moved by the Archiver.

**Dependency.** `redis` (server + client) is a hard runtime dependency for `mode: redis`.
Packaging: `pip install 'Jarvis-HEP[distributed]'` pulls `redis` (+ `msgpack`, `aiofiles`);
ship a `docker-compose` snippet for a local Redis. `mode: thread` keeps the zero-dependency
single-process path for laptops/CI.

---

## 8. Command and Environment Resolution

Source: Discussion §5, §6.

**`registered_executables`** — register a tool once, call it everywhere; removes repeated
`ln -sf` boilerplate:

```yaml
LibDeps:
  registered_executables:
    - name: eggboxlk
      source: "${LibDeps:EggBoxSafe}/eggbox"
      resolution: "direct_path"   # direct_path (default, zero cleanup) | symlink
```

**Two-phase `CommandParser`** (new component):

- **Phase 1 — after YAML load (batch pre-resolve):** `registered_executables`, `LibDeps`
  paths, and all static tokens are resolved once.
- **Phase 2 — in the Worker, before execution:** only strong per-Sample tokens remain —
  `@SampleID`, `@Sdir`, `@PackID`.

**`env_setup`** — activate environments (e.g. Rivet's `source rivet_env.sh`) inside an
isolated subprocess:

```yaml
Calculators:
  Modules:
    - name: RivetAnalysis
      env_setup:
        - source: "&J/External/Rivet/rivet_env.sh"
      execution:
        commands: ["rivet --analysis=… input.hepmc"]
```

Implementation: run `source xxx.sh && env`, capture+parse into a dict, merge with
`os.environ`, pass via `SubprocessJob(env=…)` (already supported). **Must cache** — each
source script runs once per Worker, not once per Sample.

---

## 9. Logging

Source: `discussions/jarvis_hep_logging_design.md`. **Drop loguru**; use a thin std-`logging`
wrapper. Loguru's global-sink model fights multiprocess + per-Sample files.

**Two layers:**

- **Top-level `JarvisLogger`** (process-level): Worker / Factory / Sampler / core summaries
  → console + `logs/jarvis_*.log` (rotation). High-level only ("started Sample X", "done in
  14 s", queue state).
- **Per-Sample `SampleLogger`** (detailed): one file per Sample under
  `SAMPLE/<bucket>/<uuid>/Sample_running.log`, full trace (materialize → each calculator
  call → exception → result). Created and closed **inside the Worker**; never passed across
  processes. Lazy path uses `BufferedSampleLogger` (bounded in-memory events); success discards
  the buffer, failure replays into the sample file (invariant #9).

**Sample command log ownership (V1 contract, V2 target):**

- `SampleLogger` is the sole owner of per-sample command formatting and sinks (file always;
  terminal only when check-modules sets a local `sample_console` flag → `console=True`).
- The subprocess scheduler streams calculator header / raw stdout-stderr / raw command summary
  into `stream_logger` in that order, and must not become a second formatter.
- Production scans stay file-only; check-modules may mirror the **same** rendered text to the
  terminal via a process-wide console lock. See [`components/logger.md`](components/logger.md)
  §3–§4.

V2 reimplements the sample sink (`jarvishep2/sample_logger.py`) rather than importing V1; the
file format and failure-replay semantics stay frozen. `colorlog` is an optional pretty-console
dependency for the **top** layer; `QueueHandler` keeps top-level file sinks non-blocking under
load. Sample console mirroring is intentionally outside that queue.

---

## 10. Checkpoint / Resume Under The Distributed Model

The minimal-checkpoint design (`DESIGN_CHECKPOINT_1.7.0.md`) still governs *what* a
checkpoint contains. New concerns:

- **In-flight tasks are not serialized.** On resume, undelivered `hep:task_queue` entries
  and any Worker-held Samples are discarded and re-proposed (the Sampler is the source of
  truth for what still needs doing). Matches the old "never restore inflight" rule.
- **Sampler state** (RNG, chains, ready queue) checkpoints as today, from the control
  process — unaffected by Redis.
- **Worker / calculator slot state** is reconstructible: on resume, Redis pools are rebuilt
  from config; calculator installs are idempotent (re-use installed slots).
- **Open question (§13):** whether the queue checkpoints to Redis RDB/AOF or stays
  file-based with the Sampler re-driving. Default: **file-based sampler checkpoint + drain
  Redis on resume**, no Redis persistence dependency.

Checkpoint UX (30 s heartbeat, `state.pkl` location, resume prompt) stays frozen.

---

## 11. What Is Reused (Do Not Rewrite)

- **All samplers** (`jarvishep/Sampling/`): Dynesty, MultiNest, the MCMC family, Bridson,
  MALA/HMC/NUTS, RLTPMCMC, … Change: emit `task_dict` to Redis instead of
  `factory.submit_task(sample_info)`; the `u_coords` proposal logic is untouched.
- **Likelihood / Operas / Calculator modules** (`jarvishep/Module/`): execution logic kept;
  add `preload_templates()` + `execute(sample_info)` to calculators; Operas funcs get cached.
- **`AsyncSubprocessScheduler`** (`jarvishep/async_subprocess.py`): reused **per Worker** for
  same-layer calculator concurrency — exactly the role the discussion assigns it
  (Discussion §7).
- **Workflow / flowchart** (`jarvishep/workflow.py`): DAG + semantic flowchart export; feeds
  `execution_plan` (with explicit/auto layering).
- **HDF5 / CSV / DATABASE** output structure: frozen; the Archiver is the new writer transport.
- **Project scaffolding**, config loader, distributor, run_summary skeleton.
- **M0/M1 work**: lazy materialization, `BufferedSampleLogger`, benchmark mode — all live.

---

## 12. What Is Retired

| Retired | Replaced by |
|---------|-------------|
| Cross-sample SoA `StructuredBatch` (inflight = batch) | inflight = one Sample; "batch" only in the Archiver |
| SPSC shared-memory ring buffers | Redis task queue (`blpop`/`rpush`) |
| Shared-memory coordination region (`JARVIS_COORD_FORMAT`) | Redis hashes + `op_count` counters |
| `JarvisState` shm snapshot (`jarvis-monitor-{pid}`) | TaskFactory `get_monitor_snapshot()` over Redis |
| Vectorized-opera fast-regime throughput gates | slow-regime gates (calculator throughput, scaling, archive latency) |
| Single-process thread pool | **V1's design, frozen at 1.7.4** — not carried into V2 (see §0.1); V2 is Redis + process Workers only |

Retired ideas are archived in the Jarvis-HEP repo (`docs/archive/2026-06_v2_throughput_core/`).

---

## 13. Open Questions

1. **Layer declaration.** Auto-derive same-layer calculators from `required_modules`, or
   declare parallel layers explicitly in YAML? (Discussion §8.1) — proposed: auto first,
   explicit override later.
2. **Worker main loop.** Strictly synchronous per layer, or keep some asyncio for
   layer-internal concurrency only? (Discussion §8.4) — proposed: async **only** inside a
   layer, sequential across layers.
3. **`env_setup` field** new vs folded into `initialization`. (Discussion §8.3)
4. **Nuisance-parameter scan inside the Worker** — Sampler proposes interest params only;
   Worker runs the nuisance optimization loop (worker_design §7). Interface to design.
5. **Queue durability** — Redis RDB/AOF vs file-based sampler re-drive (§10).
6. **Packaging** — is `redis` a hard core dep or `[distributed]` extra? (proposed: extra;
   `mode: thread` needs no Redis.)
7. **Monitor independence** — `--monitor` attaches to Redis directly, so it can be a fully
   independent process on another host (Blueprint §6); confirm key/ACL scheme.

---

## 14. Mapping To The Development Plan

This design is executed by [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md),
milestones **D0–D7**. Per-class detail (code structure, member functions, tests/verification)
lives in [`components/`](components/); [`components/V1_TO_V2_MAP.md`](components/V1_TO_V2_MAP.md)
is the full V1→V2 coverage checklist (every V1 module → own-doc / reused / dissolved / retired).

| Milestone | Delivers | Design §|
|-----------|----------|---------|
| D0 Foundations | new `Sample` + Redis schema/wrapper + std-logging module + `[distributed]` extra | §3, §7, §9 |
| D1 Single-Worker MVP | TaskFactory.start + one Worker (blpop→execute→result), parity vs thread mode | §4, §6 |
| D2 Multi-Worker + calculators | N Workers, held calculator instances, Redis free-pool, clone_shadow, layer concurrency | §4, §7 |
| D3 Command & env | `registered_executables`, two-phase `CommandParser`, `env_setup`, `delete_method` | §8, §5 |
| D4 Async Archiver | staging mv + Archiver process, batch persistence, output parity | §5 |
| D5 Monitoring | `op_count` 60 Hz snapshot + `--monitor` dashboard + run_summary | §6 |
| D6 Resume + failure | heartbeats, dead-Worker respawn, in-flight re-queue, RNG spawning, checkpoint | §10 |
| D7 Acceptance | slow-regime gates: scaling, archive latency, 256-Worker chaos, parity | §1, §12 |
