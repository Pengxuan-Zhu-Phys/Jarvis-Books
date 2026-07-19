# Component — Samplers catalog (`jarvishep2/Sampling/`)

**Role**: the concrete sampling algorithms. Each proposes `u_coords` (or replays rows), builds
light task dicts, and submits them to Redis via the [sampler base](sampler.md).
**Status**: **As-built** @ D13.6 (stateless + ALS + MCMC/ensemble + Dynesty/MultiNest + diagnostics).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §11;
[`../DESIGN_SAMPLERS_2.0.md`](../DESIGN_SAMPLERS_2.0.md); [sampler.md](sampler.md),
[checkpoint.md](checkpoint.md), [feedback_sampler.md](feedback_sampler.md).
**Reuses V1**: none by import (engines copied under `Sampling/Source/`, not imported).

> **Out of scope (still):** RobustAM, DREAM*, Slice/ESS, MALA/HMC/NUTS, RLTPMCMC, DNN, Diver,
> Fortran MultiNest. Nested YAML surface is official dynesty 3.x (see YAML_REFERENCE §6.10).

---

## 1. Class hierarchy

```
SamplingVirtial (sampler.py)               # base: build_sample + Redis submit
 └─ CheckpointedSampler (checkpointed_sampler.py)   # + checkpoint heartbeat + resume
      ├─ RandomS  (randoms.py)        method="Random"
      ├─ Grid     (grid.py)           method="Grid"
      ├─ CSVSampler (csv_sampler.py)  method="CSV"
      ├─ Bridson  (bridson.py)        method="Bridson"
      └─ FeedbackSampler (feedback_sampler.py)   # propose → hep:feedback → absorb
           ├─ AdaptiveLevelSetSampler (adaptive_level_set.py)
           ├─ MCMCSampler (mcmc_sampler.py)  # MCMC/AM/DRAM + Ensemble/DE/PT
           ├─ DynestySampler (dynesty_sampler.py)
           └─ MultiNestSampler (multinest_sampler.py)  # static NestedSampler
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
- **MCMC / AMMCMC / AM / DRAM** (D13.2) — **shipped** on `FeedbackSampler`
  (`mcmc_sampler.py` + `Sampling/Source/MCMC/` engines). V1 Bounds surface
  (`num_chains`/`chains`, `num_iters`/`steps`, `proposal_scale`, adapt/dr keys).
  Tests: `tests/test_mcmc_sampler.py` (worker-count independence, Core DRAM e2e).
- **Ensemble / DEMCMC / PT** (D13.3) — **shipped**: stretch (`EnsembleMCMC`/`Ensemble`),
  DE (`DEMCMC`), parallel tempering (`PTMCMC`/`PT`/`PTEnsemble`). Half-ensemble
  barriers; control-side temperature swaps. Tests: `tests/test_ensemble_samplers.py`.
- **Dynesty** (D13.5) — **shipped** (`dynesty_sampler.py` + vendored
  `Sampling/Source/Dynesty` 3.0.0 with UUID channel + RedisEvaluationPool).
  **YAML = official dynesty 3.x API**: `Bounds.sampler` / flat constructor keys
  (`bound`, `sample`, `walks`, …) + `Bounds.run_nested` (static `dlogz` /
  dynamic `dlogz_init` alias map). Default DynamicNestedSampler.
  Post-run: `DATABASE/dynesty_result.csv` + Jarvis-PLOT `dynesty_runplot`.
  Tests: `tests/test_dynesty_sampler.py`, `tests/test_nested_yaml_kwargs.py`.
- **MultiNest** — **shipped** as **static** NestedSampler wrapper
  (`multinest_sampler.py`). Same full constructor/run_nested pass-through;
  always static (ignores `dynamic: true`). CSV: `multinest_result.csv` + same
  jplot card. Tests: `tests/test_multinest_sampler.py`.
