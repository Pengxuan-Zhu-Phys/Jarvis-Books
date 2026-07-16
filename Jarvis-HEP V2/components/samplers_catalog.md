# Component — Samplers catalog (`jarvishep2/Sampling/`)

**Role**: the concrete sampling algorithms. Each proposes `u_coords` (or replays rows), builds
light task dicts, and submits them to Redis via the [sampler base](sampler.md).
**Status**: **As-built** @ `jarvis2` `0a5e85e` (expression/selection cache; EnvReqs.V2 batch_size).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §11;
[sampler.md](sampler.md), [checkpoint.md](checkpoint.md).
**Reuses V1**: none by import.

> **As-built drift (large):** the design catalog listed MCMC (`mcmc_standard`, `tpmcmc`, `dream*`,
> `ess`, …), nested samplers (`dynesty`, `multinest`), and gradient samplers (`mala/hmc/nuts`).
> **None of those exist in V2 yet (D13.2+).** Shipped: **four stateless samplers** (Bridson,
> Random, Grid, CSV) + a seeded test sampler on `CheckpointedSampler`, plus **AdaptiveLevelSet**
> on the new **`FeedbackSampler`** base (D13.1). Porting guide:
> [feedback_sampler.md](feedback_sampler.md).

---

## 1. Class hierarchy

```
SamplingVirtial (sampler.py)               # base: build_sample + Redis submit
 └─ CheckpointedSampler (checkpointed_sampler.py)   # + checkpoint heartbeat + resume
      ├─ RandomS  (randoms.py)        method="Random"
      ├─ Grid     (grid.py)           method="Grid"
      ├─ CSVSampler (csv_sampler.py)  method="CSV"
      ├─ Bridson  (bridson.py)        method="Bridson"
      └─ FeedbackSampler (feedback_sampler.py)   # D13.1: propose → feedback → absorb
           └─ AdaptiveLevelSetSampler (adaptive_level_set.py)  method="AdaptiveLevelSet"
SeededOperaSampler (seeded_sampler.py)     # SamplingVirtial directly (test/acceptance)
```

Each subclass implements the same contract: `set_config` → `propose_next` →
`run_distributed` → `repropose_unfinished` (resume) → `at_safe_barrier` →
`export_runtime_state` / `import_runtime_state`. Helper module `stateless_batch.py` provides
`deterministic_sampler_uuid`, `flush_batch`, `run_stateless_distributed`; `sampling_utils.py`
provides `evaluate_selection` + the `map_*` helpers. Its process-local `ExpressionContext`
lazily compiles and caches selections by `(expression, available variable names)`; candidate
checks then perform only a `CompiledExpression` numerical call.

---

## 2. `RandomS` (`randoms.py`, `method="Random"`)

Uniform random sampler. Config (`Sampling`): `Point number`, optional `selection`, `Seed`.
**Attributes:** `vars`, `_index`, `_accepted_index`, `_maxp`, `_dimensions`, `_selectionexp`,
`_seed`, `_batch_size`, `_uuid_by_accepted_index`, `_u_by_uuid`, `_generator_ready`.
**Key methods:** `set_config`, `initialize` (seed + probe selection), `propose_next` (draw +
selection filter + deterministic uuid), `_uuid_for_accepted_index` (sha256 of
`random:seed:index`), `repropose_unfinished`, `run_distributed` (via `run_stateless_distributed`),
`at_safe_barrier`, `export/import_runtime_state` (carries `numpy_random_state`).

## 3. `Grid` (`grid.py`, `method="Grid"`)

Cartesian grid. Each variable needs `distribution.parameters.num`. Module function
`grid_sampling(dimensions)` builds the `itertools.product` grid in `[0,1]^d`.
**Attributes:** `vars`, `_P` (grid array), `_index`, `_accepted_index`, `_selectionexp`, `_seed`,
`_batch_size`, `_uuid_by_accepted_index`, `_u_by_uuid`. Same method set; uuid prefix `grid`.

## 4. `CSVSampler` (`csv_sampler.py`, `method="CSV"`)

