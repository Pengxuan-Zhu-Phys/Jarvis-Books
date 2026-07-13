# V2 Distributed Runtime — Development Plan (Agent Execution Playbook)

Last updated: 2026-07-14 (prototype phase closed — see
`PROTOTYPE_CLOSEOUT_REVIEW_2026-07-14.md`; D12 milestone added. Completed WPs and their
full notes are archived in `archive/V2_PLAN_ARCHIVE_2026-07-14.md`; this file tracks only
open work. Code baseline: `jarvis2` `0a5e85e`; test baseline: 283 passed, 1 skipped).
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
| **D8** | Agent Bridge (machine-readable control surface for Jarvis-Agent) | additive `--json` verbs (`--validate`/`--results`/`--status`/`--version-json`), `run_state.json` lifecycle file, SIGINT/SIGTERM graceful-stop contract; frozen human CLI untouched. Design: [`DESIGN_AGENT_BRIDGE_2.0.md`](DESIGN_AGENT_BRIDGE_2.0.md) |
| **D9** | Architecture Hardening (behavior-preserving refactors) | token resolution single-owner, dead async twins removed, `CalculatorModule`/`TaskFactory`/`Jarvis2Core` de-godded, sampler/IO registries (extension points for MCMC + SLHA), `Sample.info` single source of truth. Current baseline: 236 collected (235 passed, 1 flowchart-export skip) + golden parity; D9.4/D9.6 remain partial. Design + WP details: [`DESIGN_PRINCIPLES_REVIEW_2.0.md`](DESIGN_PRINCIPLES_REVIEW_2.0.md) §4 |
| **D10** | Adaptive Level-Set Sampler (first feedback-driven sampler, 2 ≤ d ≤ 5) | opt-in `hep:feedback` result channel (zero cost when off), `AdaptiveLevelSetSampler`: low-discrepancy gen-0 → crossing detection on a neighbor graph (exact Delaunay ridges d≤3 / approximate kNN d=4–5) → barrier-synchronized edge refinement → `levelset.json`; registered `stateless=False`; deterministic across worker counts; checkpoint at generation barriers; d≥6 rejected. Design: [`components/adaptive_voronoi_contour.md`](components/adaptive_voronoi_contour.md) |
| **D11** | User Interface & Integration Closure | one-intent CLI (`run/check/monitor/plot/portal/operas`) with compatibility aliases; truthful `RunOutcome` and machine-readable failures; Portal HEP formats exposed and discoverable; Operas validation/runtime parity; scan-driven PLOT + flowchart + D10 overlay. Review: [`USER_INTERFACE_INTEGRATION_REVIEW_2026-07-13.md`](USER_INTERFACE_INTEGRATION_REVIEW_2026-07-13.md) |
| **D12** | Calculator V1 parity + user-experience alignment ("V1 用户无感迁移") | unmodified V1 process-calculator card (Eggbox `Example_Bridson_process.yaml`) runs end-to-end with golden parity; V1-style core log rendering; flowchart.{json,png} export; `EnvReqs.V2` grouped settings (workers/factory/redis override); `Jarvis2 project create/pack/browse/fetch/info`; Jarvis-Examples-owned official catalog (zero HEP change to add examples). Review + design: [`PROTOTYPE_CLOSEOUT_REVIEW_2026-07-14.md`](PROTOTYPE_CLOSEOUT_REVIEW_2026-07-14.md) |

---

## 3. Progress Ledger

Allowed statuses: `todo`, `in-progress`, `done`, `blocked`.

> **Completed WPs are archived** with their full notes in
> [`archive/V2_PLAN_ARCHIVE_2026-07-14.md`](archive/V2_PLAN_ARCHIVE_2026-07-14.md)
> — D0.1–D0.5, D1.1–D1.2, D2.1–D2.3, D3.1–D3.3, D4.1–D4.2, D5.1–D5.2, D6.1–D6.2,
> D7.1 (2026-06-28/29); D9.1–D9.3, D9.5, D9.7–D9.8 (2026-07-10); D10.1–D10.2
> (2026-07-11); D11.4a–D11.4d (2026-07-13); plus the pre-V2 M0/M1 row. This table
> keeps only open work: todo / in-progress / partial / blocked.

