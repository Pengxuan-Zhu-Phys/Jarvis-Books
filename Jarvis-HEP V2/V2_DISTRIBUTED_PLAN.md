# V2 Distributed Runtime — Development Plan (Agent Execution Playbook)

Last updated: 2026-08-02. Branch `jarvis2`; D17.1–D17.9, D18.1–D18.5, and D20.1–D20.8 are complete. **V1 migration status + open gaps: [`V1_MIGRATION_STATUS_2026-07-31.md`](V1_MIGRATION_STATUS_2026-07-31.md)** (11 V1 samplers unmigrated after nuisance reclassification; `--refs`/`--benchmark` absent). LibDeps builds once in the control preflight and reuses stamps ([`DESIGN_LIBDEPS_2.0.md`](DESIGN_LIBDEPS_2.0.md)). Multimode Calculator is shared-only with dot-qualified logical nodes, physical-pack affinity, bounded affinity waiting, and pre-acquire `selection` filtering ([`DESIGN_CALCULATOR_MODES_2.0.md`](DESIGN_CALCULATOR_MODES_2.0.md)). D20 targeted gates are green; the full suite records 735 passed / 6 unrelated known failures, tracked by D13.15.
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
| **D21** | 断点续跑（Checkpoint / Resume 重做） | 维护者定调 2026-08-02：**只有落进 HDF5 的点才算完成**，其余一律重算；**generator 必须恢复到"在算的点尚未提交之前"的状态**。目标是 resume 后的轨迹与从未中断过的运行等价。现状实测为**坏的**：4000 点扫描中断在 3132 点后 resume，DB 变成 **7132 行 / 4000 unique / 3132 个 uuid 各重复 2 次**——已完成的点被整体重算并重复写盘。设计: [`DESIGN_RESUME_2.0.md`](DESIGN_RESUME_2.0.md) |

---

## 3. Progress Ledger

Allowed statuses: `todo`, `in-progress`, `done`, `blocked`.

> **Completed WPs are archived** with their full notes in
> [`archive/V2_PLAN_ARCHIVE_2026-07-14.md`](archive/V2_PLAN_ARCHIVE_2026-07-14.md)
> — D0.1–D0.5, D1.1–D1.2, D2.1–D2.3, D3.1–D3.3, D4.1–D4.2, D5.1–D5.2, D6.1–D6.2,
> D7.1 (2026-06-28/29); D9.1–D9.3, D9.5, D9.7–D9.8 (2026-07-10); D10.1–D10.2
> (2026-07-11); D11.4a–D11.4d (2026-07-13); D9.4/D9.6, D10.3–D10.5, D11.1–D11.5,
> D12.0–D12.8 (2026-07-14…16); D13.1–D13.5b (2026-07-16/17); D17.1–D17.9 (2026-07-31/08-01); D18.1–D18.5 (2026-08-01); D20.1–D20.8 (2026-08-02); D13.6–D13.10
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

