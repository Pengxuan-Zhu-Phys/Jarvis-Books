# Jarvis-HEP V2 — Design & Development Docs

V2-only design home. The V1 line (`Jarvis` 1.7.4, single-process threads) is archived
in the Jarvis-HEP repo and is out of scope here.

**Product names (as shipped):** PyPI `Jarvis-HEP`, import `jarvishep2`, CLI **`Jarvis`**.
Older docs in this tree still say `Jarvis2`; treat that as the same CLI.

## Architecture in one line

```text
Task YAML → sampler → Redis → spawn Workers → Archiver
                                           ├─ DATABASE/samples.hdf5
                                           ├─ SAMPLE/<bucket>/<uuid>/
                                           └─ logs/<scan>/ + run_summary.*
```

One runtime path: Redis + `spawn` Workers. No thread mode.

## How to read this tree

| If you need… | Open |
|---|---|
| Runtime architecture (binding decisions) | [`DESIGN_2.0_DISTRIBUTED.md`](DESIGN_2.0_DISTRIBUTED.md) |
| **Open work** (agent playbook) | [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md) |
| **Structure / layering track (D25)** | [`DESIGN_ARCHITECTURE_HARDENING_2.0.md`](DESIGN_ARCHITECTURE_HARDENING_2.0.md) |
| As-built YAML keys | [`YAML_REFERENCE_2.0.md`](YAML_REFERENCE_2.0.md) |
| `Jarvis man` design | [`DESIGN_YAML_MAN_2.0.md`](DESIGN_YAML_MAN_2.0.md) |
| Per-class as-built notes | [`components/README.md`](components/README.md) |
| User “I want to…” recipes | [`skills/README.md`](skills/README.md) |
| Closed WPs and old reviews | [`archive/README.md`](archive/README.md) |

Do **not** start from a dated `*_REVIEW_*.md`. Those live under
[`archive/reviews/`](archive/reviews/README.md).

## Layout

```
Jarvis-HEP V2/
├── README.md                              ← you are here
├── DESIGN_2.0_DISTRIBUTED.md              ← architecture SSOT
├── V2_DISTRIBUTED_PLAN.md                 ← open work only
├── DESIGN_ARCHITECTURE_HARDENING_2.0.md   ← D25 structure track
├── YAML_REFERENCE_2.0.md                  ← as-built task-card keys
├── DESIGN_*.md                            ← subsystem designs (keep at root;
│                                            HEP code comments point here)
├── YAML-Example/                          ← public YAML recipes
├── components/                            ← as-built per-class notes
├── skills/                                ← user skills (one intent, one card)
└── archive/
    ├── V2_PLAN_ARCHIVE_2026-07-14.md      ← closed D0–D13 ledger extract
    └── reviews/                           ← dated reviews / closeouts
```

Subsystem designs at the root stay there on purpose: `jarvishep2` comments cite
`Jarvis-Books/Jarvis-HEP V2/DESIGN_ADAPTIVE_BRIDSON_LIVE_BAND.md` and friends.
Do not move those files.

### Living subsystem designs (by topic)

| Topic | Doc | Milestone |
|---|---|---|
| Portal I/O + HEP save | [`DESIGN_PORTAL_IO_2.0.md`](DESIGN_PORTAL_IO_2.0.md) | shipped |
| Samplers / feedback | [`DESIGN_SAMPLERS_2.0.md`](DESIGN_SAMPLERS_2.0.md) | D13 shipped |
| MCMC 2.1 | [`DESIGN_MCMC_ARCHITECTURE_2.0.md`](DESIGN_MCMC_ARCHITECTURE_2.0.md) | shipped / unstable Bounds |
| AdaptiveBridson loop | [`DESIGN_ADAPTIVE_BRIDSON_LIVE_BAND.md`](DESIGN_ADAPTIVE_BRIDSON_LIVE_BAND.md) | shipped; D13.12 still open |
| YAML validation | [`DESIGN_YAML_VALIDATION_2.0.md`](DESIGN_YAML_VALIDATION_2.0.md) | D13.9 / D17 shipped |
| Resume | [`DESIGN_RESUME_2.0.md`](DESIGN_RESUME_2.0.md) | D21 shipped |
| Mapper | [`DESIGN_SAMPLING_MAPPER_2.0.md`](DESIGN_SAMPLING_MAPPER_2.0.md) | D22 mostly shipped |
| Calculator modes | [`DESIGN_CALCULATOR_MODES_2.0.md`](DESIGN_CALCULATOR_MODES_2.0.md) | D20 shipped |
| LibDeps | [`DESIGN_LIBDEPS_2.0.md`](DESIGN_LIBDEPS_2.0.md) | D18 shipped |
| `Jarvis man` | [`DESIGN_YAML_MAN_2.0.md`](DESIGN_YAML_MAN_2.0.md) | D24 mostly shipped |
| Cluster | [`DESIGN_CLUSTER_EXECUTION_2.0.md`](DESIGN_CLUSTER_EXECUTION_2.0.md) | **D14 open** |
| Analyze / reuse | [`DESIGN_RESULTS_ANALYSIS_2.0.md`](DESIGN_RESULTS_ANALYSIS_2.0.md) | **D15 open** |
| Physics plot scenes | [`DESIGN_PHYSICS_PLOT_SCENES_2.0.md`](DESIGN_PHYSICS_PLOT_SCENES_2.0.md) | **D15.5–7 open** |
| Skills library | [`DESIGN_SKILLS_LIBRARY_2.0.md`](DESIGN_SKILLS_LIBRARY_2.0.md) | v1 shipped; D16.2+ open |
| Agent bridge | [`DESIGN_AGENT_BRIDGE_2.0.md`](DESIGN_AGENT_BRIDGE_2.0.md) | **D8 parked** |

V1 surface still missing (measured, not transcribed):
[`V1_MIGRATION_STATUS_2026-07-31.md`](V1_MIGRATION_STATUS_2026-07-31.md).

## Current status (2026-08-20)

**Shipped:** D0–D7 runtime, D9 first hardening, D10 AdaptiveBridson, D11 UI,
D12 calculator/UX parity, D13 samplers through nested/MCMC, D17 strict cards,
D18 LibDeps, D19 archiver honesty, D20 multimode, D21 resume, D22 Mapper (except
example-card / schema-vs-runtime leftovers), D23 Operas constants (HEP side),
D24 `Jarvis man` (wheel/cold-start polish still partial).

**Open structure track:** **D25** — catalog SSOT, Core/RedisQueue/Worker splits,
validation import graph. Start at
[`DESIGN_ARCHITECTURE_HARDENING_2.0.md`](DESIGN_ARCHITECTURE_HARDENING_2.0.md).

**Open feature track:** D14 cluster, D15 analyze/plots, D16 skills-in-package.
Do not grow `core.py` or `adaptive_bridson.py` before D25.3 / D13.12.

**Parked:** D8 Agent Bridge.

## Hard rules

- One runtime path: Redis + `spawn` Workers.
- One Worker = one Sample; Redis is the only cross-process broker.
- Parity is against **captured V1 golden outputs**; CI uses `fakeredis`.
- New YAML keys are optional with v1-equivalent defaults unless a WP says otherwise.
- Closed work belongs in `archive/`. The live plan must not accumulate done essays.

## Out of scope here

V1 design, the retired throughput-core attempt, June 2026 discussion notes, and
frozen `specs/` live in the **Jarvis-HEP** repo (`docs/v1/`,
`docs/archive/2026-06_v2_throughput_core/`, `docs/specs/`).
TeX user manuals live in `Jarvis-Books/TeX/`, not in this folder.