| WP   | Title                                                                                  | Milestone | Depends on       | Status                                 | Date       | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ---- | -------------------------------------------------------------------------------------- | --------- | ---------------- | -------------------------------------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| D8.1 | Agent API facade: JSON envelope + `--validate`/`--results` + `--version-json`        | D8        | —                | todo                                   |            | Design: `DESIGN_AGENT_BRIDGE_2.0.md` §4. NEW `jarvishep2/api.py`. |
| D8.2 | `run_state.json` writer + `--status --json` / `--monitor-json`                        | D8        | D8.1             | todo                                   |            | Bridge §5. NEW `jarvishep2/run_state.py`; heartbeat thread in core. |
| D8.3 | Control-process graceful stop (SIGINT/SIGTERM → checkpoint → clean exit)              | D8        | —                | todo                                   |            | Bridge §6. Today only Workers handle signals; control process has none. |
| D8.4 | Strict-validate diagnostics (silent-coercion warnings, dead keys, unknown keys)       | D8        | D8.1             | todo                                   |            | Bridge §4.2; findings from `YAML_REFERENCE_2.0.md` Appendix A. |

| D9.4 | `TaskFactory` de-singleton + internal MonitorLoop/Watchdog split + honest metrics     | D9        | D8.3             | **partial**                            | 2026-07-10 | Core owns explicit `TaskFactory`; `get_instance` deprecated shell; honest `None` metrics. MonitorLoop/Watchdog class split deferred. |
| D9.6 | `Sample` single source of truth (`info` becomes projection)                           | D9        | D9.1, D9.3       | **partial**                            | 2026-07-10 | `merge_observables` / `set_status` / `record_pack_id` write APIs; Worker uses them. Full `info` projection deferred. |

| D10.3 | Finalize + outputs (d=2): polyline chaining, `levelset.json`, PLOT overlay hook       | D10       | D10.2            | **partial**                            | 2026-07-11 | `levelset.json` + polylines_u; PLOT overlay hook deferred. |
| D10.4 | Determinism + checkpoint/resume + core test suite (§9.1–9.8); YAML_REFERENCE §6.9 entry | D10       | D10.2            | **partial**                            | 2026-07-11 | Tests: circle 2D, seed reproducibility, kNN vs Delaunay, feedback unit, d=5 Sobol smoke; full Hausdorff efficiency gate / YAML_REFERENCE deferred. |
| D10.5 | Dimension extension d=3–5: `KNNGraph` proximity mode, Sobol gen-0 (d=5; Bridson grid is ~344 MB at d=5 — C10), d=3 crossing cloud, slice projections, §9.9–9.10 tests | D10       | D10.2            | **partial**                            | 2026-07-11 | KNNGraph + Sobol d=5 gen-0 + slice_projections in finalize; dedicated d=3/d=4 fixture tests deferred. |

| D11.1 | Truthful run result + machine-readable failure contract (`RunOutcome`)                 | D11       | —                | todo                                   |            | Fix all-failed exit-0; unify CLI/run_summary/Agent counters; add `error_type`, `error`, `failed_module`; D8.1 consumes the same contract. |
| D11.2 | One-intent CLI + compatibility aliases + Redis override fix                            | D11       | D11.1            | todo                                   |            | `run/check/monitor/plot/portal/operas`; reject mode conflicts; parser overrides default to `None`; retire dead `--pid`. |
| D11.3 | Portal HEP format exposure + discovery + real SLHA fixture                             | D11       | D11.2            | todo                                   |            | Bridge works today for CSV/DAT/JSON/TSV/Wolfram; expose SLHA/xSLHA/File in Portal, then verify end to end. |
| D11.4 | Operas strict validation + signature/logger/Functions parity + discovery               | D11       | D11.2            | todo                                   |            | Keep importlib-first; validate `call_mode`; add thin `list/info` proxy and explicit expression-function registration. **Confirmed 2026-07-14**: add argument-resolution layer over snapshotted `numpy_impl` — freeze `inspect.signature` at preload, filter kwargs at call (pass `logger`/`observables` only if declared; `**kwargs` keeps full passthrough), friendly missing-param error instead of bare `TypeError` (`PROTOTYPE_CLOSEOUT_REVIEW_2026-07-14.md` §3.2). |
| D11.5 | Scan-driven PLOT scene + workflow graph + AdaptiveLevelSet overlay                      | D11       | D10.3, D11.2     | todo                                   |            | Current bridge is render-only. Preserve JarvisPLOT ownership of all rendering algorithms. |

