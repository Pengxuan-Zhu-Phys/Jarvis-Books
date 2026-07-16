# Component — FeedbackSampler base (`jarvishep2/Sampling/feedback_sampler.py`)

**Role**: shared base for barrier-synchronized, feedback-driven samplers. Owns the
propose → Redis publish → `hep:feedback` drain → absorb → checkpoint loop so each
method only ports its science hooks.
**Status**: ✅ **Implemented** (D13.1, 2026-07-16). Consumers:
[`AdaptiveLevelSetSampler`](adaptive_voronoi_contour.md); **MCMC / AMMCMC / AM / DRAM**
(D13.2) and **Ensemble / DEMCMC / PT** (D13.3) via `mcmc_sampler.py`. Next: dynesty
pool (D13.5), nuisance step (D13.4).
**Design refs**: [`../DESIGN_SAMPLERS_2.0.md`](../DESIGN_SAMPLERS_2.0.md) §3.2,
[checkpoint.md](checkpoint.md), [redis_queue.md](redis_queue.md),
[sampler.md](sampler.md), [samplers_catalog.md](samplers_catalog.md).
**Maintainer constraint**: **D8 stays parked** — nothing here may depend on agent verbs
or `run_state.json`.

---

## 1. Why this exists

AdaptiveLevelSet (D10) was the first V2 sampler that **cannot** be a stateless batch:
each generation’s proposals depend on the previous generation’s evaluated results.
D13 ports ~25 V1 inference methods (MCMC, ensemble, nested, nuisance) onto the same
pattern. Without a shared base, every method would re-implement:

- pending-uuid bookkeeping and generation barriers,
- `hep:feedback` drain with timeout,
- per-generation `SeedSequence` children (D6.2 determinism),
- batch publish + `submitted_uuids` / `completed_uuids` tracking,
- safe-barrier checkpoint plumbing.

`FeedbackSampler` writes that once. Method science is two hooks.

---

## 2. Inheritance

```
SamplingVirtial          # config / redis / _build_sample / _submit(_group)
 └─ CheckpointedSampler  # heartbeat + persist_runtime_checkpoint + at_safe_barrier
     └─ FeedbackSampler  # pending + drain + seed + propose/absorb loop
          ├─ AdaptiveLevelSetSampler   # custom run_adaptive (cross/refine)
          └─ MCMCSampler               # MCMC/AM/DRAM (D13.2) + Ensemble/DE/PT (D13.3)
```

---

## 3. Propose / absorb contract

```python
class MySampler(FeedbackSampler):
    method = "MyMethod"

    def propose_generation(self) -> Sequence[Sample] | None:
        """Return the next batch of Samples, or None when the run is finished.

        - Assign deterministic uuids *before* return (replays / resume).
        - Empty list: absorb an empty generation without terminating.
        - None: exit run_adaptive.
        """

    def absorb_generation(self, results: Sequence[Mapping[str, Any]]) -> None:
        """Fold barrier feedback into method state.

        Each record has at least: uuid, status ("Completed"|"Failed"), observables.
        Default physics-safe policy for Failed: reject the proposal
        (self._on_failure = "reject"; set "halt" to abort).
        """
```

### 3.1 Default control loop (`run_adaptive`)

```
while True:
    proposals = propose_generation()
    if proposals is None: break
    if proposals:
        _submit_sample_batch(proposals)   # registers pending; batch push
        results = wait_for_generation()   # drain hep:feedback for all pending
    else:
        results = []
    absorb_generation(results)
    checkpoint_at_barrier()
    _generation += 1
```

Core drives this via `Jarvis2Core.run_adaptive_scan` (same path as AdaptiveLevelSet).

### 3.2 Custom control loops

Methods whose stop conditions are not “propose returned None” **override**
`run_adaptive` and call the primitives directly:

