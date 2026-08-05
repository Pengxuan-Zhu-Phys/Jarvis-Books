# V2 Distributed Runtime — Development Plan (Agent Execution Playbook)

Last updated: 2026-08-04. Branch `jarvis2`; D17.1–D17.9, D18.1–D18.5, **D18.7–D18.8**, D20.1–D20.8, D21.1–D21.14, D19.2, and **D22.1** are complete. **New: D22 — `Sampling.Mapper`** ([`DESIGN_SAMPLING_MAPPER_2.0.md`](DESIGN_SAMPLING_MAPPER_2.0.md), 2026-08-04): an expression-based, closed-namespace u→x mapper that lets a card sample `t` and derive `x: "cos(t)"`, `y: "sin(t)"` as **named parameters** (not observables, and with **no** second coordinate layer — `u` stays the sole persisted coordinate, per D21). D22.1 (docs↔code alignment) is done; D22.2–D22.8 are open. Two measured defects were registered on the way: **D22.8 (MEDIUM)** the schema rejects `Operas.Modules[].input: ["x"]` although `operas.py:319-321` implements it, and **D22.7 (LOW)** removed-root-key suggestions match on the leaf name so a nested key is told to "remove top-level …". **New: D23 — Jarvis-Operas 命名空间常量** (design lives in the Operas repo: [`agent_maintenance/DESIGN_NAMESPACE_CONSTANTS_CN.md`](../../Jarvis-Operas/agent_maintenance/DESIGN_NAMESPACE_CONSTANTS_CN.md), 2026-08-04): let an expression write `pdg.m_Z` **without parentheses**, as an arity-0 `OperaFunction` carrying `flags={"constant"}` — measured to fold to a literal at parse time (zero per-sample cost, `sqrt(pdg.m_Z**2 + pdg.m_W**2)*x` lambdifies to `121.555092541448*x`) and to coexist with a same-named observable. **All 7 rows are todo and gated on the architecture owner's sign-off** (the doc is the "架构需求单" JO's playbook §12 requires); the release-blocking row is **D23.5** (V1/PLOT regression — V1's `except Exception: pass` would swallow a broken `build_sympy_dicts` signature and silently drop the whole Operas function table). **Breaking change (`f983772`, 2026-08-03): sampler YAML is Bounds-only** — every method knob (`Radius`, `MaxAttempt`, `MaxWorker`, `Point number`, `Seed`, the CSV path, the whole AdaptiveBridson knob set) now lives under `Sampling.Bounds`; the old Sampling-top layout is **rejected** with a diagnostic naming the destination (`JV2-MTH-001` / `JV2-MTH-020`). `Method` / `Variables` / `LogLikelihood` / `selection` / `FeedbackReturn` stay at the top. `Bounds` is closed per method, so a typo inside it is an error. Docs realigned 2026-08-03 ([`YAML_REFERENCE_2.0.md`](YAML_REFERENCE_2.0.md) §2/§3.2/§6, `skills/`, `components/`). Checkpoint cadence is now the single knob `EnvReqs.V2.checkpoint_heartbeat_sec`, and the HDF5 stale-writer-flag repair is **pure h5py** (the `h5clear` CLI dependency is gone). **V1 migration status + open gaps: [`V1_MIGRATION_STATUS_2026-07-31.md`](V1_MIGRATION_STATUS_2026-07-31.md)** (11 V1 samplers unmigrated after nuisance reclassification; `--benchmark` absent). LibDeps builds once in the control preflight and reuses stamps ([`DESIGN_LIBDEPS_2.0.md`](DESIGN_LIBDEPS_2.0.md)). Multimode Calculator is shared-only with dot-qualified logical nodes, physical-pack affinity, bounded affinity waiting, and pre-acquire `selection` filtering ([`DESIGN_CALCULATOR_MODES_2.0.md`](DESIGN_CALCULATOR_MODES_2.0.md)). Crash-safe resume now treats HDF5 UUIDs as the sole completion truth, uses rolling safe restore points, deduplicates at the Archiver, recovers stale process/HDF5 state, and gives busy FileOperation children an owner-watch plus process-group cleanup; fixed `ZP` classification now exposes and cleans all known Jarvis components without a matching live Control ([`DESIGN_RESUME_2.0.md`](DESIGN_RESUME_2.0.md)). Full suite baseline failures remain tracked by D13.15.
Archive: `archive/V2_PLAN_ARCHIVE_2026-07-14.md`.
**Priority note (maintainer):** **D8 Agent Bridge stays parked** (D8.1/8.2/8.4 + agent stop-ack;
re-confirmed 2026-07-16). **D9–D12 closed. D13 closed through D13.9** (YAML validation gate —
[`DESIGN_YAML_VALIDATION_2.0.md`](DESIGN_YAML_VALIDATION_2.0.md); flat feedback D13.8 —
[`DESIGN_FEEDBACK_RETURN_2.0.md`](DESIGN_FEEDBACK_RETURN_2.0.md)).
D13.1–D13.7: diagnostics export, review fixes — see
[`D13_SAMPLERS_REVIEW_2026-07-17.md`](D13_SAMPLERS_REVIEW_2026-07-17.md).
**Post-D13 polish (2026-07-21):**
- **D13.9** early config validation (`task_validation` + `contracts/*`; `Jarvis2 validate`);
- vendored **dynesty 3.1.0** + UUID patches; Method=engine for Dynesty/MultiNest (no `Bounds.dynamic`);
- **AdaptiveBridson** renames AdaptiveLevelSet (legacy name removed);
- **check-modules** UX: CSV-or-N-samples, workers=1, `SAMPLE/test/<uuid>/` flat, no tar pack;
- narrowed Sampling templates under `project_template/bin/sampling/`.
Next phase open: **D14 cluster → D15 reuse/analysis + plot scenes → D16 skills**.
New 2026-07-21: [`DESIGN_PHYSICS_PLOT_SCENES_2.0.md`](DESIGN_PHYSICS_PLOT_SCENES_2.0.md)
(D15.5–7 — auto plot YAML must match real physics analysis; evidence: maintainer's iDM
hand edits) and [`DESIGN_SKILLS_LIBRARY_2.0.md`](DESIGN_SKILLS_LIBRARY_2.0.md) (D16 —
YAML complexity is the #1 adoption barrier; **v1 shipped under `skills/`**, D16.1 done).
**Strict validation (D17, maintainer request 2026-07-31)**: illegal key / wrong value type
must log clearly and exit — [`DESIGN_STRICT_VALIDATION_2.0.md`](DESIGN_STRICT_VALIDATION_2.0.md).
Sequencing is forced: **44 of 65 shipped example cards fail validation today** because the
schema omits legitimate keys, so the vocabulary must be completed (D17.1) before strictness
is flipped (D17.5).
**Schema review 2026-07-31** ([`SCHEMA_REVIEW_2026-07-31.md`](SCHEMA_REVIEW_2026-07-31.md)):
D13.11 confirmed landed and correct (install-control epoch fan-out, feedback warning,
Lua slot release, atomic CSV). New **D13.14** (scoped with maintainer): I/O validation pins the format
list in HEP's manifest instead of following the Portal source — a Portal upgrade both
rejects valid cards and reddens HEP's suite; and `AdaptiveBridson` is the one migrated
method whose sub-block accepts any key silently. MCMC-family schemas stay open until that
family is migrated.
**Code review 2026-07-23** ([`CODE_REVIEW_2026-07-23.md`](CODE_REVIEW_2026-07-23.md)):
one **high-severity** finding — after editing a calculator's source the user silently gets
results from stale code, with no operator control over reinstall — plus three medium fixes
and a `run_adaptive` refactor, filed as **D13.11–D13.13**. Accepted fix for the high one:
[`DESIGN_CALC_INSTALL_CONTROL_2.0.md`](DESIGN_CALC_INSTALL_CONTROL_2.0.md) (operator-facing
`jarvis_install.json` + reinstall epoch; **no** source-tree content hashing).
Next pick: **D14.1**; **D15.5** and
**D16.2** remain parallel-safe.
Do not start Agent JSON verbs until D8 is unparked.
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
   **Archive rule (token hygiene):** when a WP reaches `done`, move its ledger row (with
   full notes) and its §4 WP section into
   [`archive/V2_PLAN_ARCHIVE_2026-07-14.md`](archive/V2_PLAN_ARCHIVE_2026-07-14.md) and add
   its ID to the archived-WP pointer at the top of §3. This file must stay open-work-only —
   never let completed entries accumulate here. One-off reports (acceptance runs, dated
   reviews) superseded by a newer review also go to `archive/` once nothing active links to
   them.
   **Skills rule (D16, binding from D16.4):** a user-facing WP is not `done` until the
   affected skill in `skills/` exists or is updated — same standing as YAML_REFERENCE
   updates. See [`DESIGN_SKILLS_LIBRARY_2.0.md`](DESIGN_SKILLS_LIBRARY_2.0.md) §5.
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
| **D4** | Async Archiver | Layer-2 persistence: default **direct** SAMPLE handoff (staging optional), process Archiver, Redis SAMPLE buckets + tar-after-archive; output parity gate |
| **D5** | Monitoring | `op_count`-driven 60 Hz snapshot + `--monitor` dashboard + run_summary from Redis |
| **D6** | Resume + failure handling | heartbeats, dead-Worker respawn, in-flight re-queue, RNG spawning, distributed checkpoint |
| **D7** | Acceptance | slow-regime gates: worker scaling, archive latency, 256-Worker chaos, parity |
| **D8** | Agent Bridge (machine-readable control surface for Jarvis-Agent) | additive `--json` verbs (`--validate`/`--results`/`--status`/`--version-json`), `run_state.json` lifecycle file, SIGINT/SIGTERM graceful-stop contract; frozen human CLI untouched. Design: [`DESIGN_AGENT_BRIDGE_2.0.md`](DESIGN_AGENT_BRIDGE_2.0.md) |
| **D9** | Architecture Hardening (behavior-preserving refactors) | token resolution single-owner, dead async twins removed, `CalculatorModule`/`TaskFactory`/`Jarvis2Core` de-godded, sampler/IO registries (extension points for MCMC + SLHA), `Sample.info` single source of truth. Current baseline: 236 collected (235 passed, 1 flowchart-export skip) + golden parity; D9.4/D9.6 remain partial. Design + WP details: [`DESIGN_PRINCIPLES_REVIEW_2.0.md`](DESIGN_PRINCIPLES_REVIEW_2.0.md) §4 |
| **D10** | AdaptiveBridson Sampler (first feedback-driven sampler, 2 ≤ d ≤ 5) | opt-in `hep:feedback` result channel (zero cost when off), `AdaptiveBridsonSampler`: low-discrepancy gen-0 → crossing detection on a neighbor graph (exact Delaunay ridges d≤3 / approximate kNN d=4–5) → barrier-synchronized edge refinement → `levelset.json`; registered `stateless=False`; deterministic across worker counts; checkpoint at generation barriers; d≥6 rejected. Design: [`components/adaptive_voronoi_contour.md`](components/adaptive_voronoi_contour.md) |
| **D11** | User Interface & Integration Closure | one-intent CLI (`run/check/monitor/plot/portal/operas`) with compatibility aliases; truthful `RunOutcome` and machine-readable failures; Portal HEP formats exposed and discoverable; Operas validation/runtime parity; scan-driven PLOT + flowchart + D10 overlay. Review: [`USER_INTERFACE_INTEGRATION_REVIEW_2026-07-13.md`](USER_INTERFACE_INTEGRATION_REVIEW_2026-07-13.md) |
| **D12** | Calculator V1 parity + user-experience alignment ("V1 用户无感迁移") | unmodified **Eggbox Calculator card** (historical file `Example_Bridson_process.yaml` = `Calculators` external program, not Operas, not a runtime mode) runs end-to-end with golden parity; V1-style core log rendering; flowchart.{json,png} export; `EnvReqs.V2` grouped settings; `Jarvis2 project …`; Jarvis-Examples-owned official catalog. Review: [`PROTOTYPE_CLOSEOUT_REVIEW_2026-07-14.md`](PROTOTYPE_CLOSEOUT_REVIEW_2026-07-14.md) §4.0 naming |
| **D13** | Feedback-driven samplers: MCMC / Nested / Nuisance (V1 science surface on the V2 runtime) | unmodified V1 `Example_DRAM_Operas.yaml` + `Example_Dynesty_Operas.yaml` run end-to-end with convergence/evidence parity vs captured V1 goldens; `FeedbackSampler` base extracted from ALS; `nuisance_optimize` Worker step implemented; worker-count-independent chains. **Closed 2026-07-20** (D13.1–D13.7). Design: [`DESIGN_SAMPLERS_2.0.md`](DESIGN_SAMPLERS_2.0.md); review: [`D13_SAMPLERS_REVIEW_2026-07-17.md`](D13_SAMPLERS_REVIEW_2026-07-17.md). |
| **D14** | Cluster execution + broker durability | `Jarvis2 worker start --connect` remote pools (template over Redis, `<host>:<n>` ids, no-respawn foreign watchdog); broker `requirepass` + AOF + broker-restart resume; SLURM/HTCondor submitters render Phase-1 pools; shared-FS invariant documented. Design: [`DESIGN_CLUSTER_EXECUTION_2.0.md`](DESIGN_CLUSTER_EXECUTION_2.0.md) |
| **D15** | Result reuse + analysis closure | `--warm-start` point cache (fingerprint-miss-by-default, `cached_from` provenance); `Jarvis2 analyze` corner/best-fit/posterior via JarvisPLOT scenes; Portal File/HepMC/LHE exposure + fixtures. Design: [`DESIGN_RESULTS_ANALYSIS_2.0.md`](DESIGN_RESULTS_ANALYSIS_2.0.md) |
| **D16** | User skills library ("一句话一个技能" — kill the YAML complexity barrier) | v1 shipped 2026-07-21 (9 skills + index under `skills/`, every card `Jarvis2 validate`-verified); in-package shipping + `Jarvis2 skill list/show`; card CI; per-milestone coverage growth. Design: [`DESIGN_SKILLS_LIBRARY_2.0.md`](DESIGN_SKILLS_LIBRARY_2.0.md) |
| **D17** | Strict task-card validation (illegal key / wrong type → clear diagnostic → exit) | closed blocks reject unknown keys and wrong types with a one-pass, "did you mean"-annotated report and exit 2 before any side effect; delegated blocks (Portal formats, dynesty pass-through, Operas kwargs) never rejected; un-migrated methods explicitly `open`; **all 65 shipped example cards + 23 template/parity cards green under strict before the flip**. Design: [`DESIGN_STRICT_VALIDATION_2.0.md`](DESIGN_STRICT_VALIDATION_2.0.md) |
| **D18** | LibDeps: install-once shared dependencies (V1 parity) | shared libraries build **once** in the control-process preflight, reuse by fingerprint, rebuild on an operator flag; `${LibDeps:path}` / `${LibDeps:make_paraller}` / `@{ROOT path}` resolve; `--skip-library-installation`; `LibDeps` declared in the schema (also unblocks D17.9). Design: [`DESIGN_LIBDEPS_2.0.md`](DESIGN_LIBDEPS_2.0.md) |
| **D19** | Archiver 刷盘与运行结果诚实性 | 小规模扫描不再误报失败；刷盘判据符合 flush-interval 的通常语义；stall 文案指向真实原因 |
| **D20** | 多模式 Calculator | **V2-only 新功能，不回移 V1**（维护者定调 2026-08-01：V1 已设计定型，非 bug 不加新功能）。一个软件包在一次扫描内以 N 种用途运行（各自的 initialization / execution，必要时各自构建）；`modes` 展开为兄弟模块，运行时零新增概念。设计: [`DESIGN_CALCULATOR_MODES_2.0.md`](DESIGN_CALCULATOR_MODES_2.0.md) |
| **D21** | 断点续跑（Checkpoint / Resume 重做） | **完成并归档（2026-08-02）**。HDF5 UUID 是唯一完成事实；generator checkpoint 只保存已落盘边界之前的滚动安全恢复点；Archiver 写入端去重；SIGINT/SIGKILL、无 checkpoint DATABASE 对账、Adaptive 世代恢复与 stale 进程/HDF5 清理均通过端到端验收。D21.9 封闭 FileOperation 孤儿路径；D21.10 为所有没有对应 live Control 的 Jarvis 进程加入固定 `ZP` 检查/一键清理分类。设计与证据: [`DESIGN_RESUME_2.0.md`](DESIGN_RESUME_2.0.md) |

---

## 3. Progress Ledger

Allowed statuses: `todo`, `in-progress`, `done`, `blocked`.

> **Completed WPs are archived** with their full notes in
> [`archive/V2_PLAN_ARCHIVE_2026-07-14.md`](archive/V2_PLAN_ARCHIVE_2026-07-14.md)
> — D0.1–D0.5, D1.1–D1.2, D2.1–D2.3, D3.1–D3.3, D4.1–D4.2, D5.1–D5.2, D6.1–D6.2,
> D7.1 (2026-06-28/29); D9.1–D9.3, D9.5, D9.7–D9.8 (2026-07-10); D10.1–D10.2
> (2026-07-11); D11.4a–D11.4d (2026-07-13); D9.4/D9.6, D10.3–D10.5, D11.1–D11.5,
> D12.0–D12.8 (2026-07-14…16); D13.1–D13.5b (2026-07-16/17); D17.1–D17.9 (2026-07-31/08-01); D18.1–D18.5 (2026-08-01); D20.1–D20.8 and D21.1–D21.10 (2026-08-02); D13.6–D13.10
> and D13.11 (2026-07-29)
> (2026-07-20/21); plus the pre-V2 M0/M1
> row. This table
> keeps only open work: todo / in-progress / partial / blocked.

| WP   | Title                                                                                  | Milestone | Depends on       | Status                                 | Date       | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ---- | -------------------------------------------------------------------------------------- | --------- | ---------------- | -------------------------------------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| D8.1 | Agent API facade: JSON envelope + `--validate`/`--results` + `--version-json`        | D8        | —                | **deferred**                           | 2026-07-14 | **Parked (maintainer):** Agent bridge not in current priority. Design remains `DESIGN_AGENT_BRIDGE_2.0.md` §4. |
| D8.2 | `run_state.json` writer + `--status --json` / `--monitor-json`                        | D8        | D8.1             | **deferred**                           | 2026-07-14 | **Parked** with D8.1. |
| D8.3 | Control-process graceful stop (SIGINT/SIGTERM → checkpoint → clean exit)              | D8        | —                | **partial** (human path enough)        | 2026-07-16 | Interactive stop (`64d7486`) + **interrupt checkpoint** (`_save_interrupt_checkpoint` before teardown; tests `test_graceful_stop.py`). Agent pieces (run_state interrupted, stop-ack) + D8.2 still **parked**. |
| D8.4 | Strict-validate diagnostics (silent-coercion warnings, dead keys, unknown keys)       | D8        | D8.1             | **deferred**                           | 2026-07-14 | **Parked** with D8.1. |

| D13.14 | Task-card schema: I/O follows Portal (not the pinned manifest) + AdaptiveBridson sub-block schema | D13 | — | todo |            | [`SCHEMA_REVIEW_2026-07-31.md`](SCHEMA_REVIEW_2026-07-31.md) (scoped after maintainer feedback: MCMC family is **not migrated**, so leave its schemas open; I/O authority is **Portal source**, not HEP's registry). **(1) HIGH — I/O validation pins the format list.** `_validate_selected_io` never consults Portal; a name absent from `manifest["io"]` is a hard `JV2-SCH-002`. Simulated a Portal upgrade adding `HepMC`: the card is rejected ("register a local schema in schema/manifest.json") even though Portal can read it — the exact coupling the plugin design exists to avoid — and `test_manifest_matches_builtin_portal_formats` asserts **set equality**, so the same upgrade turns HEP's suite red with no HEP change. Invert: accept when `available_io_formats(direction)` supports it (manifest schema only *enriches*); no schema → accept with at most an info note; not Portal-supported → hard error listing Portal's actual formats. Relax the test to a **subset** assertion. **(2) ~~HIGH — `AdaptiveBridson` is the one migrated method with zero coverage~~ — RESOLVED by `f983772`** (实测 2026-08-03：`Bounds.initial_radiuz` 现在报 `JV2-SCH-001 $.Sampling.Bounds`；`common.json#/$defs/adaptiveBridson` 已是 `additionalProperties: false` 的封闭块)。原文如下：`Sampling.AdaptiveBridson` is typed as a bare object: `{initial_radiuz: 0.08, 瞎写: 1}` validates identically to the correct card, and the sampler silently uses defaults. Keys are already specified in YAML_REFERENCE §6.9 — transcription, not design. (Bridson/Random/CSV/Grid are covered by schema; Dynesty/MultiNest by the legacy contracts layer.) **(3)** A **registered** method with no manifest entry fails open — simulated: garbage `Bounds` → zero diagnostics. Add the missing Distributor↔manifest test (Portal↔manifest has one) plus a diagnostic. **(4)** `test_manifest_gives_each_sampling_method_its_own_schema` asserts only URI uniqueness. **(5)** root `$id` ≠ filename (cosmetic). **Out of scope**: MCMC-family `Bounds` schemas — revisit when that family is actually migrated. **Open question for maintainer**: the un-migrated MCMC family is registered and passes validation today, so users can launch it; consider marking it experimental at the registry level. |
| D13.15 | Triage the 8 pre-existing test failures (fix or quarantine with a reason) | D13 | — | todo |            | The plan header has recorded "608 passed with 8 pre-existing failures" since 2026-07-29. A standing set of known-failing tests erodes the suite's value as a gate: the next agent cannot tell its own breakage from the background, and Protocol §5 ("a WP is not done with failing tests") becomes unenforceable. Triage each: fix it, or mark it `skip`/`xfail` **with the reason and a tracking note** so green means green. Record the outcome here. |
| D13.12 | Decompose `AdaptiveBridson.run_adaptive` (959-line method, 9 exit points) | D13 | D13.11 | todo |            | `CODE_REVIEW_2026-07-23.md` §3. `adaptive_bridson.py` is 4 068 lines; the main loop inlines gen-0 bootstrap, fill passes, root correction, gap bridging, and advance/converge into one scope with nine inline `return`s. Extract each seam behind an explicit `LoopDecision` (continue / advance / stop-with-reason) so exit conditions become individually testable. Behavior-preserving; existing `test_adaptive_bridson.py` is the gate. Do before the next feature lands in that file. |
| D13.13 | Doc/design debt: `convert` in `components/cli.md` | D13 | — | todo |            | `CODE_REVIEW_2026-07-23.md` §2.2. `Jarvis2 convert` is missing from `cli.md` (all 10 other subcommands are there). ~~Install-reuse design doc~~ — **done 2026-07-23**: [`DESIGN_CALC_INSTALL_CONTROL_2.0.md`](DESIGN_CALC_INSTALL_CONTROL_2.0.md). YAML_REFERENCE §9.4's false idempotency note corrected 2026-07-23; its final rewrite ships with D13.11. |



| D18.6 | `Utils.interpolations_1D` → Jarvis-Operas 迁移指引（不是补功能） | D18 | — | todo |            | [`V1_MIGRATION_STATUS_2026-07-31.md`](V1_MIGRATION_STATUS_2026-07-31.md) §6.1. **15/65 张真实卡片**用了这个块；V2 目前只回一句 `"Remove top-level Utils: it is not supported by V2."`（`task_schema.py:28`）。但**能力其实已经在 Jarvis-Operas 里**：`interp1.interp1_{xy_flat,xlog_yflat,xflat_ylog,xy_log}` 正好覆盖 V1 的 logX/logY 四种组合，`dmddxe.*` 已内置整套直接探测限制（CDEX/CDMS/DAMIC/DarkSide50/AllLimits*），自有曲线可用 `add_interpolation_namespace` 注册。所以这是**迁移路径缺失而非功能缺失**——用户被告知删掉却不知道去哪找。改成指路消息 + YAML_REFERENCE 对照写法 + 一篇 skill（D16 规则）。V1 语义参考：`core.py:343-353` 把插值注册进 `_funcs`，与 Operas 函数同一个表达式命名空间，故 `XenonSD2019(mchi)` 可直接写在 LogLikelihood/selection 里。 |
| D18.7 | 补齐 5 种变量分布（Binomial/Poisson/Beta/Exponential/Gamma） | D18 | — | **done** | 2026-08-02 | V1 10 种变量分布现全部可用。补齐 schema、严格参数契约与 Worker 的确定性 inverse-CDF 映射：Binomial/Poisson 输出整数；Beta/Exponential/Gamma 使用 scipy inverse CDF。`num`/`length` 明确为方法的 u-space 参数，亦可用于这五种类型。新增卡片验证、参数域拒绝和 mapper 测试；同步 YAML_REFERENCE。 |
| D19.1 | **HIGH — 小规模扫描全部误报失败**：Archiver 部分批次永不刷盘 | D19 | — | **done** | 2026-08-02 | **已修并实测确认（2026-08-02 复核）**：`archiver.py:158` 现为 `bool(self._batch) and (len(batch) >= batch_size **or** 间隔已到)`。端到端实跑：12 样本扫描 → `RunOutcome status=success archived=12 exit=0`；20000 样本扫描 → `archived=20000 exit=0`。以下为原始记录。发现于 2026-08-01 的 LibDeps 实跑验收。`archiver.py:157` `_flush_due_locked` 用的是 **AND**：`len(batch) >= batch_size` **and** 间隔已到——所以**不满一批的数据永远不会按间隔刷盘**，`flush_interval_sec` 实际只对满批起节流作用，起不到延迟上界的作用。后果：样本数少于 `batch_size`（默认 **200**）的扫描在运行期一行都不写，控制进程的 drain 等待循环看到 `written==0`、`archive_q==0`、workers 已完成 → 5 秒后判定 stall → **`Run failed` + 退出码 1**，而数据其实在 teardown 的 `stop(drain=True)` 强制刷盘时全部落盘。实测：两次 21/23 样本的扫描均报 *only 0/23 DATABASE rows written* 并退出 1，而 `samples.hdf5`/`samples.csv` 里 **44 行数据完好**。**每个小于 200 样本的扫描都会中招**（含大量冒烟/测试跑）。雪上加霜：报错文案 *Likely early SAMPLE bucket pack pruned dirs...* 指向的是另一个（已修的）bug，会把排查者引到错误方向。修法：刷盘判据改为 **OR**（满批 **或** 间隔到），这才是 flush interval 的通常语义；并修正 stall 文案。 |
| D18.8 | `Jarvis2 --refs`：打印内置采样器的参考文献 | D18 | — | **done** | 2026-08-02 | `Jarvis2 --refs` 输出 V2 logo、框架、Bridson/AdaptiveBridson、MultiNest、Nested sampling、Dynesty 与 MCMC family 的静态参考文献；Random/Grid/CSV 明确标为无外部 sampler citation 的原生策略。命令无需 task YAML、Redis 或 Worker；CLI parser/output 测试覆盖。 |
| D18.9 | 核实 nuisance 局部优化的维度覆盖 | D18 | — | todo |            | 维护者澄清：nuisance sampler 由 **Worker 对某个参数点做局部优化**——与 D13.4 已交付的 `nuisance_optimize` 定位一致，故它**不是**未迁移采样器（未迁移数由 12 修正为 11）。待核实：V2 的 `Profile1D` 只做**一维**黄金分割；若 V1 支持多个 nuisance 参数**联合**优化，则属部分迁移，需补多维优化。 |
| **D21.9** | **HIGH — checkpoint 拖垮运行时吞吐**：改为心跳快照 + 索引水位 | D21 | — | **done** | 2026-08-02 | **复审实测确认**：同卡同机 20000 点吞吐 **144.40 samples/s**（改动前基线 145.27，差 0.6%，门槛 5%）；控制进程 RSS **64 MB**（原 1.6–4.2 GB 锯齿）。`_checkpoint_snapshots` 为 `deque(maxlen=8)`，有界。 以下为原始工作包描述。 设计 [`DESIGN_RESUME_2.0.md`](DESIGN_RESUME_2.0.md) §13–§15。**维护者定调（2026-08-02）：checkpoint 一定不能影响运行时性能，吞吐与并发不受影响；resume 时可以多跑一些点，一个心跳周期的重算不浪费多少算力。** 实测第一版违反此约束：同卡同机 20000 点基线 **145 samples/s**，第一版在 N≈25k 处降到 **12 samples/s** 且随 N 继续劣化；控制进程 RSS **1.6–4.2 GB 锯齿**（2 变量玩具扫描）；checkpoint 文件 20k 样本时 **2.26 MB** 且线性增长。病灶 `fixed_set_sampler.py:80` 每批 `deepcopy(export_runtime_state())`，而该 export 每次物化三个 O(N) 容器（`u_by_uuid` 全量 `tolist()`、`uuid_by_accepted_index`、`submitted_uuids`），`batch_size: 2` ⇒ 每 2 个样本一次 O(N) 拷贝 ⇒ **O(N²)**；`_pending_restore_batches` 只在心跳排空，峰值驻留约千份 O(N) 快照。**这条路径对所有运行生效，不只 resume**。改法：(a) 心跳线程只**置一个布尔标志**，由 sampler 在下一个批边界做 **O(1)** 捕获（热路径固定开销 = 每批一次布尔判断，无锁竞争）；(b) 维护**连续落盘索引水位 `P`**（`[0,P)` 全部已落盘，乱序到达暂存在以在途窗口为界的小集合）；(c) 保留最近 K 个心跳快照，**恢复点取最新的满足 `index_j ≤ P` 者**。正确性：`index_j ≤ P` 保证小于它的索引全在盘上，重放覆盖 `≥ index_j`，**没有索引会被遗漏**。注意**裸的心跳快照不安全**——心跳时刻 generator 已越过在途点，直接用会让它们永不被重新提出（S2 被打破、点静默丢失），水位条件正是堵这个洞的。 |
| **D21.10** | checkpoint 载荷瘦身：只存 generator 状态 | D21 | D21.9 | **done** | 2026-08-02 | **复审实测确认**：20k 样本处 checkpoint 文件 **4598 字节**（原 2.26 MB 且线性增长）；导出键只剩 generator 状态（`numpy_random_state`/`index`/`accepted_index`/`ready_queue`/`seed`/`selectionexp`/`checkpoint_index`），`u_by_uuid`/`uuid_by_accepted_index`/`submitted_uuids`/`completed_uuids` 均已消失。 以下为原始工作包描述。 设计 §16。快照只保留 `(rng_state \| Generator, index, accepted_index, ready_queue)`——**~3 KB 定长**，对比现在 20k 样本时的 2.26 MB。删除 `u_by_uuid`、`uuid_by_accepted_index`、`submitted_uuids`、`repropose_unfinished()`：它们存在的唯一理由是"按 uuid 把在途点重投一遍"，改为"恢复 + 重放"后全部多余（`uuid = sha256(prefix:seed:index)` 是纯函数可重算；重放用同一 RNG 状态重新产出同样的 u_coords；完成事实在 DATABASE）。**用 pickle 不用 dill**——payload 全是 dict/int/ndarray/RNG 元组，原生可 pickle，`np.random.Generator` 本身也可 pickle；dill 的价值在 lambda/闭包/动态类，这里一个都没有。**不要直接 pickle 活的 sampler 对象**（挂着 loguru handle、redis socket、config，拖不动；写 `__getstate__` 剔除等于现在的显式 export）。保留显式 export，但补一条防漏测试：**断言 sampler 每个属性都被显式归类为"保存"或"排除"**，新增字段未归类即失败——既不会"保险起见全存"退回 O(N²)，也不会静默漏状态。**明确不做**：用 factory 的 pickle 记录"哪些点落盘了"——那会重新引入第二个真相来源，崩溃点必然落在"写了盘"与"更新簿记"之间、两者一定不一致（`completed_uuids`/`acked_uuids` 就是这么错的）；且核对过 factory 无状态可存（bucket 编号由 `init_sample_buckets()` 扫目录得出，池由配置重建，记录本身带 `bucket_dir`/`bucket_id`）。 |
| D21.11 | 落盘确认增量化 + 心跳间隔可配置 | D21 | D21.9 | **done** | 2026-08-02 | **复审实测确认**：记录已带 `sample_index`（选了设计 §18 的方案 (b)，更彻底）；`ArchiverProcess.persistence_state()` 注明 *"Read the durable child prefix in O(1), never SMEMBERS"*；`ArchiveProcessor` 用 `persisted_index_prefix` + 有界乱序集合，`acked_uuids` 降级为旧 DATABASE 的兼容回退。 以下为原始工作包描述。 设计 §18。控制进程现在每次心跳用 `SMEMBERS` 读整个 `hep:archived:<scan>`——**每 30 s 一次 O(N)**，心跳自己就是热路径污染。要求改为 **O(新增)**：(a) Archiver 在 `SADD` 之外再 `RPUSH` 增量，控制进程每心跳只 drain 新增；或 **(b) 记录里加 `sample_index`**，Archiver 直接发布单调连续索引水位，控制进程连映射表都不用维护。**(b) 更彻底**——让水位、写入端去重、resume 的索引对账三件事共用同一字段，代价是每条记录多一个整数；采用后写入端去重可从"全部已落盘 UUID 的内存集合"（10⁶ 样本约百 MB）退化为**索引区间判断**，内存降到常数。另：`CHECKPOINT_HEARTBEAT_SEC` 目前写死 30，应改为可配置——重算上界就是一个心跳周期的墙钟工作量，贵样本的用户需要能调小。 |
| **D21.12** | 性能门禁进回归 + 不丢点负面测试 | D21 | D21.9–D21.11 | **done** | 2026-08-02 | **复审实测确认**：未中断的 20000 点基线 vs 中途 Ctrl+C 续跑，**逐 uuid 集合完全一致**（20000 行 / 20000 unique / 重复 0 / missing 0 / extra 0）；SIGKILL + 手动删除 checkpoint 后仍靠 DATABASE 续跑成功（6000/6000/0），孤儿舰队自动清理、租约自动接管；续跑的物理点重算量为 0（`advance_to_persisted_prefix` 本地快进）。全量测试 780 passed / 6 failed，6 个全属 D13.15 既有抖动（4 个 legacy `--plot` parser、1 个 Redis 端口冲突 fixture、1 个 scaling gate），无本轮新增失败。 以下为原始工作包描述。 设计 §19。**性能是门禁不是附注**：P1 同一张 20000 点卡片吞吐回到 **≥ 基线 145 samples/s 的 90%**，与关闭 checkpoint 的对照跑差异 < 5%；P2 控制进程 RSS **平坦**（不随 N 增长）且全程 < 500 MB；P3 checkpoint 文件大小不随 N 增长（20k 与 200k 处相差 < 2×）；P4 每批热路径新增开销 = 一次布尔判断、无锁竞争；P5 单次心跳耗时不随 N 增长。行为断言沿用并加强：A1′ 中断+resume 后 DB `rows == unique`；A3′ 重算量 ≤ **一个心跳周期**的提交量（不再是 in-flight 上界）并记录实测值；A5′ 删除 checkpoint 后仍能靠 DATABASE 续跑；**A8 新增负面测试**——人为让某个索引晚落盘，确认恢复点不会跨过它（验 `index_j ≤ P` 水位条件，这是"不丢点"的唯一防线）。 |
| **D21.13** | **MEDIUM — 续跑把失败样本报成成功（建议 2.0.0 发版前修）** | D21 | — | **done** | 2026-08-02 | **已修**：`read_persisted_outcome_counts` 按 `status` 统计 completed/failed；`prepare_resume(..., persisted_failed=)` 写入 `SAMPLE_STATS`；core `_load_persisted_database_state` 在 bootstrap/resume 路径统一加载。验收单测：`test_prepare_resume_seeds_failed_counts_from_database`、`test_read_persisted_outcome_counts_split_failed_and_completed`。 以下为原始工作包描述。 发现于 2026-08-02 的发版前复审。**同一张卡、同一份数据，两次调用给出两个结论**：首跑 `status=partial_failure completed=20 failed=20 exit=1`；`--resume` 后 `status=success completed=40 failed=0 exit=0`，而 DATABASE 一行未变（40 行 = 20 `Failed` + 20 `Completed`）。**数据是对的**（D21.6 正常，失败点不会被无限重算），错的是**汇报**。根因：`persisted_count` 不看 `status`；`reconcile_resume_ephemeral(completed=…, failed=0)` 把失败点全部计入 completed。 |
| **D19.2** | **MEDIUM — 只要有一个样本失败就静默不出图**（既有问题，非 D21 引入） | D19 | — | **done** | 2026-08-02 | **已修（采用「partial_failure 也出图」）**：`success` 与 `partial_failure` 均走 nested CSV + plot scenes；`partial_failure` 打 WARNING 标注 completed/failed 与失败比例；全失败 / interrupt 跳过并 INFO 说明原因。单测：`test_partial_failure_still_emits_plot_scenes`、`test_full_failure_skips_plot_scenes_with_reason`。 以下为原始描述。 旧逻辑 `if outcome.ok` 挡住出图；物理扫描部分失败是常态。 |
| D21.14 | 复审 LOW 项打磨 | D21 | — | **done** | 2026-08-02 | **四条均已落地**：(1) Bridson/FixedSet `advance_to_persisted_prefix` 置 `_suppress_submit_progress` 并清空 barinfo；(2) thread `ArchiveProcessor.persistence_state` 改为 O(1) highwater（去掉 `sorted(acked_uuids)`）；(3) `JV2-VAR-031/032` 在 `diagnostic_guidance` 先于 Variables 前缀匹配，suggestion 点名缺失参数；(4) skill `resume-and-recover.md` + `h5clear` RuntimeError 写明 HDF5 CLI 依赖（非 h5py）。**发布说明待写**：DATABASE/CSV 新增 `sample_index` 与 `status` 两列。 |

| **D22.1** | **文档修正：顶层键表 / 骨架 / §7 / §11 与代码对齐** | D22 | — | **done** | 2026-08-04 | 实测发现 `YAML_REFERENCE_2.0.md` §4 记录的 12 个顶层键里**有 6 个被 schema 拒绝**（`project_name`、`scan_name`、`task_result_dir`、`run_id`、`Mapper`、`Likelihood`；实际只接受 `Calculators/EnvReqs/LibDeps/Operas/Sampling/Scan`），且 §2 可复制骨架第一行就是被拒的 `project_name:`，§7 用一整张 type 表记录了一个不存在的顶层 `Mapper` 块，§11 整节文档化了被拒的顶层 `Likelihood`。**已改**：§2 骨架（identity / mapper / likelihood 三处）、§3.2/§3.3 Key Index、§4 拆成"接受 6 个 + 拒绝 7 个（含 `Utils`）"两张表、§6.6 CSV 措辞、§7 重写为"无 YAML 键 + 自动推导 + 双实现现状 + D22 前瞻"、§10 Operas input、§11 改为 `Sampling.LogLikelihood`。骨架不再报 Mapper/project_name/Likelihood 错。 |
| D22.2 | `MapperPipeline` + `MapperSpec`：收编控制侧与 Worker 侧两套 u→x 实现（**无新 YAML**，纯重构） | D22 | D22.1 | **done** | 2026-08-04 | [`DESIGN_SAMPLING_MAPPER_2.0.md`](DESIGN_SAMPLING_MAPPER_2.0.md) §6。现状实测：`mapper.py:35 DistributionUMapper.map`（Worker 侧，写 params）与 `Sampling/sampling_utils.py:59 map_u_to_physical`（控制侧，喂 `selection`，11 个调用点）是两套独立实现，今天 bit 相同。Bridson 的 row 路径可等价换成 `pipeline.map(row_to_u_coords(row))`——实测 2000 组随机 row 跨 Flat/Log/Normal `max|diff| = 0.0`。同时把 `mapper.py:37` 的 `<` 收紧为 `!=`（现状多余 u 被静默丢弃，实测 `\|u\|=3` 喂 2 变量返回 2 key）。Accept: 不变量 M2/M5/M6；65 张示例卡 params 输出逐位不变。 |
| D22.3 | `Sampling.Mapper` 扁平 map schema + `JV2-MAP-0xx` 装载期诊断 | D22 | D22.2 | **done** | 2026-08-04 | 设计 §4/§5。**YAML：`Sampling.Mapper` 直接是 `{名字: 表达式}`，无嵌套 `derive`。** `Sampling` closed zone 一键增量。封闭性：`free_symbols ⊆ Variables ∪ Mapper 名 ∪ 常量`。DAG 拓扑 + 环检测、命名冲突、CSV 拒绝、`JV2-MAP-050/051` 先验告警。 |
| D22.4 | 求值接线：`selection` 符号集扩展 / `bind_params` 播种 / DATABASE 列 / 绘图轴优先级 | D22 | D22.3 | **done** | 2026-08-04 | 设计 §4.6/§4.7。`plot_scene` 轴优先级：levelset → **`Mapper` 书写序** → `Variables` 序 → 归档列。 |
| D22.5 | checkpoint 卡片指纹纳入 Mapper 文本哈希；`--resume` 变更拒绝 | D22 | D22.3 | **done** | 2026-08-04 | 设计 §8。改 `Sampling.Mapper`（或 Variables）后 `--resume` 硬拒绝（`mapper_hash`）。 |
| D22.6 | 参数方程示例卡 + skill + 参考文档 §6.14 | D22 | D22.4 | todo | | 验收 P4/P7：200 点跑出 `t,x,y` 三列，`x²+y²=1` 在 1e-12 内，`selection: "y > 0"` 只留半圆，自动出图轴是 `x,y`。 |
| D22.7 | LOW：`_root_key_suggestion` 改为按路径匹配 | D22 | — | **done** | 2026-08-04 | 嵌套 `$.Sampling.Mapper` 不再被误报为 top-level Mapper。 |
| **D22.8** | **MEDIUM — schema 比 runtime 严：Operas `input: ["x"]` 被拒但运行时支持** | D22 | — | todo | | 实测：`operas.py:319-321` 显式处理裸字符串（`payload[item] = observables.get(item)`），但 schema 要求 `input[]` 每项是 object，`Jarvis2 validate` 报 `JV2-SCH-001 $.Operas.Modules[0].input[0] 'x' is not of type 'object'`。**这是 D17 严格校验引入的能力回退**：一份能跑的卡片过不了校验。修 schema（允许 string \| object），不是改 runtime。已在 `YAML_REFERENCE §10` 标注临时写法 `- {name: x, entry: x}`。**第二处疑似同类（待验证，勿照抄结论）**：`YAML_REFERENCE §2` 骨架第 257 行记录 Calculator dump `variables[] - name: yy`（无 `expression` = 原样拷贝该 param），schema 报 `'expression' is a required property`；HEP 侧 `flowchart.py:199` 显式处理 `"expression" not in spec` 这一形状，**但真正执行 dump 的是 Jarvis-Portal（外部包），本轮未实测其行为**。做这个 WP 时先确认 Portal 是否真支持无 expression 拷贝，再决定是放宽 schema 还是改文档。 |

### D23 — Jarvis-Operas 命名空间常量（`pdg.m_Z` 无括号形式）

设计文档在 **Jarvis-Operas 仓**：[`agent_maintenance/DESIGN_NAMESPACE_CONSTANTS_CN.md`](../../Jarvis-Operas/agent_maintenance/DESIGN_NAMESPACE_CONSTANTS_CN.md)。
该文档同时是 JO playbook §12 要求的**架构需求单**，**需架构负责人签字后才可开工**。
**执行顺序不可颠倒**：D23.1 → D23.2 → D23.3 →（A5/A6 回归）→ D23.4/D23.6。
先改 V2 正则只会把 `AttributeError: 'Symbol' object has no attribute 'm_Z'` 换成一个更晚才炸的报错。

| ID | 工作包 | 主题 | 依赖 | 状态 | 完成日 | 备注 |
|----|--------|------|------|------|--------|------|
| **D23.1** | **JO：`build_constant_dicts` + `build_sympy_dicts(constants=)`**（仓库：Jarvis-Operas） | D23 | — | todo | | 设计 §6。约 40 行，**唯一的架构级改动**。常量 = arity-0 `OperaFunction` + `flags={"constant"}` + `metadata["value"]`，**`core/spec.py` 与 `core/registry.py` 一行不改**（实测 flags/metadata 已能穿透注册表；`registry.call('pdg.m_Z')` → 91.1876；同名重复注册被 `OperatorConflict` 挡住）。硬契约 C-1/C-2：**返回值必须保持二元组、`mapping` 必须保持第一位置参数、新参数必须关键字且有默认值**——V1 `jarvishep/inner_func.py:119-127`/`:160-168`、PLOT `jarvisplot/inner_func.py:178-186`、HEP2 `operas_functions.py:137` 四处都按 `func_locals, numeric_funcs = build_sympy_dicts(mapping, include_all=True)` 解包，且 **V1 那两处外面包着 `except Exception: pass`——破了不报错，只静默丢掉整个 Operas 函数表**。合并规则：常量优先、跳过同名 arity-0 impl（这是正常路径不是错误路径，`build_register_dicts` 必然含它），**不要 raise**。`_refresh_sympy_dicts`（`integration.py:80-88`）同步。 |
| D23.2 | JO：`pdg` namespace 首批常量 + `constant_decl` helper（仓库：Jarvis-Operas） | D23 | D23.1 | todo | | 设计 §7。`namespaces/_constants.py` 的 `constant_decl` **由 `value` 生成 `numpy_impl`**，杜绝值漂移。首批只放高频无歧义量（`m_Z/m_W/m_t/m_h/m_e/m_mu/m_tau/alpha_em/alpha_s_mZ/G_F/hbarc/c`），**不要**塞混合角、CKM 元素这类依赖参数化约定的量。metadata `source`（PDG 版本）**必填**。`catalog/index.py:13-19` 注册 + `namespace_policy.py:10-21` 把 `pdg` 加进保护名单（常量表是物理事实，不能被用户 op 覆盖）。**单位只做文档不做换算**；用 `sp.Float` 不用 `sp.Rational`（实测 `Rational(1,3)*x` 在 `x=3` 处返回恰好 1.0，语义不同）。 |
| **D23.3** | **HEP2：放宽发现闸门 + 接线常量**（`jarvishep2/operas_functions.py`） | D23 | D23.1 | **done** | 2026-08-04 | 设计 §8。`:39` 的 `_QUALIFIED_FUNCTION_CALL` 要求限定名后必须跟 `(`，所以 `pdg.m_Z` 根本不触发发现。**不要只删尾部 `\s*\(`**——实测裸放宽会误命中 `/usr/local/bin/foo.sh`、`./run.sh`、`results/out.dat`、`C:\tools\a.exe`；用带守卫的 `(?<![/\\.\w])[A-Za-z_]\w*(?:\.[A-Za-z_]\w*)+(?![\w.]*[/\\])`，实测这四条全挡住且真阳性一条不丢。**误判代价要报实数**：一次 Operas 冷启发现实测 **0.86–1.19 s/进程**，Worker 是 spawn 的，N 个 Worker 就是 N 次。`:136-137` 改传 `constants=build_constant_dicts(target_registry)`。**改名**：`_QUALIFIED_FUNCTION_CALL` / `expression_uses_operas_function` / `operas_expression_functions_required` 全串改成 "reference" 口径（常量不是 function call）；`expression_uses_operas_function` 是 `__all__` 导出（`:184`），留别名或同步全部调用点。 |
| D23.4 | HEP2：`Operas.Modules[].operator` 指向常量的校验诊断 + `YAML_REFERENCE` 表达式语法补"限定常量" | D23 | D23.3 | **done** | 2026-08-04 | 设计 §8.4/§11.5。`operator: pdg.m_Z` + `call_mode: call` 运行时能跑通并返回 91.1876，但语义无意义；给一条诊断指向"直接在 `expression` 里写 `pdg.m_Z`"。 |
| **D23.5** | **V1 / PLOT 回归门禁**（跨仓） | D23 | D23.1 | todo | | 设计 §10 A5/A6，**本 WP 是发布闸门**。冻结 V1 `jarvishep/inner_func.py` 不动，比对 `build_expression_context()` 返回的 `parse_locals`/`numeric_modules` 键集合与改动前**逐键相同**；并**临时把 V1 的 `except Exception: pass` 改成 `raise` 跑一遍**，确认没有被触发。PLOT `inner_func.py:178-186` 同法。V1 定位说明（待维护者确认）：本设计下 V1 会多出一批**可调用的零参函数**，这与"给 Operas 新增 namespace，V1 自动看见"是同一类事，**不是 V1 功能变更**；若要连这个也挡，需让 `build_register_dicts` 跳过 `constant` flag，代价是 `jopera call pdg.m_Z` 一并失效——**建议不挡**。 |
| D23.6 | JO 文档：playbook §6.5「新增常量」+ `docs/architecture.md` 公共面 + 目录生成器分开渲染常量 | D23 | D23.2 | todo | | 设计 §11。**实现完成前不要先写**（否则又是一份记录未实现行为的文档，重蹈 D22.1 覆辙）。playbook 需补一条：常量的 `defs/` 可为空——这是对 §4 目录规范的唯一偏离。 |
| D23.7 | LOW（可选）：`ConstantValue` / `ConstantNamespace` 可读报错 | D23 | D23.1 | todo | | 设计 §7.4。现状报错实测很差：`pdg.m_Z()` → `TypeError: 'Float' object is not callable`；`pdg.m_H` → `AttributeError: 'types.SimpleNamespace' object has no attribute 'm_H'`。两个小类即可修好，**原型已实测**（代数折叠不受影响、可 pickle、参与运算后退化为普通 `sp.Float`）：`pdg.mZ` → "Did you mean m_Z? Known: hbarc, m_W, m_Z"。采用时 `integration.py:74-75` 只给**含常量的 namespace** 换类型，纯函数 namespace 保持 `SimpleNamespace`。 |

| D14.1 | Worker template over Redis + `Jarvis2 worker start --connect`                           | D14       | —                | todo                                   |            | `DESIGN_CLUSTER_EXECUTION_2.0.md` §3/§4. Template `hep:worker:template:<scan>` (JSON, version-guarded); two pools on one host drain a live scan with DATABASE parity. |
| D14.2 | Remote-aware watchdog + `<host>:<n>` worker ids                                         | D14       | D14.1            | todo                                   |            | Sweep foreign dead workers, never respawn; pool-local D12.7 orphan reaping. |
| D14.3 | Broker auth (`requirepass`) + AOF knob + broker-restart resume                          | D14       | —                | todo                                   |            | Password never logged; checkpoint stays authoritative — wiped broker must resume from checkpoint (§13.5). |
| D14.4 | `Jarvis2 cluster submit --backend slurm` (+ HTCondor)                                   | D14       | D14.1            | todo                                   |            | Renders/submits sbatch calling Phase-1 pools; dry-run prints script; INSTALL.md docs. |

| D15.1 | Point cache + `--warm-start`                                                            | D15       | —                | todo                                   |            | `DESIGN_RESULTS_ANALYSIS_2.0.md` §2/§3. Fingerprint = params(rounded per `cache_decimals`) + module config hash; miss-by-default; `cached_from` provenance; `cached` metric in run_summary. |
| D15.2 | `Jarvis2 analyze`: corner / marginals / best-fit via JarvisPLOT scenes                  | D15       | —                | todo                                   |            | Reads DATABASE only; reduced mode without chain columns; zero drawing code in HEP. |
| D15.3 | Posterior weighting mode (nested weights / MCMC post-burn-in)                            | D15       | D13.6, D15.2     | todo                                   |            | Reproduces V1 golden analysis numbers. |
| D15.4 | Portal File / HepMC / LHE exposure + V2 fixtures                                        | D15       | Portal release   | todo                                   |            | Same model as Portal 1.4.0 SLHA work; one real fixture per format. |
| D15.5 | Physics plot scenes I: `SceneMeta` (log axes, limits, latex labels, robust color clamp, best-fit, no-clobber) | D15 | — | todo |            | [`DESIGN_PHYSICS_PLOT_SCENES_2.0.md`](DESIGN_PHYSICS_PLOT_SCENES_2.0.md) §3/§4. Accept: regenerated iDM scene reproduces the maintainer's hand edits without hand edits; failed/−inf rows excluded; hand-edited files never overwritten. |
| D15.6 | Physics plot scenes II: method-aware (nested posterior-weighted, MCMC post-burn-in, profile-likelihood contours) | D15 | D15.5 | todo |            | Nested dead-point scatter demoted to `enable: false`; every emitted scene smoke-rendered via stock `jplot` in tests. |
| D15.7 | `Scan.plot` block + `Variables[].latex` in validation contracts + YAML_REFERENCE | D15 | D15.5 | todo |            | Optional additive keys (invariant 1); `Jarvis2 validate` coverage. |

| D16.1 | Skills library v1 (9 skills + index + template, cards validate-verified) | D16 | — | **done** | 2026-07-21 | Shipped with the design commit under `Jarvis-Books/Jarvis-HEP V2/skills/`. Cards for first-scan/MCMC/nested/external-calculator verified with `Jarvis2 validate` on `2daf417`. |
| D16.2 | Ship skills in-package + `Jarvis2 skill list / show <name>` | D16 | — | todo |            | Copy under `jarvishep2/skills/`; plain-text CLI; appears in `--help`. |
| D16.3 | Card CI: extract skill ```yaml blocks, `Jarvis2 validate --strict` complete cards | D16 | D16.2 | todo |            | Broken skill card fails the suite; partial snippets markable `no-validate`. |
| D16.4 | Coverage growth: SLHA calculator / EnvReqs tuning / cluster (post-D14) / analyze (post-D15.2) skills | D16 | D16.3 | todo |            | Binding rule (design §5): user-facing WPs are not done without their skill update. |

**Post-D12 polish already landed (no open WP; status only):** CLI `-v` / `ps` / `kill` /
logging flags; FileOperation SAMPLE process; full `samples.csv` + scan performance log;
**sample `op_count` single-increment contract** (`submit_result` only — see
`components/redis_queue.md` §5).

**Post-D13 polish already landed (no open WP; status only):** D13.8 flat feedback wire;
nested production hygiene (archive catch-up, MultiNest CSV parity, toy-logL disabled);
component multi-sink `logs/<scan>/{core,factory,sampler,archiver,datarecorder,worker-NN}.log`
+ `Jarvis-HEP.*` presentation labels; control archive ‰ at DEBUG; nested
`Results.summary()` → sampler.log; `project_template/bin/sampling/` Dynesty/MultiNest
YAML templates. Baseline docs: `README.md` + `components/cli.md`.

**Next pick order (current):** **D14** (cluster), then D15 (reuse/analysis).
**D8 stays fully deferred** (maintainer decision, re-confirmed 2026-07-16) — do not start
Agent JSON verbs until unparked. Designs:
[`DESIGN_SAMPLERS_2.0.md`](DESIGN_SAMPLERS_2.0.md), [`DESIGN_CLUSTER_EXECUTION_2.0.md`](DESIGN_CLUSTER_EXECUTION_2.0.md), [`DESIGN_RESULTS_ANALYSIS_2.0.md`](DESIGN_RESULTS_ANALYSIS_2.0.md).

Parallelism note: D11.1 has no dependency on parked D8 (Agent JSON can consume `RunOutcome`
later). D9.4/D9.6 may proceed without D8. Agent-side M4.5 (Jarvis-Agent repo) stays blocked
on parked D8.

---

## 4. Work Packages

Field key — **Goal**: outcome. **Design refs**: `DESIGN_2.0_DISTRIBUTED.md` sections (+ the
`docs/v2/discussions/` note). **Files**: existing files to read/modify; `NEW` marks suggested new
modules (keep the flat `jarvishep/` layout). **Steps**: order. **Accept**: definition of
done. **Rollback**: how it is disabled. **Out of scope**: excluded work.

---

> **WP-D0.1 … WP-D7.1 (all done) are archived verbatim** in
> [`archive/V2_PLAN_ARCHIVE_2026-07-14.md`](archive/V2_PLAN_ARCHIVE_2026-07-14.md).
> Only open WPs remain below.

### WP-D8.1 — Agent API facade: JSON envelope + `--validate` / `--results` / `--version-json`

- **Goal**: one-shot machine-readable verbs so Jarvis-Agent can validate configs and harvest
  results without importing `jarvishep2` or parsing human text.
- **Design refs**: [`DESIGN_AGENT_BRIDGE_2.0.md`](DESIGN_AGENT_BRIDGE_2.0.md) §4 (envelope,
  data shapes), §8 (version handshake); [`YAML_REFERENCE_2.0.md`](YAML_REFERENCE_2.0.md).
- **Files**: NEW `jarvishep2/api.py`, `jarvishep2/client.py` (additive flags),
  `jarvishep2/task_config.py` (reuse loaders), NEW `tests/test_agent_api.py`.
- **Steps**:
  1. Envelope helper (`api_version=1`, `kind`, `ok`, `data`, `error`); stdout = exactly one
     JSON object, humans on stderr; exit codes 0/1/2.
  2. `--validate --json`: config load + normalization + module/sampler inventory +
     `diagnostics[]`; **must not** connect Redis, spawn processes, or write files. Baseline
     diagnostics: missing `Runtime.redis` under `mode: redis` + workers>0 (error), unknown
     `Sampling.Method` (error).
  3. `--results --json (--task|--scan-dir) [--columns] [--limit] [--stats]`: read
     `DATABASE/samples.hdf5`, rows sorted best-`LogL`-first, limit default 50 / cap 1000.
  4. `--version-json`: `{package_version, api_version}`.
- **Accept**: envelope schema stable under tests (ok + error paths); validate catches the
  fakeredis split-brain config; results verbs return correct rows/stats on a parity-project
  scan; frozen human CLI output unchanged (existing CLI tests green).
- **Rollback**: flags are additive; revert the WP commit.
- **Out of scope**: `--status` (D8.2), strict diagnostics (D8.4), any daemon/RPC mode.

### WP-D8.2 — `run_state.json` writer + `--status --json` / `--monitor-json`

- **Goal**: scan lifecycle observable from outside the process, surviving process exit,
  without requiring Redis access.
- **Design refs**: Bridge §5 (schema, atomicity, staleness rule), §3 (precedence).
- **Files**: NEW `jarvishep2/run_state.py`, `jarvishep2/core.py` (lifecycle wiring),
  `jarvishep2/client.py` (`--status`, `--monitor-json`), `jarvishep2/dashboard.py` (JSON
  projection), NEW `tests/test_run_state.py`.
- **Steps**:
  1. Atomic writer (`tmp` + `os.replace`) to `<task_result_dir>/run_state.json`; heartbeat
     thread (5 s default) started in `bootstrap_distributed_runtime`, final write in
     `shutdown` (status `completed|failed|interrupted`).
  2. Populate schema per Bridge §5 (pid, counters from factory metrics, effective redis
     connection, checkpoint_file).
  3. `--status --json`: read the file, apply the 3×heartbeat staleness rule
     (`presumed_dead`), enrich with a live Redis snapshot when `running` and reachable.
  4. `--monitor-json`: JSON twin of `--monitor` (SnapshotReader → dict, no formatting).
- **Accept**: state file present and fresh during a 2-worker parity scan; final status
  correct for completed/failed runs; SIGKILL mid-scan ⇒ next `--status` reports
  `presumed_dead: true`; readers tolerate extra fields.
- **Rollback**: revert commit; file is write-only output (V2 never reads it back).
- **Out of scope**: multi-scan registry, event push/pub-sub.

### WP-D8.3 — Control-process graceful stop (SIGINT/SIGTERM → checkpoint → clean exit)

- **Goal**: a documented stop contract so an external supervisor (Jarvis-Agent, shell, CI)
  can interrupt a scan and always land on a resumable checkpoint.
- **Design refs**: Bridge §6; `DESIGN_2.0_DISTRIBUTED.md` §10 (checkpoint model).
- **Files**: `jarvishep2/core.py`, `jarvishep2/client.py`, `jarvishep2/redis_server.py`,
  NEW `tests/test_graceful_stop.py` (still open).
- **Status: partial (human path enough)** — agent contract still open with D8.
- **Done**:
  1. SIGINT/SIGTERM → `KeyboardInterrupt` → `run()` `finally` → idempotent `shutdown()`.
  2. Always stop managed Redis; refuse Ctrl+Z/SIGTSTP with a warning.
  3. CLI exit **130** on interrupt.
  4. **Force interrupt checkpoint** before teardown (`_save_interrupt_checkpoint`, reason
     `interrupt`); unit tests in `tests/test_graceful_stop.py`.
- **Remaining (parked with Agent D8)**:
  1. `run_state.status="interrupted"` once D8.2 lands; formal stop-ack for agents.
  2. Optional: live stop→resume→parity integration under fakeredis TCP (beyond unit tests).
- **Accept (remaining / agent)**: with D8.2, run_state reports interrupted; Agent stop-ack.
- **Rollback**: revert signal-handler commit.
- **Out of scope**: pause/steering; Worker-side signals (already shipped).

### WP-D8.4 — Strict-validate diagnostics

- **Goal**: `--validate` surfaces what normalization silently hides today, closing the
  YAML-review findings.
- **Design refs**: Bridge §4.2; `YAML_REFERENCE_2.0.md` §13 + Appendix A (A.2, A.3, A.5, A.10).
- **Files**: `jarvishep2/api.py`, `jarvishep2/runtime_config.py` (normalizers report
  rejected raw values instead of swallowing them), `tests/test_agent_api.py` (extend).
- **Steps**:
  1. Normalizers return (value, rejected?) or record coercions into a diagnostics sink;
     `--validate` reports each silently-coerced key as a warning with the rejected value.
  2. Dead-key warnings: `Runtime.Subprocess`, `execution.input[].save`.
  3. `--strict`: unknown-key warnings inside known blocks (`Runtime`, `Sampling`,
     `Calculators.Modules[]`, …) with a did-you-mean hint.
- **Accept**: `mode: redsi` yields a warning naming the rejected value (not silence);
  strict mode flags a misspelled `sample_artifacts` key; zero behavior change for runs
  (diagnostics only).
- **Rollback**: revert commit.
- **Out of scope**: hard schema validation / jsonschema; changing normalization semantics.

---

## 5. Verification Commands (Shared)

```bash
python3 -m pip install -e ".[dev,distributed,operas,plot]"
python3 -m pytest -q -rs tests/
Jarvis2 <calculator-reference>.yaml --check-modules              # slow-regime smoke
Jarvis2 <plot-scene>.yaml --plot                    # current render-only PLOT bridge
# integration against a real server: start redis (docker compose), set REDIS_URL, then
Jarvis2 <task>.yaml
```

`--benchmark` and `--convert` are not implemented in the current V2 CLI; do not use them as
acceptance commands until an explicit work package adds them. Throughput/parity currently run
through `tests/test_distributed_acceptance.py` and captured fixtures.

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
