# V2 Distributed Runtime — Development Plan (Agent Execution Playbook)

Last updated: 2026-06-28 (D1.1 closed — `TaskFactory` op_count-gated monitor + opera-only
single-Worker path green under `fakeredis` + captured V1 golden parity).
Audience: **AI coding agents** (Claude Code, Codex, Grok, …) and maintainers.
Status: active execution plan for [`DESIGN_2.0_DISTRIBUTED.md`](DESIGN_2.0_DISTRIBUTED.md).
Scope: **V2 only** — a fully independent line (new branch + git **worktree** + **`Jarvis2`** CLI). V1 (`Jarvis`, thread pool) is **frozen at 1.7.4, bug-fix only** (design §0.1); never land V2 work on the V1 line.
Purpose: decompose the distributed (Redis + process-Worker + async-Archiver) design into
ordered, session-sized work packages with explicit preconditions, real file paths,
acceptance gates, and rollback switches, so any session can pick up the next package cold
and execute it safely.

> **Predecessor.** The throughput-core plan (`V2_DEVELOPMENT_PLAN.md`, M0–M5) is archived in the
> Jarvis-HEP repo (`docs/archive/2026-06_v2_throughput_core/`). Its **M0 + M1 shipped and are live**
> on the frozen V1 line; its M2–M5 are retired. Do not execute M2–M5.

---

## 0. Agent Protocol — How To Use This Document

1. **Find your task.** Read the Progress Ledger (§3). Your task is the first work package
   whose status is `todo` and whose `Depends on` entries are all `done`. If the user named a
   WP, verify its dependencies are `done` first.
2. **Read before coding.** In order: `CLAUDE.md`, the design sections in the WP's *Design
   refs*, the matching **per-class design in `docs/v2/components/`** (structure, member
   functions, tests — see `components/V1_TO_V2_MAP.md` for which doc), the relevant
   `docs/v2/discussions/` sub-design note, and every file in *Files*.
3. **Stay inside the WP.** One WP = one PR = one session. *Out of scope* lines are binding.
4. **Code is ground truth.** If the plan or design contradicts the code (drifted lines,
   renamed symbols), trust the code, do the equivalent change, record the deviation in Notes.
5. **Verify.** Run §5 acceptance commands. A WP is not done with failing tests or unmet
   numbers — report honestly, leave status `in-progress`.
6. **Close out.** Update the ledger row, `Last updated`, and `CLAUDE.md` if the WP changed
   anything it documents (file roles, conventions, CLI, known bugs).
7. **V1 is frozen and separate — do not touch it.** The thread-pool runtime is **V1
   (`Jarvis`, 1.7.4), bug-fix only**: never add V2 features to it, never backport V2 to it.
   V2 lives on its own branch/worktree with the **`Jarvis2`** CLI (design §0.1). Verify V2
   output against **captured V1 golden fixtures**; run CI against `fakeredis` (no Redis
   server). There is no thread-mode fallback inside V2.

When blocked (ambiguous requirement, design conflict touching user-visible behavior,
unreachable gate): stop, write findings into Notes, ask the maintainer. Do not improvise
user-visible behavior.

---

## 1. Hard Invariants (Never Violate)

| # | Invariant |
|---|-----------|
| 1 | Task-YAML schema is frozen. Every new key (`Runtime.mode: redis`, `Calculators.Archiver`, `LibDeps.registered_executables`, `env_setup`, `Runtime.FileOperation`, `logging`) is **optional** with v1-equivalent defaults. Existing YAMLs run unmodified. |
| 2 | V2 ships a **separate CLI entry point `Jarvis2`** (tentative), independent of V1's `Jarvis`. New subcommands (`Jarvis2 worker start`, `Jarvis2 <task>.yaml --monitor --pid N`) live under `Jarvis2`; the frozen `Jarvis` CLI is never modified. |
| 3 | Output contracts frozen: HDF5/CSV structure, `DATABASE/` layout, `run_summary.{json,csv,txt}` per `docs/specs/RUN_SUMMARY_METRICS.md`. The Archiver changes the **transport**, never the **format**. |
| 4 | Checkpoint UX frozen: 30 s heartbeat, `state.pkl` location, resume prompt wording, `--resume`. |
| 5 | V1 (thread pool, `Jarvis` 1.7.4) is **frozen on a separate line** and is **not** a V2 runtime mode. V2 carries no thread path. Parity is verified against **captured V1 golden outputs** (DATABASE/SAMPLE/CSV fixtures); CI runs the Redis path against `fakeredis`. Never land V2 work on the V1 line. |
| 6 | One Worker = one Sample at a time. Parallelism lives **inside** a Sample (same-layer calculators). No cross-sample batching of execution. |
| 7 | Redis carries only IDs + light dicts. No large objects, no pickled live instances, no observ­able blobs — products stay on disk and move via the Archiver. |
| 8 | A `logger` is never serialized across a process boundary. Workers create/close their own per-Sample logger (`logger=None` on the wire). |
| 9 | A failed Sample always leaves a readable log on disk (failure-replay; reuse `materialize_failure_artifacts`, `jarvishep/sample.py`). |
| 10 | `multiprocessing` always uses the **spawn** context. |
| 11 | `pack_id` traceability is preserved end to end (Blueprint §3, §9). |
| 12 | Do **not** rename `SamplingVirtial` (`Sampling/sampler.py:27`) or the YAML key `make_paraller`. |
| 13 | Reuse, don't reimplement: samplers, Likelihood/Operas/Calculator modules, `AsyncSubprocessScheduler`, workflow/flowchart, HDF5/CSV writers, project scaffolding. |
| 14 | If a WP touches a file containing a P0/P1 bug listed in `CLAUDE.md` ("Known Bugs"), fix that bug in the same PR with a test. Do not copy buggy code into new modules. Do not fix bugs in files the WP does not touch. |

---

## 2. Milestone Map

