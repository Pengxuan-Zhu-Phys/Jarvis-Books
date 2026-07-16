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
├── DESIGN_SAMPLE_COORDINATION_2.0.md ← **accepted**: uuid key; task=u_coords; feedback=logL; stage opt-in
├── DESIGN_SAMPLE_PROGRESS_MONITOR.md ← monitor stage-on-heartbeat (aligned with coordination design)
├── YAML_REFERENCE_2.0.md           ← as-built task-YAML reference (every key, defaults, gaps)
├── YAML-Example/                   ← public YAML recipes (per method)
│   └── ADAPTIVE_LEVEL_SET.md       ← AdaptiveLevelSet: full Sampling block + key tables
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

> **Current status (2026-07-16):** code baseline `jarvis2` (see recent commits: D9.4/9.6, D10.4/10.5,
> interrupt checkpoint; working tree may have benchmark JSON / `checkpoints/` dirt).
> Prototype closed; **D11 + D12 done**. Post-D12 polish: version/ps/kill/logging flags, logs layout,
> FileOperation, CSV + scan perf, D1.1 `op_count` contract.
> **D10 + D9.4/9.6 done.** **D8 Agent Bridge parked** (JSON API / run_state / agent stop-ack).
> Human stop path includes **interrupt → runtime checkpoint** then teardown.
> Open: maintainer-directed polish only until D8 is unparked.
> CLI: [components/cli.md](components/cli.md). Broker contract: [components/redis_queue.md](components/redis_queue.md).
> Ledger: [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md). Project crypto:
> [`components/project_tools.md`](components/project_tools.md) + `Jarvis-HEP-v2/INSTALL.md`.

1. [`DESIGN_2.0_DISTRIBUTED.md`](DESIGN_2.0_DISTRIBUTED.md) — **single source of truth.** The
   full architecture, the three binding decisions (Redis-first, slow-regime-first, worker =
   long-lived process), §0.1 the single-runtime-path rule, §11 what is reused from V1, §12 what
   is retired, §13 the consolidated open questions.
2. [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md) — **the execution playbook (open work only).**
   Prefer **D11 / D12**; **D8 Agent Bridge parked**. Completed ledger history is in `archive/`.
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