| D13.14 | Task-card schema: I/O follows Portal (not the pinned manifest) + AdaptiveBridson sub-block schema | D13 | — | todo |            | [`SCHEMA_REVIEW_2026-07-31.md`](SCHEMA_REVIEW_2026-07-31.md) (scoped after maintainer feedback: MCMC family is **not migrated**, so leave its schemas open; I/O authority is **Portal source**, not HEP's registry). **(1) HIGH — I/O validation pins the format list.** `_validate_selected_io` never consults Portal; a name absent from `manifest["io"]` is a hard `JV2-SCH-002`. Simulated a Portal upgrade adding `HepMC`: the card is rejected ("register a local schema in schema/manifest.json") even though Portal can read it — the exact coupling the plugin design exists to avoid — and `test_manifest_matches_builtin_portal_formats` asserts **set equality**, so the same upgrade turns HEP's suite red with no HEP change. Invert: accept when `available_io_formats(direction)` supports it (manifest schema only *enriches*); no schema → accept with at most an info note; not Portal-supported → hard error listing Portal's actual formats. Relax the test to a **subset** assertion. **(2) HIGH — `AdaptiveBridson` is the one migrated method with zero coverage.** `Sampling.AdaptiveBridson` is typed as a bare object: `{initial_radiuz: 0.08, 瞎写: 1}` validates identically to the correct card, and the sampler silently uses defaults. Keys are already specified in YAML_REFERENCE §6.9 — transcription, not design. (Bridson/Random/CSV/Grid are covered by schema; Dynesty/MultiNest by the legacy contracts layer.) **(3)** A **registered** method with no manifest entry fails open — simulated: garbage `Bounds` → zero diagnostics. Add the missing Distributor↔manifest test (Portal↔manifest has one) plus a diagnostic. **(4)** `test_manifest_gives_each_sampling_method_its_own_schema` asserts only URI uniqueness. **(5)** root `$id` ≠ filename (cosmetic). **Out of scope**: MCMC-family `Bounds` schemas — revisit when that family is actually migrated. **Open question for maintainer**: the un-migrated MCMC family is registered and passes validation today, so users can launch it; consider marking it experimental at the registry level. |
| D13.15 | Triage the 8 pre-existing test failures (fix or quarantine with a reason) | D13 | — | todo |            | The plan header has recorded "608 passed with 8 pre-existing failures" since 2026-07-29. A standing set of known-failing tests erodes the suite's value as a gate: the next agent cannot tell its own breakage from the background, and Protocol §5 ("a WP is not done with failing tests") becomes unenforceable. Triage each: fix it, or mark it `skip`/`xfail` **with the reason and a tracking note** so green means green. Record the outcome here. |
| D13.12 | Decompose `AdaptiveBridson.run_adaptive` (959-line method, 9 exit points) | D13 | D13.11 | todo |            | `CODE_REVIEW_2026-07-23.md` §3. `adaptive_bridson.py` is 4 068 lines; the main loop inlines gen-0 bootstrap, fill passes, root correction, gap bridging, and advance/converge into one scope with nine inline `return`s. Extract each seam behind an explicit `LoopDecision` (continue / advance / stop-with-reason) so exit conditions become individually testable. Behavior-preserving; existing `test_adaptive_bridson.py` is the gate. Do before the next feature lands in that file. |
| D13.13 | Doc/design debt: `convert` in `components/cli.md` | D13 | — | todo |            | `CODE_REVIEW_2026-07-23.md` §2.2. `Jarvis2 convert` is missing from `cli.md` (all 10 other subcommands are there). ~~Install-reuse design doc~~ — **done 2026-07-23**: [`DESIGN_CALC_INSTALL_CONTROL_2.0.md`](DESIGN_CALC_INSTALL_CONTROL_2.0.md). YAML_REFERENCE §9.4's false idempotency note corrected 2026-07-23; its final rewrite ships with D13.11. |



| D18.6 | `Utils.interpolations_1D` → Jarvis-Operas 迁移指引（不是补功能） | D18 | — | todo |            | [`V1_MIGRATION_STATUS_2026-07-31.md`](V1_MIGRATION_STATUS_2026-07-31.md) §6.1. **15/65 张真实卡片**用了这个块；V2 目前只回一句 `"Remove top-level Utils: it is not supported by V2."`（`task_schema.py:28`）。但**能力其实已经在 Jarvis-Operas 里**：`interp1.interp1_{xy_flat,xlog_yflat,xflat_ylog,xy_log}` 正好覆盖 V1 的 logX/logY 四种组合，`dmddxe.*` 已内置整套直接探测限制（CDEX/CDMS/DAMIC/DarkSide50/AllLimits*），自有曲线可用 `add_interpolation_namespace` 注册。所以这是**迁移路径缺失而非功能缺失**——用户被告知删掉却不知道去哪找。改成指路消息 + YAML_REFERENCE 对照写法 + 一篇 skill（D16 规则）。V1 语义参考：`core.py:343-353` 把插值注册进 `_funcs`，与 Operas 函数同一个表达式命名空间，故 `XenonSD2019(mchi)` 可直接写在 LogLikelihood/selection 里。 |
| D18.7 | 补齐 5 种变量分布（Binomial/Poisson/Beta/Exponential/Gamma） | D18 | — | todo |            | §6.2. V1 `Sampling/variables.py` 支持 10 种，V2 白名单只有 5 种并硬拒（`JV2-VAR-002`），V1 卡片写 `Poisson` 直接失败。**0 张现有卡片用到**，故不阻塞任何工作流，属 V1 卡片兼容缺口。V1 实现是几行 numpy 调用，照抄即可；补齐后同步 schema 的 `distribution.type` 枚举与 YAML_REFERENCE。 |
| D19.1 | **HIGH — 小规模扫描全部误报失败**：Archiver 部分批次永不刷盘 | D19 | — | **done** | 2026-08-02 | **已修并实测确认（2026-08-02 复核）**：`archiver.py:158` 现为 `bool(self._batch) and (len(batch) >= batch_size **or** 间隔已到)`。端到端实跑：12 样本扫描 → `RunOutcome status=success archived=12 exit=0`；20000 样本扫描 → `archived=20000 exit=0`。以下为原始记录。发现于 2026-08-01 的 LibDeps 实跑验收。`archiver.py:157` `_flush_due_locked` 用的是 **AND**：`len(batch) >= batch_size` **and** 间隔已到——所以**不满一批的数据永远不会按间隔刷盘**，`flush_interval_sec` 实际只对满批起节流作用，起不到延迟上界的作用。后果：样本数少于 `batch_size`（默认 **200**）的扫描在运行期一行都不写，控制进程的 drain 等待循环看到 `written==0`、`archive_q==0`、workers 已完成 → 5 秒后判定 stall → **`Run failed` + 退出码 1**，而数据其实在 teardown 的 `stop(drain=True)` 强制刷盘时全部落盘。实测：两次 21/23 样本的扫描均报 *only 0/23 DATABASE rows written* 并退出 1，而 `samples.hdf5`/`samples.csv` 里 **44 行数据完好**。**每个小于 200 样本的扫描都会中招**（含大量冒烟/测试跑）。雪上加霜：报错文案 *Likely early SAMPLE bucket pack pruned dirs...* 指向的是另一个（已修的）bug，会把排查者引到错误方向。修法：刷盘判据改为 **OR**（满批 **或** 间隔到），这才是 flush interval 的通常语义；并修正 stall 文案。 |
| D18.8 | `Jarvis2 refs`（V1 `--refs`）：打印内置采样器的参考文献 | D18 | — | todo |            | 维护者确认要补（2026-08-01）。V1 `client.py:158 _render_refs_text()` = logo + 文献列表，供用户写论文引用 dynesty/emcee 等。这是唯一直接影响最终用户产出的 V1 CLI 缺口，实现成本低（静态引用表 + 一条命令）。 |
| D18.9 | 核实 nuisance 局部优化的维度覆盖 | D18 | — | todo |            | 维护者澄清：nuisance sampler 由 **Worker 对某个参数点做局部优化**——与 D13.4 已交付的 `nuisance_optimize` 定位一致，故它**不是**未迁移采样器（未迁移数由 12 修正为 11）。待核实：V2 的 `Profile1D` 只做**一维**黄金分割；若 V1 支持多个 nuisance 参数**联合**优化，则属部分迁移，需补多维优化。 |
| **D21.1** | **HIGH — resume 重复写盘**：落盘对账 + 写入端去重 | D21 | — | todo |            | 设计 [`DESIGN_RESUME_2.0.md`](DESIGN_RESUME_2.0.md) §4 I1/I3。**实测的坏行为**：4000 点扫描中断在 3132 点，`--resume` 后 DB = **7132 行 / 4000 unique / 3132 个 uuid 各重复 2 次**。根因链（逐环实测）：(a) `mark_completed()` **生产代码从未调用**，真实 checkpoint 内容为 `submitted=3132, completed=0`；(b) 于是 `repropose_unfinished() = submitted − completed` = 全量重投；(c) 写入端不去重。**决定设计的关键事实**：`stateless_batch.py:13` 的 `uuid = sha256(prefix:seed:accepted_index)` 是索引的**纯函数**，且 HDF5 每条记录都带 `uuid` 字段——所以"哪些点已完成"可以在 resume 时精确重建，不需要任何运行期簿记。做法：resume 启动读 DATABASE 的 uuid 列作为**唯一真相**，`repropose = submitted − persisted`；archiver 用同一集合播种去重集，写前查重。**单独做这一条即可消灭重复写入**。影响范围：`FixedSetSampler` 家族（Random/Grid/Bridson）+ `CSVSampler` 全中；`AdaptiveBridson` 不受影响（`feedback_sampler.py:159` 会填 `_completed_uuids`）。 |
| **D21.2** | Archiver 落盘确认上报 Redis | D21 | — | todo |            | 设计 §6。进程模式 `ArchiverProcess.persistence_state()` **按设计返回 `acked_uuids: []`**（`archiver.py:665`），所以 checkpoint 里没有任何落盘真相（实测 checkpoint 的 `persistence.acked_uuids: 0`，而同时 `records_written: 18175`——只知道数量不知道身份）。改为 **flush → fsync → `SADD hep:archived:<scan>`**，顺序不可颠倒；`persistence_state()` 读该集合。Redis 只是快路径，D21.1 的落盘对账仍是最终权威（覆盖 fsync 与 SADD 之间崩溃的窗口）。 |
| **D21.3** | 滚动安全恢复点（枚举型采样器） | D21 | D21.2 | todo |            | 设计 §4 I2 / §5.1，兑现维护者的 S2（*"sampler 需要恢复到那些正在计算的点没有提交之前的状态"*）。**不能指望把 numpy 随机流倒带**——它是单向的。改为 checkpoint **永不写当前活状态，只写滚动恢复点**：`flush_batch()` 之前捕获 `(np.random.get_state(), _index, _accepted_index, ready_queue)`；**只有该批全部 uuid 落盘确认后**，恢复点才前进到下一批之前。于是 checkpoint 里的 generator 永远处于"在途点尚未提交之前"，resume 恢复它并前向生成就会重新提出这些点。滞后有界：最多 `_max_inflight`（默认 `max(batch_size, workers*4)`）个样本，**不是**落后 30 秒心跳。 |
| D21.4 | 反馈型采样器的世代屏障恢复点 | D21 | D21.3 | todo |            | 设计 §5.2。`FeedbackSampler`/`AdaptiveBridson` 的提议依赖既往**结果**，恢复点必须落在**世代屏障**上——而这个屏障它已经有了。规则同构、粒度更粗：一个世代全部落盘确认后才推进恢复点；resume 回到最后一个完整落盘的世代边界重放。代价是最多重算一个世代，换取自适应逻辑严格可重现。 |
| D21.5 | `safe_barrier_confirmed` 改为实算；删除 `mark_completed` 死接口 | D21 | D21.2 | todo |            | 设计 §8.4/§8.5。`core.save_runtime_checkpoint` 把 `safe_barrier_confirmed=True` **写死**，从不调用已实现的 `safe_barrier_ready()`；实测真实 checkpoint 里 `reason=checkpoint_heartbeat` 却 `safe_barrier_confirmed=True`。改为真调，心跳 checkpoint 如实记 `False`。`mark_completed()` 在新设计里不再需要（完成与否由盘上说了算），连同 `_completed_uuids` 一并移除，避免留下第二个"看起来是真相"的来源。 |
| D21.6 | 失败样本的落盘语义 | D21 | D21.1 | todo |            | 设计 §8.1。物理性失败的点永远不会进 HDF5，若不处理，resume 会**无限重算它**。要求失败样本以带 `status` 的记录落盘。**需先核实现状**：`worker.py:804-811` 的 `except` 记录失败后走 `finally` 的归档路径，看起来会落盘，但本次实测 `failed=0`，未取得实证。 |
| **D21.7** | 端到端回归测试 A1–A7 | D21 | D21.1–D21.4 | todo |            | 设计 §10。**必须是端到端行为断言，不是单元桩**——现有的 `test_kill_and_resume_completes_without_duplicate_uuids` **自己调用了 `sampler.mark_completed(uuid)`**（第 333 行），所以它验证的是库契约而非生产接线，这正是本 bug 存活至今的原因。核心断言：中断 + resume 后 **DB `unique_uuids == rows`**；重算量 ≤ `max_inflight` + 未落盘批次（不是全量）；同 seed 下"未中断"与"中断+resume"结果集完全一致；删掉 checkpoint 仍能靠 DATABASE 对账续跑（自愈性）。 |
| **D21.8** | **HIGH — 崩溃后 resume 根本起不来**：进程生命周期 | D21 | — | todo |            | 设计 §9。实测：SIGKILL 控制进程后 `Jarvis-Redis` + 2 Worker + Archiver + 2 FileOperation **全部存活无人收尸**，随后三条路全堵：`Jarvis2 kill R1 -y` → *Refusing unverified runtime: control PID does not match metadata.*；`Jarvis2 ps R1` 同上；`run --resume` → *Redis port 6379 is already in use*；而 `Jarvis2 monitor` 照常列为"运行中"。**`kill` 恰恰在唯一需要它的场景下拒绝工作**——控制 PID 对不上 metadata，正是"它已经死了"的定义。修法：控制进程写租约（runtime.json + Redis TTL），Worker 反读租约、过期自杀；`kill` 的判断**反转**（控制 PID 不存在 ⇒ stale runtime ⇒ 允许清理）；`--resume` 接管或清理 stale runtime。另需处理 **HDF5 陈旧写标志**——实测 SIGKILL 后该文件连只读都打不开（`file is already open for write ... may use h5clear`），resume 第一步就会失败。独立观察：机器上累积 **13 个 ppid=1 的孤儿 `Jarvis2-FileOperation`**，最老 **2 天 13 小时**，早于本次会话，说明泄漏不只发生在崩溃路径。 |

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