| Milestone | Theme | Exit criterion |
|-----------|-------|----------------|
| **D0** | Foundations: data model, transport, logging | new `Sample` round-trips through Redis; std-logging two-layer in place; **D0.4 (review fixes: Redis namespace/race) + D0.5 (payload validation, spawn-pickling, integration test, polish) both green**; V1 line untouched. D1.1 may start after D0.4; D0 is *closed* only when D0.5 is also green. |
| **D1** | Single-Worker Redis MVP | one Worker pulls from Redis and runs opera + calculator scans; DATABASE parity vs V1 golden fixtures |
| **D2** | Multi-Worker + calculator reuse + concurrency | N Workers, held calculator instances, Redis free-pool, clone_shadow, layer-internal concurrency; scales with workers |
| **D3** | Command & environment resolution | `registered_executables`, two-phase `CommandParser`, `env_setup` cache, `delete_method` |
| **D4** | Async Archiver | staging mv + Archiver process, batched NAS persistence; output parity gate |
| **D5** | Monitoring | `op_count`-driven 60 Hz snapshot + `--monitor` dashboard + run_summary from Redis |
| **D6** | Resume + failure handling | heartbeats, dead-Worker respawn, in-flight re-queue, RNG spawning, distributed checkpoint |
| **D7** | Acceptance | slow-regime gates: worker scaling, archive latency, 256-Worker chaos, parity |

---

## 3. Progress Ledger

Allowed statuses: `todo`, `in-progress`, `done`, `blocked`.

| WP   | Title                                                                                  | Milestone | Depends on       | Status                                 | Date       | Notes                                                                                                                                                                                                                                                                                                                             |
| ---- | -------------------------------------------------------------------------------------- | --------- | ---------------- | -------------------------------------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| —    | M0/M1 (benchmark, lazy materialization, buffered logger, single executor, parity gate) | (V1)      | —                | **done — now part of frozen V1 1.7.4** | 2026-06    | Committed on the V1 line; **not** a V2 WP. V2 reuses the lazy-`Sample`/buffered-logger *ideas*, reimplemented on the V2 branch.                                                                                                                                                                                                   |
| D0.1 | New `Sample` model (uuid/u_coords/task_dict/info_dict)                                 | D0        | —                | in-progress                            | 2026-06-27 | Skeleton landed on `jarvis2`. Review ★★★★☆: dataclass/`to_task_dict` forbidden-key guard/`from_task_dict`/lazy `materialize`+replay all good. Remaining: finish `materialize`/`resolve_token`/`start`/`close`/`record`/`to_info_dict`, `bind_params` placeholder, `execution_plan.type` validation, tests. → tracked in **D0.4**. |
| D0.2 | Redis schema + queue wrapper + serialization + `[distributed]` extra                   | D0        | —                | in-progress                            | 2026-06-27 | Landed on `jarvis2`. Review ★★★☆☆: most complete, but two **hard bugs** — `_calc_free_key` uses `{name}:free` instead of `calc:free:{name}`; `_adjust_calc_counts` reads `HGETALL` **outside** the pipeline (cross-Worker race). → fixes in **D0.4**.                                                                             |
| D0.3 | std-`logging` two-layer module (drop loguru)                                           | D0        | —                | in-progress                            | 2026-06-27 | Landed on `jarvis2`. Review ★★★☆☆: two-layer structure correct, loguru dropped, V1 `SampleLogger`/`BufferedSampleLogger` reused. Remaining: `QueueHandler`/`QueueListener` non-blocking, `key=value` contract binding, child-bind via `logger_name`, tests. → **D0.4**.                                                           |
| D0.4 | D0 review fixes + test backfill (gate D0)                                              | D0        | D0.1, D0.2, D0.3 | todo                                   |            | From the 2026-06-27 code review; closes the D0 hard bugs (Redis namespace + race) + missing methods/tests before D1.                                                                                                                                                                                                              |
| D0.5 | D0 wrap-up: defensive hardening + spawn-pickling + integration test + polish           | D0        | D0.4             | todo                                   |            | 2026-06-27 follow-up review. **Non-blocking for D1.1** (D1.1 may start after D0.4), but **required to close D0**.                                                                                                                                                                                                                 |
| D1.1 | TaskFactory skeleton + Worker process (opera-only MVP)                                 | D1        | D0.4             | **done**                               | 2026-06-28 | All six §WP-D1.1 acceptance gates green under `fakeredis` + spawn (`tests/test_worker_mvp.py`, 17 tests). **Shipped:** `TaskFactory` lifecycle + `op_count`-gated monitor (~120 Hz), explicit `connection_config()` spawn boundary, `shutdown` (monitor stop, snapshot clear, Redis close), `Worker` opera+likelihood pipeline, single `sample` op_count via `submit_result`, SIGTERM+SIGINT graceful stop, Factory read-only monitor path, captured-V1 golden DATABASE parity. **Out of scope (later WPs):** watchdog (D6.1), dashboard/`get_run_metrics` (D5.2). Docs: [`factory.md`](components/factory.md), [`worker.md`](components/worker.md). |
| D1.2 | Calculator in Worker (`preload_templates`, `execute`) + single-Worker parity           | D1        | D1.1             | **done**                               | 2026-06-28 | `jarvishep2/Module/calculator.py` + `worker.py` calculator layer on `jarvis2`. **Shipped:** held `CalculatorModule` per type (`from_config_list` + `preload_templates`), `_run_calculator_step` via local `mint_pack_id` (D2.1 upgrades to Redis free-pool), `@Sdir`/`@PackID` token resolution, `build_execution_plan` (calculator → likelihood), parity vs `tests/fixtures/parity_m1/` + `tests/parity_project` (10-point check_modules scan), `tests/test_worker_calculator.py` (6 tests). **Out of scope:** cross-Worker free-pool, layer concurrency (D2). |
| D2.1 | Multi-Worker pool + held calculators + Redis free-pool + `pack_id`                     | D2        | D1.2             | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |
| D2.2 | Layer-internal calculator concurrency (per-Worker `AsyncSubprocessScheduler`)          | D2        | D2.1             | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |
| D2.3 | `clone_shadow` isolation + LibDeps symlink path                                        | D2        | D2.1             | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |
| D3.1 | `registered_executables` + two-phase `CommandParser`                                   | D3        | D1.2             | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |
| D3.2 | `env_setup` capture-from-source + cache                                                | D3        | D3.1             | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |
| D3.3 | `Runtime.FileOperation.delete_method`                                                  | D3        | —                | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |
| D4.1 | Worker staging mv + Archiver process skeleton + handoff                                | D4        | D2.1             | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |
| D4.2 | Archiver batch persistence (HDF5/CSV/SAMPLE) + output parity gate                      | D4        | D4.1             | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |
| D5.1 | `op_count` writers + TaskFactory background snapshot                                   | D5        | D2.1             | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |
| D5.2 | `--monitor` dashboard + run_summary from Redis                                         | D5        | D5.1             | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |
| D6.1 | Heartbeats + dead-Worker respawn + in-flight re-queue                                  | D6        | D2.1             | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |
| D6.2 | RNG spawning + distributed checkpoint/resume                                           | D6        | D6.1             | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |
| D7.1 | Slow-regime acceptance: scaling + archive latency + chaos + parity                     | D7        | D4.2, D5.2, D6.2 | todo                                   |            |                                                                                                                                                                                                                                                                                                                                   |

