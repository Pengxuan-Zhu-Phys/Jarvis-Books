# Jarvis-HEP V2 — Design & Development Docs

This is the home for the **Jarvis-HEP V2** design and development documentation. It is
**V2-only**. The V1 line (the single-process thread runtime, `Jarvis` 1.7.4) is **fully
archived in the Jarvis-HEP repo** and is out of scope here — nothing in this tree depends on it.

## The architecture in one line

Redis task queue → long-lived multiprocess Workers (one Sample per Worker; same-layer
calculators run concurrently inside a Sample) → async NAS Archiver → HDF5/CSV. A read-only
`op_count`-driven monitor at 60 Hz. Sized for the **slow regime** (external calculators).
**V2 has exactly one runtime path: Redis + `spawn` Workers** — no thread mode, no fallback.

## Layout

```
Docs/
├── README.md                       ← you are here (master index)
├── DESIGN_2.0_DISTRIBUTED.md       ← architecture, single source of truth
├── V2_DISTRIBUTED_PLAN.md          ← execution playbook — **open work only** (D8 parked; D0–D12 archived)
├── PROTOTYPE_CLOSEOUT_REVIEW_2026-07-14.md ← prototype closeout review + D12 (Calculator/UX parity) design
├── archive/                        ← completed WPs moved out of the plan (full ledger notes + WP details)
│   └── V2_PLAN_ARCHIVE_2026-07-14.md ← D0–D7 all, D9/D10/D11 done rows; frozen history, read on demand only
├── DESIGN_PORTAL_IO_2.0.md         ← Portal formats + HEP FileOperation for SAMPLE save
├── DESIGN_SAMPLERS_2.0.md          ← D13: MCMC/nested/nuisance on the feedback channel
├── DESIGN_FEEDBACK_RETURN_2.0.md   ← D13.8: projected hep:feedback (default uuid+LogL)
├── DESIGN_YAML_VALIDATION_2.0.md   ← D13.9: early task-YAML gate (contracts + Jarvis2 validate)
├── DESIGN_ADAPTIVE_BRIDSON_LIVE_BAND.md ← AdaptiveBridson **final** algorithm flow (outer/core + endpoints)
├── DESIGN_CLUSTER_EXECUTION_2.0.md ← D14: remote worker pools, SLURM, broker auth/AOF
├── DESIGN_RESULTS_ANALYSIS_2.0.md  ← D15 (todo): warm-start cache, Jarvis2 analyze, Portal formats
├── DESIGN_CALCULATOR_MODES_2.0.md  ← D20: 多模式 Calculator（modes 展开为兄弟模块）
├── DESIGN_LIBDEPS_2.0.md           ← D18: LibDeps 装一次的共享依赖（V1 对齐）
├── V1_MIGRATION_STATUS_2026-07-31.md ← 迁移状态 + 未迁移功能清单（实测，非文档转抄）
├── CODE_REVIEW_2026-07-23.md       ← latest code review: 1 high + 3 medium findings → D13.11–13
├── DESIGN_STRICT_VALIDATION_2.0.md ← D17: strict card validation (illegal key/type → exit); corpus gate first
├── DESIGN_CALC_INSTALL_CONTROL_2.0.md ← D13.11: operator-facing calculator reinstall control (jarvis_install.json)
├── DESIGN_PHYSICS_PLOT_SCENES_2.0.md ← D15.5–7 (todo): physics-grade auto plot YAML (log axes, posterior weights, profile contours)
├── DESIGN_SKILLS_LIBRARY_2.0.md    ← D16: kill the YAML complexity barrier — one intent, one runnable card
├── skills/                         ← **user skills library v1** (9 skills, cards validate-verified)
│   └── README.md                   ← "我想……" index; start here as a user
├── DESIGN_SAMPLE_COORDINATION_2.0.md ← **accepted**: uuid key; task=u_coords; feedback=logL; stage opt-in
├── DESIGN_SAMPLE_PROGRESS_MONITOR.md ← monitor stage-on-heartbeat (aligned with coordination design)
├── YAML_REFERENCE_2.0.md           ← as-built task-YAML reference (every key, defaults, gaps)
├── YAML-Example/                   ← public YAML recipes (per method)
│   └── ADAPTIVE_BRIDSON.md         ← AdaptiveBridson: minimal + full YAML recipes
├── CODE_REVIEW_2.0.md              ← functional-completeness review (scope gaps, tests, risks)
├── DEVELOPMENT_REVIEW_2026-07-10.md ← current development status, bugs, priorities, next phases
├── USER_INTERFACE_INTEGRATION_REVIEW_2026-07-13.md ← current UI + PLOT/Portal/Operas review
├── EGGBOX_BRIDSON_OPERAS_ACCEPTANCE_2026-07-13.md ← first unmodified V1 YAML full-run gate
├── V1_LIGHTWEIGHT_FUNCTION_MIGRATION_2026-07-13.md ← complete 38-function Expression Core migration
├── OPERAS_DYNAMIC_FUNCTION_DISCOVERY_2026-07-13.md ← external registered function discovery
├── DESIGN_PRINCIPLES_REVIEW_2.0.md ← SOLID/DRY/pattern review + D9 refactor work packages
├── DESIGN_AGENT_BRIDGE_2.0.md      ← V2 ↔ Jarvis-Agent bridge: Agent API + native hep_* tools
├── Jarvis-Agent_Design_Document.md ← the upper orchestration layer (local MLX-LM agent)
└── components/                     ← per-class design docs + the V1→V2 coverage map
    ├── README.md
    ├── V1_TO_V2_MAP.md
    └── <33 component docs>
```

