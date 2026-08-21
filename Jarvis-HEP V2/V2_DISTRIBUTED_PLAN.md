# V2 Distributed Runtime — Development Plan (Agent Execution Playbook)

Last updated: 2026-08-21. CLI is **`Jarvis`** (older rows still say `Jarvis2` — same entry point).
Architecture: [`DESIGN_2.0_DISTRIBUTED.md`](DESIGN_2.0_DISTRIBUTED.md).
**This file is open work only.** Closed D0–D24 history:
[`archive/V2_PLAN_ARCHIVE_2026-07-14.md`](archive/V2_PLAN_ARCHIVE_2026-07-14.md) and
[`archive/reviews/`](archive/reviews/README.md).

**Shipped (do not re-open):** D0–D7 runtime, D9 first hardening, D10 AdaptiveBridson,
D11 UI, D12 calculator/UX, D13 samplers (through nested/MCMC), D17 strict cards,
D18 LibDeps, D19 archiver honesty, D20 multimode, D21 resume, D22 Mapper (except
D22.6 / D22.8), D23 Operas constants on the HEP side (except D23.6 in the Operas repo),
D24 `Jarvis man` (D24.11 partial).

**Structure track (new):** **D25** —
[`DESIGN_ARCHITECTURE_HARDENING_2.0.md`](DESIGN_ARCHITECTURE_HARDENING_2.0.md).
That document is the WP spec; this ledger only tracks status.

**Parked:** D8 Agent Bridge (maintainer, re-confirmed). Do not start Agent JSON verbs.

**Next pick (structure):** D25.6 (MCMC names) or D25.8 (runtime read model);
D25.9 waits on both D25.2 (done) and D25.3 (done).
**Next pick (feature):** D14.1 is allowed now that D25.3 landed. **Do not grow
`adaptive_bridson.py`** until D13.12. New Core behavior goes on the collaborators,
not the façade.

Audience: coding agents and maintainers. One WP = one PR = one session.

---

## 0. Agent Protocol

1. **Find your task.** First ledger row whose status is `todo` and whose dependencies
   are `done`. Structure vs feature: see Next pick above.
2. **Read before coding.** In order: this protocol, the WP's *Design refs*, the matching
   [`components/`](components/README.md) page, every file in *Files*.
3. **Stay inside the WP.** *Out of scope* lines are binding.
4. **Code is ground truth.** If a design line number drifted, trust the code, record the
   deviation in Notes.
5. **Verify.** A WP is not `done` with failing tests. After D25.7, "green" means the
   fast gate; slow tests are explicit.
6. **Close out.** Update the ledger row and `Last updated`. **Archive rule:** when a WP
   reaches `done`, do not leave a long essay here — move notes to `archive/` and keep a
   one-line pointer.
7. **Skills rule (D16):** a user-facing WP is not `done` until the affected `skills/`
   page exists or is updated.
8. **V1 is frozen.** Never land V2 work on the V1 line. Parity = captured V1 goldens.
   CI uses `fakeredis`.

When blocked: stop, write Notes, ask the maintainer. Do not improvise user-visible behavior.

---

## 1. Hard Invariants (Never Violate)