Parallelism note: D0.1 / D0.2 / D0.3 are independent and may proceed in any order. D3.3 is
independent of the rest of D3.

---

## 4. Work Packages

Field key — **Goal**: outcome. **Design refs**: `DESIGN_2.0_DISTRIBUTED.md` sections (+ the
`docs/v2/discussions/` note). **Files**: existing files to read/modify; `NEW` marks suggested new
modules (keep the flat `jarvishep/` layout). **Steps**: order. **Accept**: definition of
done. **Rollback**: how it is disabled. **Out of scope**: excluded work.

---

### WP-D0.1 — New `Sample` model

- **Goal**: a `Sample` that round-trips through Redis and rebuilds itself inside a Worker,
  without losing the M1 lazy-materialization contract.
- **Design refs**: §3; `V2/worker_design.md` §3.1, §5; `V2/Jarvis-HEP_v2_架构升级_新增与重构计划.md` §3.1.
- **Files**: `jarvishep/sample.py` (extend, do not rewrite — `materialize`, `create_info`,
  `logger_name` already exist), `jarvishep/base.py` (path tokens), `jarvishep/Sampling/sampler.py`
  (where samples are built), NEW `tests/test_sample_taskdict.py`.
- **Steps**:
  1. Add `u_coords` (alias `u_space`) as the one heavy field; add `execution_plan`/`opera_params`
     carriers on `info`.
  2. `to_task_dict()` → light JSON-able dict (`uuid`, `u_coords`, `execution_plan`,
     `opera_params`, `sample_artifacts`); **exclude** logger and live handles.
  3. `from_task_dict(d)` → reconstruct; `to_info_dict()` → result/monitor projection.
  4. Keep `@SampleID/@Sdir/@PackID` resolution working via the existing `info` adapter.
  5. While here (invariant #14): confirm the `sample.py:140,142` status bug is fixed (it was
     in M1) and that `cunsom_constants`→`custom_constants` (`sample.py:130`) is noted.
- **Accept**: `to_task_dict → from_task_dict` is lossless for the light fields; a lazy
  opera-only Sample still leaves `SAMPLE/` empty on success; existing `tests/test_lazy_materialization.py`
  + `tests/test_failure_replay_log.py` green; full suite green.
- **Rollback**: new methods are additive; thread mode ignores them.
- **Out of scope**: the Worker, Redis, the `UMapper`.

### WP-D0.2 — Redis schema + queue wrapper + serialization

- **Goal**: a thin, tested Redis access layer + the `[distributed]` dependency, with **no**
  runtime wired to it yet.
- **Design refs**: §7; `V2/factory_design.md` §4; `V2/worker_design.md` §8.
- **Files**: NEW `jarvishep/redis_queue.py`, `jarvishep/runtime_config.py` (add `redis` to
  `_VALID_RUNTIME_MODES`, parse a `Runtime.redis` block), `pyproject.toml` (optional extra),
  NEW `tests/test_redis_queue.py` (use `fakeredis` or skip when no server).
- **Steps**:
  1. Define keys (§7) as constants; `push_task/pull_task` (`rpush`/`blpop`), `calc:free`
     acquire/release, results, and `incr_op_count(kind)`.
  2. Serialization helpers (JSON now; leave a `msgpack` hook) supporting numpy/UUID.
  3. `pyproject.toml`: `[project.optional-dependencies] distributed = ["redis>=5.0",
     "msgpack", "aiofiles>=23.0"]`.
  4. Tests behind `fakeredis` (preferred) so CI needs no server; mark a real-Redis
     integration test `skipif` no `REDIS_URL`.
- **Accept**: queue round-trip + op_count tests green under `fakeredis`; `mode: redis` parses
  and stores config but changes no behavior yet; thread mode untouched.
- **Rollback**: unused module; `mode` still defaults to `auto`/`thread`.
- **Out of scope**: Worker, Factory wiring.

### WP-D0.3 — std-`logging` two-layer module

- **Goal**: replace loguru with a std-`logging` wrapper providing top-level + per-Sample
  layers, reusing the existing `SampleLogger`/`BufferedSampleLogger`.
- **Design refs**: §9; `V2/jarvis_hep_logging_design.md`.
- **Files**: `jarvishep/sample_logger.py`, `jarvishep/log_kv.py`, NEW `jarvishep/jlogging.py`
  (top-level setup + adapter), `jarvishep/core.py` (logger init), and a `grep -rn loguru
  jarvishep/` sweep, NEW `tests/test_logging_layers.py`.
- **Steps**:
  1. Inventory loguru usage (`grep -rn "loguru\|logger.bind\|\.success(" jarvishep/`).
  2. `setup_jarvis_logging()` + `get_jarvis_logger(name).bind(**ctx)` over std `logging`
     (+ optional `colorlog`, `QueueHandler` for non-blocking files).
  3. Keep `SampleLogger` (file) + `BufferedSampleLogger` (buffer) as the per-Sample sink;
     the new module wraps, does not replace, the M1 buffer/replay logic.
  4. Migrate call sites incrementally; preserve `docs/specs/LOGGING_CONTRACT.md` format.