## Read in this order

> **Current status (2026-07-21):** branch `jarvis2`. Prototype + single-host Redis science path
> closed: **D0–D12 done** (ledger archive); **D13 closed through D13.10**.
>
> **D13 samplers + gates (done):** D13.1–D13.7 (FeedbackSampler, MCMC/Ensemble, nuisance, Dynesty,
> MultiNest static wrap, diagnostics, review fixes) + **D13.8** flat `hep:feedback`
> (`{uuid, logL}`, `-inf` for unusable) — design
> [`DESIGN_FEEDBACK_RETURN_2.0.md`](DESIGN_FEEDBACK_RETURN_2.0.md) + **D13.9** early YAML
> validation (`task_validation` + `contracts/*`; `Jarvis2 validate`) — design
> [`DESIGN_YAML_VALIDATION_2.0.md`](DESIGN_YAML_VALIDATION_2.0.md) + **D13.10** Nested UX freeze
> (vendored dynesty **3.1.0**; Method=engine — Dynesty always Dynamic, MultiNest always static;
> `Bounds.dynamic` rejected), **AdaptiveBridson** (renames AdaptiveLevelSet, no alias), and
> `Jarvis2 check` smoke layout (`workers=1`, `SAMPLE/test/<uuid>/` flat, pack off; CSV-or-N).
>
> **Earlier post-D13 polish (still landed):** nested production hygiene (Redis-only logL, UUID
> channel, archive catch-up before `samples.csv`, clean nested CSV + MultiNest parity);
> component multi-sink logs under `logs/<scan>/`; process labels `Jarvis-HEP.*`; control
> archive ‰ at DEBUG; Dynesty/MultiNest `Results.summary()` → `sampler.log`; Sampling YAML
> templates in `Jarvis-HEP-v2/jarvishep2/project_template/bin/sampling/`.
>
> **Parked:** **D8 Agent Bridge** (JSON API / `run_state` / agent stop-ack). Human stop still
> does **interrupt → runtime checkpoint** then teardown.
>
> **Code review 2026-07-23** ([`CODE_REVIEW_2026-07-23.md`](CODE_REVIEW_2026-07-23.md)):
> one high finding — after editing a calculator's source the scan silently keeps running
> the previously installed program, and the operator has no control over reinstall —
> plus three medium fixes; filed **D13.11–D13.13**. Accepted fix:
> [`DESIGN_CALC_INSTALL_CONTROL_2.0.md`](DESIGN_CALC_INSTALL_CONTROL_2.0.md)
> (`jarvis_install.json` in the calculator folder + reinstall epoch).
>
> **Strict validation (D17, maintainer request 2026-07-31):** an illegal key or wrong value
> type must log clearly and **exit** — [`DESIGN_STRICT_VALIDATION_2.0.md`](DESIGN_STRICT_VALIDATION_2.0.md).
> Sequencing is forced: **44 of 65 shipped example cards fail validation today**, almost all
> because the schema omits *legitimate* keys (`Calculators.path` ×35 — which the runtime
> explicitly tolerates). Complete the vocabulary (**D17.1**) before flipping strict (**D17.5**),
> with the whole card corpus as the CI gate.
>
> **Next pick:** **D17.1**, then **D14 cluster**
> ([`DESIGN_CLUSTER_EXECUTION_2.0.md`](DESIGN_CLUSTER_EXECUTION_2.0.md), **D14.1**), then
> **D15 reuse/analysis** ([`DESIGN_RESULTS_ANALYSIS_2.0.md`](DESIGN_RESULTS_ANALYSIS_2.0.md)).
>
> **New 2026-07-21 (maintainer feedback):** auto plot YAML must match real physics
> analysis — [`DESIGN_PHYSICS_PLOT_SCENES_2.0.md`](DESIGN_PHYSICS_PLOT_SCENES_2.0.md)
> (D15.5–7); and the YAML complexity barrier gets a **user skills library** —
> [`DESIGN_SKILLS_LIBRARY_2.0.md`](DESIGN_SKILLS_LIBRARY_2.0.md), **v1 shipped under
> [`skills/`](skills/README.md)** (D16.1 done; D16.2 CLI + card CI open). D15.5 and
> D16.2 are parallel-safe with D14.1.
>
> CLI: [components/cli.md](components/cli.md). Broker: [components/redis_queue.md](components/redis_queue.md).
> Ledger: [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md). Project crypto:
> [`components/project_tools.md`](components/project_tools.md) + `Jarvis-HEP-v2/INSTALL.md`.