| # | Invariant |
|---|-----------|
| 1 | Task-YAML: new keys are **optional** with v1-equivalent defaults unless a WP says otherwise. |
| 2 | User CLI is **`Jarvis`** (legacy docs: `Jarvis2`). Frozen V1 CLI `Jarvis` 1.7.4 is a different line. |
| 3 | Output contracts frozen: HDF5/CSV, `DATABASE/` layout, `run_summary.{json,csv,txt}`. Archiver changes transport, not format. |
| 4 | Checkpoint UX: heartbeat (`EnvReqs.V2.checkpoint_heartbeat_sec`, default 30 s), `state.pkl` location, `--resume`. |
| 5 | V2 has **no thread path**. Parity is captured V1 goldens; unit tests use `fakeredis`. |
| 6 | One Worker = one Sample. Parallelism is inside a Sample (same-layer calculators). |
| 7 | Redis carries IDs + light dicts. Products stay on disk. |
| 8 | A `logger` is never serialized across a process boundary. |
| 9 | A failed Sample always leaves a readable log on disk. |
| 10 | `multiprocessing` always uses **spawn**. |
| 11 | `pack_id` traceability is preserved end to end. |
| 12 | Do **not** rename `SamplingVirtial` or the YAML key `make_paraller` until **D25.10** explicitly lifts this. |
| 13 | Reuse, don't reimplement: samplers, Likelihood/Operas/Calculator, scheduler, workflow, HDF5/CSV, project scaffolding. |
| 14 | If a WP touches a file that contains a listed P0/P1 bug, fix that bug in the same PR with a test. Do not fix bugs in files the WP does not touch. |

---

## 2. Milestone Map

| Milestone | Theme | Status |
|-----------|-------|--------|
| **D0–D7** | Foundations → acceptance | **done** (archive) |
| **D8** | Agent Bridge | **parked** |
| **D9** | Architecture Hardening 1 | **done** — see archived [`DESIGN_PRINCIPLES_REVIEW_2.0.md`](archive/reviews/DESIGN_PRINCIPLES_REVIEW_2.0.md) |
| **D10–D13** | AdaptiveBridson + UI + samplers | **done** except D13.12–D13.15 |
| **D14** | Cluster execution | **open** — [`DESIGN_CLUSTER_EXECUTION_2.0.md`](DESIGN_CLUSTER_EXECUTION_2.0.md) |
| **D15** | Result reuse + analysis + physics plots | **open** — [`DESIGN_RESULTS_ANALYSIS_2.0.md`](DESIGN_RESULTS_ANALYSIS_2.0.md), [`DESIGN_PHYSICS_PLOT_SCENES_2.0.md`](DESIGN_PHYSICS_PLOT_SCENES_2.0.md) |
| **D16** | Skills in-package | v1 shipped; D16.2–4 open |
| **D17–D24** | Strict cards, LibDeps, resume, Mapper, constants, man | **done** except leftovers listed below |
| **D25** | Architecture Hardening 2 | **open** — [`DESIGN_ARCHITECTURE_HARDENING_2.0.md`](DESIGN_ARCHITECTURE_HARDENING_2.0.md) |

---

## 3. Progress Ledger (open / parked / partial only)

Allowed statuses: `todo`, `in-progress`, `done`, `blocked`, `deferred`.

### D8 — Agent Bridge (parked)

Design: [`DESIGN_AGENT_BRIDGE_2.0.md`](DESIGN_AGENT_BRIDGE_2.0.md). Full WP text remains in
that design; do not execute until unparked.

| WP | Title | Status |
|---|---|---|
| D8.1 | JSON envelope + `--validate` / `--results` / `--version-json` | **deferred** |
| D8.2 | `run_state.json` + `--status --json` | **deferred** |
| D8.3 | Graceful stop | **partial** — human SIGINT → interrupt checkpoint shipped; agent stop-ack parked |
| D8.4 | Strict-validate diagnostics for the Agent API | **deferred** (D17 shipped the human/`Jarvis validate` path) |

### D13 leftovers

