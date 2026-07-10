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
├── V2_DISTRIBUTED_PLAN.md          ← execution playbook (milestones D0–D9)
├── DESIGN_PORTAL_IO_2.0.md         ← Jarvis-Portal calculator IO bridge (formats without HEP releases)
├── YAML_REFERENCE_2.0.md           ← as-built task-YAML reference (every key, defaults, gaps)
├── YAML-Example/                   ← public YAML recipes (per method)
│   └── ADAPTIVE_LEVEL_SET.md       ← AdaptiveLevelSet: full Sampling block + key tables
├── CODE_REVIEW_2.0.md              ← functional-completeness review (scope gaps, tests, risks)
├── DEVELOPMENT_REVIEW_2026-07-10.md ← current development status, bugs, priorities, next phases
├── DESIGN_PRINCIPLES_REVIEW_2.0.md ← SOLID/DRY/pattern review + D9 refactor work packages
├── DESIGN_AGENT_BRIDGE_2.0.md      ← V2 ↔ Jarvis-Agent bridge: Agent API + native hep_* tools
├── Jarvis-Agent_Design_Document.md ← the upper orchestration layer (local MLX-LM agent)
└── components/                     ← per-class design docs + the V1→V2 coverage map
    ├── README.md
    ├── V1_TO_V2_MAP.md
    └── <33 component docs>
```

## Read in this order

> **Current status (2026-07-10):** start with
> [`DEVELOPMENT_REVIEW_2026-07-10.md`](DEVELOPMENT_REVIEW_2026-07-10.md). It reviews HEAD
> `63f012f`, records the 236-collected (235 passed, 1 flowchart-export skip) baseline, D8/D9
> progress, current bugs, and the recommended execution order. `CODE_REVIEW_2.0.md` remains the
> historical `d0de31a` baseline.

1. [`DESIGN_2.0_DISTRIBUTED.md`](DESIGN_2.0_DISTRIBUTED.md) — **single source of truth.** The
   full architecture, the three binding decisions (Redis-first, slow-regime-first, worker =
   long-lived process), §0.1 the single-runtime-path rule, §11 what is reused from V1, §12 what
   is retired, §13 the consolidated open questions.
2. [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md) — **the execution playbook.** Milestones
   D0–D7, work packages with file paths, acceptance gates, progress ledger. Start here to pick
   up the next task.
3. [`components/`](components/) — **per-class design docs** (one file per class: Sample,
   RedisQueue, Logger, UMapper, Workflow, Worker, Calculator, Factory, Sampler, Archiver, Core,
   the module/IO backends, the samplers, and the V1-style helper subsystems — CLI, expression
   engine, config/schema, paths/tokens, …). Each fixes code structure, member functions,
   data/schema, concurrency/failure semantics, and tests + verification logic. Start at
   [`components/README.md`](components/README.md); use
   [`components/V1_TO_V2_MAP.md`](components/V1_TO_V2_MAP.md) as the migration checklist.
4. [`DESIGN_AGENT_BRIDGE_2.0.md`](DESIGN_AGENT_BRIDGE_2.0.md) — the **V2 ↔ Jarvis-Agent
   bridge**: the machine-readable Agent API V2 exposes (JSON verbs, `run_state.json`, stop
   contract) and the native `hep_*` tool family on the agent side. Authoritative for the wire
   contract; executed by plan milestone D8 (V2) + M4.5 (Jarvis-Agent repo).
5. [`Jarvis-Agent_Design_Document.md`](Jarvis-Agent_Design_Document.md) — the **upper
   orchestration layer**: a local, macOS/MLX-LM agent that drives V2 (goal understanding, scan
   setup, monitoring). Sits *on top of* the V2 runtime; read the runtime docs first. The
   agent's living design baseline is `Jarvis-Agent/docs/DESIGN.md` in the agent repo.

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