Replay a CSV of pre-computed points. Config (`Sampling.CSV`): `path`, `uuid_column` (default
`uuid`), `variables`, `delimiter`, `encoding`. **Attributes:** `_csv_path`, `_csv_delimiter`,
`_csv_encoding`, `_uuid_column_requested/_resolved`, `_selected_variables`, `_selectionexp`,
`_batch_size`, `_runtime_csv_cursor`, `_runtime_seen_source_uuid`, `_records_exhausted`,
`_params_by_uuid`. **Key methods:** `_iter_records` (streaming reader, cell coercion via
`_coerce_cell`, duplicate-uuid guard, cursor resume), `propose_next` (sets `opera_params` from the
row), `repropose_unfinished`, `run_distributed`, `export/import_runtime_state`.

## 5. `Bridson` (`bridson.py`, `method="Bridson"`)

Poisson-disk / blue-noise sampler (2D–4D). Config (`Sampling`): `Radius`, `MaxAttempt`,
optional `selection`, `Seed`, `MaxWorker`. Module functions: `hypersphere_volume_sample`,
`hypersphere_surface_sample`, `squared_distance`, `Bridson_sampling` (Robert Bridson 2007),
`deterministic_bridson_uuid`. **Attributes:** `vars`, `_P`, `_index`, `_accepted_index`,
`_radius`, `_k`, `_selectionexp`, `_seed`, `_batch_size`, `_max_inflight`, `barinfo`,
`_uuid_by_accepted_index`. **Key methods:** `initialize` (generate the disk set; `info["NSamples"]`),
`propose_next` (+ `_emit_progress`), its own `run_distributed`/`_flush_batch`,
`repropose_unfinished` (+ `_find_row_for_accepted_index`), `export/import_runtime_state` (carries
`grid_points`).

## 6. `SeededOperaSampler` (`seeded_sampler.py`)

Deterministic opera sampler for checkpoint/resume acceptance tests (extends `SamplingVirtial`
directly). Constructor params `seed`/`total_points`/`dimensions`. Uses a master
`np.random.SeedSequence`; `derive_u_coords(index)` spawns a per-Sample child stream
(`derive_sample_seed`). Methods: `propose_next`/`propose_remaining`, `submit_next`/
`submit_all_remaining`, `at_safe_barrier`, `configure_checkpoint`/`persist_runtime_checkpoint`,
`export/import_runtime_state` (serializes the SeedSequence), `repropose_unfinished`. Module
function `deterministic_uuid(*, master, sample_index)`.

---

## 7. Determinism / submission

- Every sampler mints **deterministic uuids** (sha256 of `prefix:seed:index`) so resume + replay
  are stable. `batch_size` (from `EnvReqs.V2.batch_size` / workers) controls Redis pipelining only —
  execution is one Sample per Worker.
- Selection cuts use [`evaluate_selection`](expression.md). A given expression/variable-name
  set is `sympify`/`lambdify` compiled once per control process and reused across candidates;
  `u → x` uses [`Variable`](parameters_variables.md) via the `map_*` helpers.

---

## 8. Interfaces / collaborators

- **Distributor** ([distributor.md](distributor.md)) selects the class by name.
- **sampler base** ([sampler.md](sampler.md)) `_build_sample` / `_submit` / `_submit_group`.
- **checkpoint** ([checkpoint.md](checkpoint.md)) heartbeat + state serialization.
- **core** ([core.md](core.md)) drives `run_distributed` / `repropose_unfinished`.

---

## 9. Tests

`tests/test_samplers_catalog.py`: per-sampler propose/uuid determinism, selection filtering,
selection compile-count caching, distributed submission counts, export/import round-trip. Resume is covered by
`tests/test_distributed_resume.py` (8).

---

## 10. Feedback-driven family + planned additions

- **FeedbackSampler** (D13.1) — shared propose → Redis → `hep:feedback` → absorb base.
  Porting guide: [feedback_sampler.md](feedback_sampler.md).
- **AdaptiveLevelSetSampler** (`method="AdaptiveLevelSet"`, D10, 2 ≤ d ≤ 5) — **shipped** on
  `FeedbackSampler`. Full spec: [adaptive_voronoi_contour.md](adaptive_voronoi_contour.md).
- **MCMC / AM / DRAM / ensemble / dynesty** (D13.2–D13.5) — port V1 science onto
  `FeedbackSampler`; see [`../DESIGN_SAMPLERS_2.0.md`](../DESIGN_SAMPLERS_2.0.md).