| WP | Title | Status | Notes |
|---|---|---|---|
| D13.12 | Decompose `AdaptiveBridson.run_adaptive` | todo | [`archive/reviews/CODE_REVIEW_2026-07-23.md`](archive/reviews/CODE_REVIEW_2026-07-23.md) §3. Extract fill / root-corr / bridge / advance behind `LoopDecision`. Behavior-preserving. Gate: `tests/test_adaptive_bridson.py`. **Do before the next feature lands in that file.** Not part of D25. |
| D13.13 | `convert` in `components/cli.md` | todo | Doc-only. `Jarvis convert` exists; the component page still omits it. |
| D13.14 | I/O validation follows Portal, not a pinned manifest | todo | [`archive/reviews/SCHEMA_REVIEW_2026-07-31.md`](archive/reviews/SCHEMA_REVIEW_2026-07-31.md). Accept a format when `available_io_formats()` supports it; manifest *enriches*. Relax `test_manifest_matches_builtin_portal_formats` to subset. Distributor↔manifest equality already exists; keep it (D25.1 will own the method side). MCMC Bounds stay open/`unstable`. AdaptiveBridson closed-block is **done** (`f983772`). |
| D13.15 | Triage ignored / standing test failures | todo | D25.7 converted the eight `--ignore`s to `pytest.mark.slow`. Remaining fast-gate failures (check-modules `wait_for_results` extra kwargs; official-library browse example; Examples PTMCMC Bounds vs `proposal_scales`) still need xfail/skip-with-reason or a fix. |

### D18 leftovers

| WP | Title | Status | Notes |
|---|---|---|---|
| D18.6 | `Utils.interpolations_1D` → Operas migration guidance | todo | [`V1_MIGRATION_STATUS_2026-07-31.md`](V1_MIGRATION_STATUS_2026-07-31.md) §6.1. Capability lives in Jarvis-Operas (`interp1.*`, `dmddxe.*`). Change the reject message + YAML_REFERENCE + a skill. Not a runtime feature. |
| D18.9 | Nuisance local-opt dimension coverage | todo | Confirm whether V1 joint multi-parameter nuisance exists beyond V2 `Profile1D`. |

### D22 leftovers

| WP | Title | Status | Notes |
|---|---|---|---|
| D22.6 | Parametric-equation example card + skill + YAML_REFERENCE §6.14 | todo | P4/P7: 200 points, `t,x,y`, `x²+y²=1` within 1e-12, `selection: "y > 0"` half-circle, plot axes `x,y`. |
| D22.8 | Schema vs runtime: Operas `input: ["x"]` | todo | `operas.py` accepts a bare string; schema requires object → `JV2-SCH-001`. Relax schema to string \| object. Second question: Calculator dump `variables[]` without `expression` — confirm Portal before changing schema. |

### D23 leftover (Operas repo)

| WP | Title | Status | Notes |
|---|---|---|---|
| D23.6 | JO docs: playbook “add a constant” + public surface | todo | Lives in the **Jarvis-Operas** repo, not HEP. HEP YAML_REFERENCE already documents `pdg.mZ`. |

### D14 — Cluster (open)

Design: [`DESIGN_CLUSTER_EXECUTION_2.0.md`](DESIGN_CLUSTER_EXECUTION_2.0.md). Prefer after D25.3.

| WP | Title | Depends | Status |
|---|---|---|---|
| D14.1 | Worker template over Redis + `Jarvis worker start --connect` | — | todo |
| D14.2 | Remote-aware watchdog + `<host>:<n>` ids | D14.1 | todo |
| D14.3 | Broker `requirepass` + AOF + broker-restart resume | — | todo |
| D14.4 | `Jarvis cluster submit --backend slurm` (+ HTCondor) | D14.1 | todo |

### D15 — Analyze / plots (open)

| WP | Title | Depends | Status |
|---|---|---|---|
| D15.1 | Point cache + `--warm-start` | — | todo |
| D15.2 | `Jarvis analyze` via JarvisPLOT scenes | — | todo |
| D15.3 | Posterior weighting (nested / MCMC post-burn-in) | D13.6, D15.2 | todo |
| D15.4 | Portal File / HepMC / LHE + fixtures | Portal release | todo |
| D15.5 | Physics plot scenes I (`SceneMeta`) | — | todo |
| D15.6 | Physics plot scenes II (method-aware) | D15.5 | todo |
| D15.7 | `Scan.plot` + `Variables[].latex` in contracts | D15.5 | todo |