1. [`DESIGN_2.0_DISTRIBUTED.md`](DESIGN_2.0_DISTRIBUTED.md) — **single source of truth.** The
   full architecture, the three binding decisions (Redis-first, slow-regime-first, worker =
   long-lived process), §0.1 the single-runtime-path rule, §11 what is reused from V1, §12 what
   is retired, §13 the consolidated open questions.
2. [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md) — **the execution playbook (open work only).**
   Prefer **D14** next; **D8 Agent Bridge parked**. Completed ledger history is in `archive/`.
3. [`components/`](components/) — **per-class design docs** (one file per class: Sample,
   RedisQueue, Logger, UMapper, Workflow, Worker, Calculator, Factory, Sampler, Archiver, Core,
   the module/IO backends, the samplers, and the V1-style helper subsystems — CLI, expression
   engine, config/schema, paths/tokens, …). Each fixes code structure, member functions,
   data/schema, concurrency/failure semantics, and tests + verification logic. Start at
   [`components/README.md`](components/README.md); use
   [`components/V1_TO_V2_MAP.md`](components/V1_TO_V2_MAP.md) as the migration checklist.
4. [`DESIGN_AGENT_BRIDGE_2.0.md`](DESIGN_AGENT_BRIDGE_2.0.md) / [`Jarvis-Agent_Design_Document.md`](Jarvis-Agent_Design_Document.md)
   — Agent layer docs **kept for later**; not current execution priority.

## Hard rules (from the plan §1)

- **V2 has one runtime path** — Redis + `spawn` Workers. No `thread` mode, no in-process
  fallback, no thread-based parity oracle.
- One Worker = one Sample at a time; Redis is the sole cross-process resource broker.
- Parity is checked against **captured V1 golden outputs**; CI uses `fakeredis` (no server).
- The frozen behavior contracts (YAML schema, CLI, HDF5/CSV output, `run_summary`, logging) live
  in the **Jarvis-HEP repo** (`docs/specs/`) and must be preserved; every new YAML key is
  optional. *(If you want those contracts mirrored into this tree, say so and I'll copy them.)*

## Out of scope here

V1 design docs, the retired throughput-core V2 attempt, the raw June 2026 discussion notes, and
the frozen `specs/` contracts all live in the **Jarvis-HEP repo** (`docs/v1/`,
`docs/archive/2026-06_v2_throughput_core/`, `docs/specs/`). This `Docs/` tree intentionally keeps
**only the active V2 design + plan + components + the agent layer**.