| D12.0 | Jarvis-Operas → core dependency + expression-scan scope fix                             | D12       | —                | todo                                   |            | User decisions 2026-07-14 (`PROTOTYPE_CLOSEOUT_REVIEW_2026-07-14.md` §3.1/§3.2): (a) move `jarvis-operas` from `[operas]` extra into core deps in `pyproject.toml`; update README plugin table + INSTALL.md; (b) restrict `expression_uses_operas_function` scans to expression fields only (not calculator `cmd`/paths) — now hygiene, not a crash fix. Argument-resolution layer for operator kwargs (signature freeze at snapshot, filtered call, friendly missing-param error) is confirmed and folded into D11.4. |
| D12.1 | Calculator V1-YAML parity (string commands, `${source}/${path}`, module `selection`)    | D12       | D11.3            | todo                                   |            | Review §4.1: `CalculatorSpec` silently drops V1 string-list installation/initialization/commands — normalize to `{cmd, cwd}`; interpolate `${source}/${path}`; tolerate `make_paraller`/`modes`; accept = unmodified Eggbox process card + golden parity. SLHA path needs D11.3. |
| D12.2 | Core logging V1 visual parity (formatter, banner, file layout)                          | D12       | —                | todo                                   |            | Review §5.1: keep D0.3 queue architecture; add V1-style formatter (`·•·`/`Ϡ`, module color, `MM-DD HH:mm:ss.SSS`, raw passthrough), logo banner, `<scan>/jarvis.log` layout, per-sample log format audit. Do before D12.1 acceptance (sample-log golden depends on it). |
| D12.3 | Workflow flowchart export + JarvisPLOT rendering                                        | D12       | D11.5            | todo                                   |            | Review §5.2: port V1 `export_flowchart_semantics` onto V2 execution plan; render via `plot_bridge`; `--skip-draw-flowchart` compat; un-skip golden test in `test_workflow_execution_plan.py`. |
| D12.4 | `EnvReqs.V2` grouped settings (workers/factory/redis override)                          | D12       | —                | todo                                   |            | Review §4.3: extend whitelist to grouped schema incl. optional `redis.{host,port,db}` override over `INTERNAL_REDIS_CONFIG` default; tolerate V1 sibling keys (`Python`, `CERN_ROOT`); relax external Runtime-defaults strictness. |
| D12.5 | `Jarvis2 project` subcommands (create/pack/browse/fetch/info)                            | D12       | D11.2            | todo                                   |            | Review §5.3: port V1 `project_scaffold` / `project_packager` / `project_template` / `official_project_library`; same `jarvis.project.yaml` layout. |
| D12.6 | Jarvis-Examples-owned official catalog (schema-versioned remote index)                  | D12       | D12.5            | todo                                   |            | Review §5.4: move catalog JSON into Jarvis-Examples repo; HEP keeps only index URL + schema version + fetch logic; local cache fallback replaces packaged copy; cross-repo change needs user go-ahead on Jarvis-Examples. |

Parallelism note: D0.1 / D0.2 / D0.3 are independent and may proceed in any order. D3.3 is
independent of the rest of D3. D8.3 is independent of D8.1/D8.2 and may land first; the
agent-side milestone M4.5 (Jarvis-Agent repo) depends on D8.1–D8.3. D9.1/D9.2/D9.7/D9.8
have no file overlap with D8 and may run in parallel with it; D9.4/D9.5 touch
`core.py`/`client.py` and must wait for D8.1–D8.3; D9.6 goes last. D11.1/D11.2 should be
implemented together with D8's JSON/control surface so the CLI is not redesigned twice.

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
- **Files**: `jarvishep2/core.py` (signal handlers + orderly teardown),
  `jarvishep2/client.py` (exit code), NEW `tests/test_graceful_stop.py`.
- **Steps**:
  1. Install SIGINT/SIGTERM handlers in the control process (today only Workers have them):
     stop proposing → force an immediate runtime checkpoint → stop Archiver (drain) → factory
     shutdown → `run_state.status="interrupted"` (if D8.2 landed) → exit 0.
  2. Target ≤30 s teardown on an idle queue; second signal escalates to immediate abort.
  3. Document the process-group kill fallback (SIGKILL → no final write; resume from the
     last periodic checkpoint).
- **Accept**: SIGINT mid-scan on a parity project exits 0 with a valid checkpoint;
  `--resume` completes the scan with output parity; test drives the full
  stop→resume→parity loop under fakeredis TCP.
- **Rollback**: revert commit (handlers additive).
- **Out of scope**: pause/steering semantics; Worker-side signal behavior (already shipped).

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