### D16 — Skills (open)

Design: [`DESIGN_SKILLS_LIBRARY_2.0.md`](DESIGN_SKILLS_LIBRARY_2.0.md). v1 is under `skills/`.

| WP | Title | Depends | Status |
|---|---|---|---|
| D16.2 | Ship skills in-package + `Jarvis skill list/show` | — | todo |
| D16.3 | Card CI (`Jarvis validate --strict` on skill YAML) | D16.2 | todo |
| D16.4 | Coverage growth (SLHA / EnvReqs / cluster / analyze) | D16.3 | todo |

### D24 leftover

| WP | Title | Status |
|---|---|---|
| D24.11 | `--json` contract freeze + wheel / cold-start e2e | **partial** — JSON contract tests landed; wheel/cold-start optional polish |

### D25 — Architecture Hardening 2 (open)

Spec: [`DESIGN_ARCHITECTURE_HARDENING_2.0.md`](DESIGN_ARCHITECTURE_HARDENING_2.0.md) §4.
Do not copy that spec here.

| WP | Title | Priority | Depends | Status |
|---|---|---|---|---|
| D25.0 | Docs library hygiene | — | — | **done** (2026-08-20) |
| D25.1 | `SamplerSpec` method catalog (single source of truth) | P0 | — | **done** (2026-08-20) |
| D25.2 | Validation / CLI do not import the runtime graph | P1 | D25.1 | **done** (2026-08-20) |
| D25.3 | Split `Jarvis2Core` into façade + collaborators | P0 | D25.2 recommended | **done** (2026-08-21) |
| D25.4 | Split `RedisQueue` by keyspace | P1 | D25.3 | **done** (2026-08-21) |
| D25.5 | `SampleExecutor` extracted from `Worker` | P1 | D25.3 | **done** (2026-08-21) |
| D25.6 | MCMC / `Sampling` public names | P2 | D25.1 | todo |
| D25.7 | Test markers instead of `addopts --ignore` (closes D13.15) | P1 | — | **done** (2026-08-20; DX 2026-08-21: named `.py` files drop default `-m not slow`) |
| D25.8 | One runtime read model (`Runtime` vs `EnvReqs.V2`) | P2 | — | todo |
| D25.9 | Slice `client.py` | P3 | D25.2, D25.3 | todo |
| D25.10 | Package layout + optional `SamplingVirtial` rename | P3 | D25.3–D25.6 | todo |

---

## 4. Work-package details

Feature leftovers (D13.12, D14, D15, D16, D22.8, …) keep their original design
documents as *Design refs*. **D25 WP text lives only in**
[`DESIGN_ARCHITECTURE_HARDENING_2.0.md`](DESIGN_ARCHITECTURE_HARDENING_2.0.md) §4
(Goal / Files / Steps / Accept / Rollback / Out of scope). Parked D8 WP text lives
in [`DESIGN_AGENT_BRIDGE_2.0.md`](DESIGN_AGENT_BRIDGE_2.0.md).

Do not paste completed WP essays back into this file.

---

## 5. Verification Commands (Shared)

```bash
python3 -m pip install -e '.[distributed,dev]'
python3 -m pytest -q
# after D25.7:
# python3 -m pytest -q -m slow
Jarvis validate bin/quickstart_bridson_operas.yaml
Jarvis check bin/quickstart_bridson_operas.yaml
```

- Parity = captured V1 goldens, not a V2 thread mode.
- Rollback = revert the WP commit (V2 has no `mode: thread` fallback).
- CI must not require a Redis server: `fakeredis` for unit tests.

## 6. Deviation & Escalation

- Drifted line numbers / renamed symbols: expected; trust the code, note it, continue.
- Unreachable gate: do not lower it silently — mark `blocked` and ask the maintainer.
- Design vs frozen contract: the contract wins.
- Discovered prerequisite: add a ledger row, get sign-off, then execute.