| Primitive | Purpose |
|---|---|
| `_init_seed_sequence(seed)` / `_generation_rng(g)` | Deterministic per-generation RNGs |
| `_submit_sample_batch(samples)` | Pending track + batched Redis publish |
| `wait_for_generation(timeout=…)` | Barrier drain → list of feedback dicts |
| `absorb_generation(results)` | Method science after the barrier |
| `checkpoint_at_barrier(reason=…)` | Force resume checkpoint (safe when pending empty) |
| `at_safe_barrier()` | `not _pending_uuids` (heartbeat / D6.2 gate) |

**AdaptiveLevelSet** is the reference custom loop: gen-0 → cross → refine until
converge / max_points / max_generations. It implements `absorb_generation` (fills
`LevelSetPoint.f`) and leaves `propose_generation` as a no-op stub (`return None`)
because its control flow is not a flat propose chain.

---

## 4. Sample identity and failure policy

- **Uuids are method-owned.** The base does not mint uuids. ALS uses
  `deterministic_sampler_uuid(prefix="alevelset", seed, sample_index)`; MCMC
  should hash `(scan seed, chain id, step, stage)` (D13 §3.3).
- **Barrier waits on uuids, not workers.** Watchdog re-queues (D6.1) are transparent.
- **Unrelated feedback** on the channel is ignored (stale or foreign uuids).
- **`_on_failure`**: `"reject"` (default) or `"halt"`. Helpers:
  `_failure_policy_halt(record)`. ALS maps Failed → `f=None` (crossing-conservative)
  regardless; MCMC will reject the proposal and log the failure uuid.

---

## 5. Checkpoint fields (shared)

`_feedback_export_state()` / `_feedback_import_state(state)` cover:

| Field | Meaning |
|---|---|
| `generation` | Current generation index |
| `pending_uuids` | In-flight members of the open generation |
| `seed` | Root seed (SeedSequence rebuilt on import) |
| `submitted_uuids` / `completed_uuids` | From CheckpointedSampler |
| `control_state` | e.g. `repropose_after_resume` |
| `on_failure` | `"reject"` \| `"halt"` |

Subclasses **merge** method-specific state (ALS: `points`, `levelset`, …) on top of
the shared blob. On resume, Core calls `repropose_unfinished` when
`_resume_policy == "resume"` before `run_adaptive`.

---

## 6. Porting checklist (for D13.2+ authors)

1. **Subclass `FeedbackSampler`**, set `method`, register via
   `Distributor.register(..., stateless=False, resume="implemented")`.
2. **Never import from `jarvishep` (V1).** Copy science modules under
   `jarvishep2/Sampling/…` and swap futures for Redis feedback.
3. **Implement `propose_generation` / `absorb_generation`** (or override
   `run_adaptive` + call primitives, ALS-style).
4. **Deterministic uuids** before publish; same uuid on resume re-propose.
5. **Prefer `chains ≥ workers`** for MCMC so the barrier does not idle workers
   (document in YAML_REFERENCE when the method lands).
6. **Accept = V1 card runs unmodified** + diagnostics parity vs a captured golden
   + worker-count independence (1 vs N workers, same trajectory when seeded).
7. **Tests**: unit loop with fakeredis (see `tests/test_feedback_sampler.py`) +
   method-specific golden / resume cases.
8. **D8 remains parked** — no agent stop-ack, no `run_state.json` coupling.

---

## 7. Tests

| Suite | Coverage |
|---|---|
| `tests/test_feedback_sampler.py` | inheritance, deterministic RNG, generic loop, timeout, export/import |
| `tests/test_adaptive_level_set.py` | full ALS regression gate (must stay green after the refactor) |

---

## 8. Non-goals

- Nested sampling / MultiNest / HMC / RL (D13.5+ / later).
- Nuisance step (D13.4).
- Full V1 golden diagnostic closure (D13.6).
- No change to the Archiver / DATABASE contract.
- No D8 / agent surface.

Metropolis + ensemble/PT science **is in scope** and shipped under D13.2–D13.3
(`mcmc_sampler.py`).