- **Accept**: top-level + per-Sample logs produced with the contract format; failure-replay
  test still passes; no `loguru` import remains on the hot path; full suite green.
- **Rollback**: keep loguru import shim for one release if a call site is missed.
- **Out of scope**: Redis central sink, JSON-lines (future).

---

### WP-D0.4 — D0 review fixes + test backfill (gate D0)

- **Goal**: close the **2026-06-27 code review** of the `jarvis2` D0 skeleton — the two RedisQueue
  hard bugs, the unfinished `Sample` methods, and the logging non-blocking/contract details — and
  backfill the D0 unit tests, so D1 starts on a hardened, tested foundation.
- **Origin**: maintainer code review of the D0 commit (Sample + RedisQueue + two-layer logging).
- **Design refs**: components [`redis_queue.md`](components/redis_queue.md),
  [`sample.md`](components/sample.md), [`logger.md`](components/logger.md); DESIGN §3, §7, §9.
- **Files**: `jarvishep2/redis_queue.py`, `jarvishep2/sample.py`, `jarvishep2/logging/toplevel.py`,
  `jarvishep2/logging/sample.py`, `jarvishep2/sample_logger.py` (reused), NEW
  `tests/test_redis_queue.py`, NEW `tests/test_sample_taskdict.py`, NEW `tests/test_logging_layers.py`.
- **Steps**:

  **A. RedisQueue — fix first (these affect Worker + Factory correctness):**
  1. **Namespace bug (highest).** `_calc_free_key(name)` must return `calc:free:{name}` (the
     `CALC_FREE` constant), **not** `{name}:free`. Audit *every* key against `redis_queue.md` §2 and
     pin the exact strings in a test — a wrong free-pool key corrupts `CALC_STATUS` and breaks the
     monitor/external tools.
  2. **Race in `_adjust_calc_counts`.** The `HGETALL CALC_STATUS` read happens **outside** the
     pipeline → cross-Worker count drift. Refactor to atomic **`HINCRBY`** deltas (acquire: free −1 /
     busy +1; release: inverse) inside one pipeline, or `WATCH` + transaction-retry. Free/busy must
     stay consistent under concurrent acquire/release.
  3. `heartbeat`: drop the blanket `str()` coercion (numbers become strings) — keep numeric types or
     JSON-encode the value; `snapshot_raw`/monitor decode accordingly.
  4. Minimal **schema validation** on `push_task`/`submit_result` (required keys: `uuid`, plus
     `u_coords`/`execution_plan`) — fail fast, never enqueue a malformed task.
  5. `release_calc(name, pack_id)`: **use** `pack_id` (assert it matches the busy owner, for
     `pack_id` traceability) or drop the param — do not accept-and-ignore it.

  **B. Sample — complete the `sample.md` §3 surface:**
  6. Finish `materialize`, and add `resolve_token`, `start`/`close`, `record`, `to_info_dict`.
  7. Add a `bind_params(mapper: UMapper)` **placeholder** (interface reserved now; UMapper lands
     later) so the Worker call site is stable.
  8. Validate `ExecutionStep.type` ∈ {`calculator`, `opera`, `likelihood`, `nuisance_optimize`}
     (reject the empty string).

  **C. Logging — align with `logger.md`:**
  9. `setup_jarvis_logging`: wire `QueueHandler` + a `QueueListener` for **non-blocking** file sinks
     (the high-load requirement); stop + drain the listener at shutdown (no lost tail lines).
  10. Context binding emits `key=value` per `LOGGING_CONTRACT.md` (worker_id, sample_uuid, phase…);
      align `bind`/formatting with `logger.md` §3.
  11. Confirm per-Sample child loggers bind via `logger_name` **without** forcing materialization
      (M1 lazy contract).

  **D. Tests — backfill (fakeredis, no server):**
  12. `tests/test_redis_queue.py`: exact key-string assertions (catches the namespace bug),
      `op_count` increments, calc-pool cap (≤ n concurrent acquire; race-free free/busy under two
      "Workers"), pipeline atomicity, codec round-trip (numpy/UUID).
  13. `tests/test_sample_taskdict.py`: `to_task_dict → from_task_dict` round-trip, no-logger-on-wire,
      lazy materialization (opera-only success leaves `SAMPLE/` empty), failure replay.
  14. `tests/test_logging_layers.py`: two layers produced, contract format, failure-replay parity,
      no `loguru` on the hot path.
- **Accept**: both RedisQueue hard bugs fixed — key strings **exactly** match `redis_queue.md` §2,
  and free/busy stays consistent under concurrent acquire/release in a 2-Worker `fakeredis` test;
  `Sample` §3 methods present with round-trip + lazy tests green; logging non-blocking + `key=value`
  contract + failure-replay green; `pytest tests/test_redis_queue.py tests/test_sample_taskdict.py
  tests/test_logging_layers.py` all pass under `fakeredis`; full suite green.
- **Rollback**: revert this WP commit (a fix/hardening pass over D0; no new feature flag).
- **Out of scope**: `UMapper` implementation, TaskFactory/Worker (D1.1), `msgpack` codec (keep JSON;
  note as future).

---

### WP-D0.5 — D0 wrap-up: defensive hardening + spawn-pickling + integration test + polish

- **Goal**: finalize D0 before it is declared closed — strengthen payload validation, **prove
  spawn-safety**, add a Sample↔RedisQueue integration test, and bring the modules to 100% alignment
  with the component-doc tables. **Non-blocking for D1.1** (D1.1 may start after D0.4), but D0 is not
  closed until this is green.
- **Origin**: 2026-06-27 follow-up review — the post-D0.4 "polishing + defensive + test-completion"
  items (core functionality already passed; these de-risk D1).
- **Design refs**: components [`redis_queue.md`](components/redis_queue.md),
  [`sample.md`](components/sample.md), [`logger.md`](components/logger.md); invariants #7 (only light
  dicts in Redis), #8 (no logger across a process boundary), #10 (`spawn`).
- **Files**: `jarvishep2/redis_queue.py`, `jarvishep2/sample.py`, `jarvishep2/logging/toplevel.py`,
  NEW `tests/test_spawn_pickling.py`, NEW `tests/test_d0_integration.py`, extend
  `tests/test_redis_queue.py`.
- **Steps**:
  1. **Payload validation (extends D0.4 A.4).** Replace the minimal key-check in
     `push_task`/`submit_result` with a lightweight **validator** (a small dataclass / `TypedDict` /
     explicit field+type check): a task requires `uuid: str` + (`u_coords` | `execution_plan`); a
     result requires `uuid` + `status`. **Raise** on a malformed payload *before* it enters the queue
     — a bad task must never reach a Worker. Keep pipeline atomicity. Add failing-payload cases to
     `tests/test_redis_queue.py`.
  2. **Sample method alignment.** Finish `bind_params(self, mapper)` as a **clear stub** with a stable
     signature + docstring (UMapper not implemented yet, but the Worker call site must not move); bring
     `to_info_dict()` and `record()` to **100% agreement** with `sample.md` §3 (exact field set; no
     logger/handles).
  3. **Spawn-pickling tests** (invariant #10). NEW `tests/test_spawn_pickling.py`: round-trip
     `Sample` + `ExecutionStep` + a built `RedisQueue` config through
     `multiprocessing.get_context("spawn")` pickling, and a real child process that unpickles a
     `Sample` and rebuilds it; **assert no live handle/logger survives** (invariant #8). This is the
     direct de-risk for D1 Worker spawn.
  4. **D0 integration test.** NEW `tests/test_d0_integration.py`: the full path
     `Sample → to_task_dict → RedisQueue.push_task → pull_task → from_task_dict → materialize`, both
     the **success** path (opera-only ⇒ `SAMPLE/` stays empty) and the **failure** path (failure
     replay leaves a readable log). `fakeredis`, no server.
  5. **Polish.** Complete type hints + docstrings on all new classes; curate per-module `__all__`
     exports; reconcile each module against its component-doc member-function table (100% alignment).
     Optional: a small `bind()` hot-path micro-opt behind the existing `isEnabledFor` guard.
- **Accept**: malformed task/result payloads are rejected (tested); `bind_params`/`to_info_dict`/
  `record` match `sample.md` §3; `tests/test_spawn_pickling.py` green (Sample survives `spawn`, no
  logger/handle leak); `tests/test_d0_integration.py` green (round-trip + failure replay); type
  hints/docstrings/`__all__` complete; full suite green under `fakeredis`. **D0 is declared closed
  when D0.4 and D0.5 are both green.**
- **Rollback**: revert this WP commit (defensive + test hardening; no feature flag).
- **Out of scope**: `UMapper` implementation (D-later), TaskFactory/Worker (D1.1), `msgpack` codec
  (keep JSON).

---

### WP-D1.1 — TaskFactory skeleton + Worker process (opera-only MVP)

- **Goal / scope**: get the **single-Worker opera-only path** running end to end under
  `mode: redis`. The bar is: **one** Worker pulls Samples from Redis, runs only the opera (and
  likelihood) steps of the `execution_plan`, and submits results, with a Factory that can spawn
  it, observe it, and shut it down cleanly. Calculators, multi-Worker concurrency, and the
  Archiver are explicitly **not** in this WP.
- **Design refs**: §2, §4, §6; `V2/factory_design.md`; `V2/worker_design.md`; component docs
  [`components/factory.md`](components/factory.md), [`components/worker.md`](components/worker.md),
  [`components/core.md`](components/core.md).
- **Files** (package is **`jarvishep2/`** on the `jarvis2` branch; the old `jarvishep/` paths are
  the V1 line): `jarvishep2/factory.py` (`TaskFactory`), NEW `jarvishep2/worker.py`
  (`Worker(Process)`), `jarvishep2/core.py` (`Jarvis2Core.init_factory` mode switch +
  spawn/teardown), `jarvishep2/Sampling/sampler.py` (submit `task_dict` to Redis when
  `mode: redis`), NEW `tests/test_worker_mvp.py`.
- **Steps**:
  1. `TaskFactory.start_workers(n=1)` spawns `Worker(Process)` (spawn ctx; only `redis_config` +
     picklable `worker_config` cross the boundary — see `factory.md` §3a); Workers `blpop`
     `hep:task_queue`.
  2. Worker main loop: `from_task_dict → bind_params → materialize → run opera/likelihood layers
     → to_info_dict → submit_result` (`submit_result` bumps `hep:sample:op_count` once).
  3. Sampler: when `mode: redis`, `push_task(sample.to_task_dict())` instead of the thread-pool
     submit.
  4. Result collection: control process drains results → HDF5 writer (the D1.1 `SimpleArchiver`
     writes directly; the batched Archiver lands in D4).
  5. Boot precondition: `mode == redis` brings the Factory up; otherwise `init_factory` is a
     no-op. V2 carries **no thread fallback** (invariant #5) — an unpicklable config / unimportable
     user func is a hard, early failure, not a silent downgrade.
- **Accept** (each line independently verifiable under `fakeredis` + real spawn):
  1. **Spawn** — `TaskFactory.init_redis()` then `start_workers(1)` produces one live `Worker`
     process (`is_alive()` true; refuses a second `start_workers` while it is alive).
  2. **Pull + execute** — a pushed opera-only `task_dict` is pulled and run through the
     `from_task_dict → materialize → opera/likelihood → submit_result` pipeline; the result
     appears on the results path and `op_count("sample")` advances.
  3. **Opera-only parity** — an opera-only reference scan (`Jarvis2`, 1 Worker) produces DATABASE
     records **set-equal** to the captured V1 golden output.
  4. **Graceful shutdown** — `SIGINT`/`SIGTERM` (or `TaskFactory.shutdown()`) lets the in-flight
     Sample finish, joins the Worker (no orphan), and closes Redis.
  5. **Monitor snapshot** — `get_monitor_snapshot()` returns a dict with at least worker
     rows/counts and Redis queue/stats fields (`workers`, `workers_alive`, `task_queue_length`,
     `sample_stats`, `op_counts`); `op_count`-gated incremental fetch with ~120 Hz updater
     (idle ticks avoid `HGETALL`; read path is pure memory).
  6. Full suite green under `fakeredis`.
- **Rollback**: revert the WP commit (V2 has no `mode: thread`; see §5 rollback semantics).
- **Out of scope**: multi-worker + free-pool (D2.1), Archiver batching (D4),
  watchdog/respawn (D6.1), `--monitor` dashboard UI (D5.2).
- **Notes / deviations from the original design** (code is ground truth):
  - `start_workers` receives the live control-process `RedisQueue`, but only its picklable
    connection settings (`redis_config`) are kept for spawn (`factory.md` §3a).
  - `op_count`-gated snapshot landed in D1.1 (pulled forward from D5.1); D5.2 still owns
    `--monitor` dashboard + `get_run_metrics`.
  - No watchdog/respawn yet (D6.1).

### WP-D1.2 — Calculator in Worker + single-Worker parity

- **Goal**: a Worker holds and runs a `CalculatorModule`; single-Worker calculator parity.
- **Status**: **done** on `jarvis2` (2026-06-28).
- **Design refs**: §4; `V2/worker_design.md` §3.2, §4.1, §10.
- **Files**: `jarvishep/Module/calculator.py` (add `preload_templates()`,
  `execute(sample_info)` convenience over existing `run_command`), `jarvishep/worker.py`,
  `jarvishep/moduleManager.py` (per-Sample `execute_workflow` reused inside the Worker),
  NEW `tests/test_worker_calculator.py`, reuse `tests/parity_project`.
- **Steps**:
  1. `_init_calculators()` builds one instance per calculator type at Worker startup and
     `preload_templates()`.
  2. Worker executes the calculator layer of `execution_plan` via the held instance,
     resolving `@Sdir`/`@PackID` per Sample.
  3. Fix in passing if touched (invariant #14): `calculator.py:136` `custom_format` missing
     `self`.
  4. Parity: run `tests/parity_project` (`--check-modules` scale) under `mode: redis` (1
     Worker) vs the pinned thread-mode expectation.
- **Accept**: calculator DATABASE + SAMPLE trees match the captured V1 golden output for 1
  Worker; calculator tests green; full suite green.
- **Rollback**: `mode: thread`.
- **Out of scope**: cross-Worker free-pool, layer concurrency (D2).

---

### WP-D2.1 — Multi-Worker pool + held calculators + Redis free-pool

- **Goal**: N Workers with globally capped calculator concurrency and preserved `pack_id`.
- **Design refs**: §4, §7; Blueprint §3–§5.
- **Files**: `jarvishep/factory.py` (`start_workers(n)`, lifecycle), `jarvishep/worker.py`
  (`calc:free:<name>` acquire/release), `jarvishep/modulePool.py` (slot semantics →
  Redis pool), NEW `tests/test_worker_pool.py`.
- **Steps**:
  1. `start_workers(n)` from `Runtime.workers`; each Worker registers to `hep:worker:status:*`.
  2. Calculator concurrency: acquire a slot from `calc:free:<name>` (`blpop`) before running,
     `rpush` after — caps a heavy calculator across *all* Workers (the Redis form of the
     Blueprint semaphore). Generate a fresh `pack_id` per acquire.
  3. Update `hep:calculator:status` free/busy on acquire/release.
- **Accept**: 2-Worker vs 1-Worker DATABASE fingerprint parity (set-equal UUIDs, identical
  records); a heavy calculator never exceeds its configured `max_concurrent` across Workers
  (assert via status); throughput rises 1→2 Workers.
- **Rollback**: `mode: thread`.
- **Out of scope**: layer-internal concurrency (D2.2), clone_shadow (D2.3).

### WP-D2.2 — Layer-internal calculator concurrency

- **Goal**: same-layer calculators of one Sample run concurrently via a **per-Worker**
  `AsyncSubprocessScheduler` (invariant #6 — parallelism inside a Sample, not across).
- **Design refs**: §4; Discussion §7; reuse `jarvishep/async_subprocess.py`.
- **Files**: `jarvishep/worker.py`, `jarvishep/async_subprocess.py` (one instance per Worker),
  `jarvishep/workflow.py` (layer derivation feeding `execution_plan`), NEW
  `tests/test_layer_concurrency.py`.
- **Steps**:
  1. Each Worker owns one `AsyncSubprocessScheduler`; the Worker runs a Sample's DAG layer
     by layer, dispatching all calculators in a layer concurrently, awaiting the layer.
  2. Layer derivation: auto from `required_modules` (open question §13.1 — explicit override
     deferred).
- **Accept**: a 2-calculator independent layer completes in ≈max(t1,t2) not t1+t2 (timed
  test); output parity with serial execution; full suite green.
- **Rollback**: `mode: thread`, or a config flag forcing serial layers.
- **Out of scope**: cross-Sample concurrency.

### WP-D2.3 — `clone_shadow` isolation + LibDeps symlink path

- **Goal**: physical per-Sample isolation for non-thread-safe tools; symlink path for safe
  tools.
- **Design refs**: §4 (isolation); Discussion §4.
- **Files**: `jarvishep/Module/calculator.py`, `jarvishep/library.py` (LibDeps), `jarvishep/worker.py`,
  config schema, NEW `tests/test_clone_shadow.py`.
- **Steps**:
  1. Honor `clone_shadow: true|false` per calculator: `true` → per-Sample physical copy;
     `false` → symlink/`registered_executables` into the Sample dir.
  2. Optional `isolation: full_shadow|per_sample_dir` left as a future field (do not add now).
- **Accept**: a `clone_shadow` calculator gets an isolated dir per Sample (no cross-talk under
  concurrency); a safe tool runs via symlink with no copy; parity preserved.
- **Rollback**: default `clone_shadow: true` reproduces v1 isolation.
- **Out of scope**: `registered_executables` syntax (D3.1).

---

### WP-D3.1 — `registered_executables` + two-phase `CommandParser`

- **Goal**: register tools once; split static vs per-Sample token resolution.
- **Design refs**: §8; Discussion §5.
- **Files**: NEW `jarvishep/command_parser.py`, `jarvishep/config.py` (parse
  `LibDeps.registered_executables`), `jarvishep/base.py` (token resolution), `jarvishep/worker.py`,
  NEW `tests/test_command_parser.py`.
- **Steps**:
  1. Phase 1 (post-YAML): resolve `registered_executables`, LibDeps paths, all static tokens
     once.
  2. Phase 2 (Worker, pre-exec): resolve only `@SampleID/@Sdir/@PackID`.
  3. `resolution: direct_path` (default) | `symlink`.
- **Accept**: a command using a registered executable resolves correctly in both phases;
  Phase-1 output has no per-Sample tokens; parity preserved.
- **Rollback**: feature is opt-in (absence ⇒ v1 `ln -sf` path still works).
- **Out of scope**: `env_setup`.

### WP-D3.2 — `env_setup` capture-from-source + cache

- **Goal**: activate `source`-based environments (Rivet, …) inside isolated subprocesses.
- **Design refs**: §8; Discussion §6.
- **Files**: `jarvishep/Module/calculator.py`, NEW `jarvishep/env_setup.py`
  (`capture_env_from_source` + cache), `jarvishep/async_subprocess.py` (`SubprocessJob env=`),
  NEW `tests/test_env_setup.py`.
- **Steps**:
  1. `source xxx.sh && env` → parse to dict; merge with `os.environ`; pass via `SubprocessJob(env=…)`.
  2. **Cache** per source script per Worker (run once, not per Sample).
- **Accept**: a Calculator with `env_setup` sees the sourced variables; the script runs once
  per Worker (assert via a counter); parity preserved.
- **Rollback**: absence of `env_setup` ⇒ no change.
- **Out of scope**: general shell backend.

### WP-D3.3 — `Runtime.FileOperation.delete_method`

- **Goal**: configurable delete (`shutil` default, `rm` for mass-delete).
- **Design refs**: §5; `V2/command_execution_design.md`.
- **Files**: NEW `jarvishep/file_ops.py` (`delete_path(path, method)`), call sites in
  Worker cleanup + Archiver, `jarvishep/runtime_config.py`, NEW `tests/test_file_ops.py`.
- **Accept**: both methods delete files/dirs; bad method raises; default is `shutil`.
- **Rollback**: default `shutil` is v1-equivalent.
- **Out of scope**: rsync/tar backends.

---

### WP-D4.1 — Worker staging mv + Archiver process skeleton

- **Goal**: Workers hand finished Samples to a separate Archiver via a fast staging move.
- **Design refs**: §5; Discussion §2, §3.
- **Files**: NEW `jarvishep/archiver.py` (`Archiver(Process)`), `jarvishep/worker.py`
  (mv work_dir → staging, handoff), `jarvishep/core.py` (Archiver lifecycle), NEW
  `tests/test_archiver_handoff.py`.
- **Steps**:
  1. Worker: after result, `mv work_dir → staging/<uuid>` (metadata-only), push
     `(info_dict, product_list, staging_path)` to an archive queue (Redis list or
     control-side queue).
  2. Archiver process consumes the queue (skeleton: move staging → SAMPLE/, no batching yet).
- **Accept**: products land in `SAMPLE/<uuid>/` identical to thread mode; staging emptied;
  clean shutdown drains the queue.
- **Rollback**: `mode: thread` (synchronous archive).
- **Out of scope**: batching, format conversion, async I/O (D4.2).

### WP-D4.2 — Archiver batch persistence + output parity gate

- **Goal**: batched, NAS-optimized final persistence; lock output parity.
- **Design refs**: §5, §11.
- **Files**: `jarvishep/archiver.py` (batch ≈200, async/move, xlha/slha, delete),
  `jarvishep/hdf5writer.py` (Archiver-fed), `jarvishep/observable_io.py`, `jarvishep/dataconvert.py`,
  NEW `tests/test_archiver_parity.py`.
- **Steps**:
  1. Batch `Archiver.batch_size` (default 200) finished Samples; async write + `move` into
     SAMPLE/ + DATABASE/ (HDF5/CSV); xlha/slha formatting; delete staging
     (`delete_method`).
  2. Use `os.rename`/`shutil.move` (same-volume NAS) by default; `use_shell`/`rm` optional.
- **Accept**: DATABASE/SAMPLE/CSV **byte-identical** to the captured V1 golden output on a
  fixed scan (`--convert` parity); batch boundaries do not drop/duplicate Samples.
- **Rollback**: `mode: thread` / `Archiver.batch_size: 1`.
- **Out of scope**: monitor, checkpoint.

---

### WP-D5.1 — `op_count` writers + TaskFactory background snapshot

- **Goal**: read-only 60 Hz monitor snapshot driven by `op_count` change detection.
- **Design refs**: §6, §7; `V2/factory_design.md` §5, §6, §8.
- **Files**: `jarvishep/worker.py` + `jarvishep/Sampling/sampler.py` (`incr` on state change),
  `jarvishep/factory.py` (`_start_snapshot_updater`, `_collect_latest_status_from_redis`,
  `get_monitor_snapshot`), NEW `tests/test_monitor_snapshot.py`.
- **Steps**:
  1. Writers `incr hep:{worker,calculator,sample,task}:op_count` on each meaningful change.
  2. Background thread (~100–200 Hz): per subsystem, fetch only when `current > last_seen`;
     always refresh `task_queue_length`.
  3. `get_monitor_snapshot()` = in-memory `deepcopy` (<0.5 ms).
- **Accept**: snapshot reflects advancing counters; idle subsystems cost one `GET`/tick
  (assert call counts via `fakeredis`); 60 Hz calls do not perturb throughput.
- **Rollback**: monitor is observational; off ⇒ no behavior change.
- **Out of scope**: the dashboard UI.

### WP-D5.2 — `--monitor` dashboard + run_summary from Redis

- **Goal**: `--monitor` renders the snapshot; run_summary computed from Redis counters.
- **Design refs**: §6; `docs/specs/RUN_SUMMARY_METRICS.md` (frozen contract).
- **Files**: NEW `jarvishep/dashboard.py` (or reuse `jarvishep/monitor.py`), `jarvishep/client.py`
  (`--monitor --pid`), `jarvishep/monitoring/run_summary.py`, NEW `tests/test_dashboard_reader.py`.
- **Steps**:
  1. Dashboard reads `get_monitor_snapshot()` (or attaches to Redis directly — independent
     process, Blueprint §6) at 10 FPS.
  2. `run_summary` fields projected from `hep:sample:stats` + factory timing; **field-for-field
     equal** to the frozen schema.
- **Accept**: reader unit tests green; `run_summary.json` schema-identical pre/post; `--monitor`
  with no running scan exits with a clear message.
- **Rollback**: legacy curses monitor remains until cutover.
- **Out of scope**: Prometheus/Web UI (future).

---

### WP-D6.1 — Heartbeats + dead-Worker respawn + in-flight re-queue

- **Goal**: a crashed Worker cannot stall or corrupt a scan.
- **Design refs**: §6, §10; Blueprint §9; `V2/worker_design.md` §9.
- **Files**: `jarvishep/factory.py` (watchdog), `jarvishep/worker.py` (heartbeat writes),
  `jarvishep/redis_queue.py`, NEW `tests/test_worker_failure.py`.
- **Steps**:
  1. Worker writes `last_heartbeat` to `hep:worker:status:<id>`; Factory watchdog detects
     staleness.
  2. Dead Worker: release its held `calc:free` slots, re-queue its in-flight Sample
     (bounded retries), respawn a fresh Worker.
- **Accept**: SIGKILL a Worker mid-scan → scan completes, no missing/duplicate UUIDs; held
  slots recovered.
- **Rollback**: `mode: thread`.
- **Out of scope**: RNG/checkpoint (D6.2).

### WP-D6.2 — RNG spawning + distributed checkpoint/resume

- **Goal**: reproducibility across Worker counts + resume under the Redis model.
- **Design refs**: §10; `DESIGN_CHECKPOINT_1.7.0.md`; `docs/design/STATE_SAVER_RESUME_DESIGN.md`.
- **Files**: `jarvishep/Sampling/Source/MCMC/runtime_checkpoint.py`, `jarvishep/core.py`
  (`_preload_resume_checkpoint`, drain Redis on resume), `jarvishep/Sampling/sampler.py`,
  NEW `tests/test_distributed_resume.py`.
- **Steps**:
  1. `SeedSequence` master in the sampler checkpoint; per-Sample/per-Worker child streams
     (decouple reproducibility from Worker count).
  2. Resume: rebuild Redis pools from config; **drain** stale `hep:task_queue`; never restore
     in-flight; Sampler re-proposes.
  3. Keep checkpoint UX + payload minimalism; `state.pkl` location/heartbeat unchanged.
- **Accept**: kill-and-resume mid-scan → no duplicate/missing UUIDs; seeded rerun reproduces
  sampler trajectory independent of Worker count; v1 checkpoint refusal message intact.
- **Rollback**: `mode: thread` uses the existing checkpoint path unchanged.
- **Out of scope**: Redis RDB/AOF durability (open question §13.5).

---

### WP-D7.1 — Slow-regime acceptance gates

- **Goal**: lock the architecture against the metrics that matter for real workloads.
- **Design refs**: §1, §12; `docs/specs/RUN_SUMMARY_METRICS.md`.
- **Files**: NEW `tests/test_distributed_acceptance.py`, NEW calculator fixtures under
  `tests/benchmark_project/bin/`, `docs/benchmarks/` report.
- **Gates**:
  1. **Worker scaling** (slow regime): calculator throughput scales ~linearly to physical
     cores, and is demonstrated stable at high Worker counts (toward ~256 on capable
     hardware; scale the assertion to the test machine).
  2. **Archive latency**: NAS `move` archive sustains the Worker output rate; staging never
     unboundedly grows (assert bounded backlog).
  3. **Chaos**: SIGKILL Workers under load → scan completes, parity preserved.
  4. **Parity**: DATABASE/SAMPLE/CSV identical to the captured V1 golden output on a fixed
     calculator scan.
  5. **Monitor**: `get_monitor_snapshot()` sustains 60 Hz with no throughput regression.
- **Accept**: all gates green; numbers recorded in `docs/benchmarks/` and the ledger.
- **Rollback**: n/a (gate only).

---

## 5. Verification Commands (Shared)

```bash
python3 -m pip install -e ".[dev,distributed]"     # V2 env: redis/msgpack/aiofiles + fakeredis
pytest tests/                                       # full suite — fakeredis, no Redis server
Jarvis2 tests/benchmark_project/<task>.yaml --benchmark 30       # throughput
Jarvis2 <calculator-reference>.yaml --check-modules              # slow-regime smoke
Jarvis2 <task>.yaml --convert                       # CSV parity vs V1 golden fixtures
# integration against a real server: start redis (docker compose), set REDIS_URL, then
Jarvis2 <task>.yaml
```

- **Parity oracle**: V2 carries no thread mode. Each WP compares V2 (`Jarvis2`, Redis)
  output to **captured V1 golden outputs** — DATABASE/SAMPLE/CSV produced by frozen `Jarvis`
  1.7.4, stored as fixtures. Several WPs below say "parity vs thread mode" / "vs `mode:
  thread`"; read that as **parity vs the captured V1 golden output**. Regenerate fixtures
  only from frozen V1.
- **Rollback semantics**: V2 has no thread fallback. Where a WP says "Rollback: `mode:
  thread`", substitute **revert the WP commit** (or toggle the WP's feature flag). Rollback
  never means "run V1".
- Benchmark numbers are machine-relative; slow-regime gates scale to the test machine.
- CI must not require a Redis server: use `fakeredis` for unit tests, `skipif REDIS_URL`
  unset for integration tests.

## 6. Deviation & Escalation

- **Drifted line numbers / renamed symbols**: expected; trust the code, note it, continue.
- **A gate is unreachable** after honest work: do not lower it silently — record measured
  numbers + analysis in Notes and `docs/benchmarks/`, mark the WP `blocked`, ask the maintainer.
- **Design conflict with a frozen contract** (§1): the contract wins; if the design violates
  it, that is a design bug — stop and report.
- **Discovered prerequisite**: add a new ledger row (`DX.Ya`), get sign-off before executing.
