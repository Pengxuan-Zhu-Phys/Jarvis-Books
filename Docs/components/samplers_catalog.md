# Component — Samplers catalog & MCMC kernel (`jarvishep2/Sampling/`)

**Role**: the concrete sampling algorithms and the shared MCMC state-machine kernel. **All are
reused** in V2 — they propose `u_coords` exactly as in V1; only the submission path changes
(`redis.push_task` instead of `factory.submit_task`, via the [sampler base](sampler.md)).
**Status**: design — reused; the only V2 work is the base-class submission binding + result
feedback for MCMC.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §11;
`../../v1/design/MCMC_STATE_MACHINE_DESIGN.md`, `../../v1/design/RLTPMCMC_SAMPLER_DESIGN.md`;
[sampler.md](sampler.md) (base), [checkpoint.md](checkpoint.md).
**Reuses V1**: every `Sampling/*` sampler + `Sampling/Source/MCMC/*` kernel.

---

## 1. What is reused (unchanged proposal logic)

| Family | Modules | V2 status |
|--------|---------|-----------|
| Stateless | `randoms.py`, `grid.py`, `bridson.py`, `csv_sampler.py` | reused — emit batches of `u_coords` via `_submit_group` |
| Nested | `dynesty.py`, `multinest.py`, `nested_checkpoint_bridge.py` | reused — ready-queue drain → group submit |
| Optimizer | `diver.py` | reused |
| MCMC family | `mcmc_standard.py` (`MCMC`/`ToyMCMC`), `tpmcmc.py`, `ammcmc.py`, `robustam.py`, `dram.py`, `dream.py`, `dream_lite.py`, `demcmc.py`, `ensemblemcmc.py`, `pt_ensemble.py`, `slicemcmc.py`, `ess.py` | reused — drain ready chains → group; result feedback by `uuid` |
| Gradient | `mala.py`, `hmc.py`, `nuts.py` | reused — engines are **placeholders** (`gradient_contract_level=placeholder`); carry as-is |
| Experimental | `rltpmcmc.py`, `rl_sampler_base.py`, `dnn.py` | reused (DNN/RL keep their V1 behavior + bug fixes) |

The MCMC kernel (`Source/MCMC/`: `state_machine_base`, `state_machine_multistage_base`,
`controller`, `engine_*`, `*_chain`, `metrics_bus`, `runtime_checkpoint`) is the shared runtime —
**reused intact**; its invariants are governed by `MCMC_STATE_MACHINE_DESIGN.md`.

---

## 2. The only V2 changes

1. **Submission**: all samplers submit through the [sampler base](sampler.md) `_submit`/
   `_submit_group` → Redis, instead of `factory.submit_task`. Proposal logic is untouched.
2. **Result feedback (MCMC)**: samplers that need a proposal's LogL before the next proposal
   consume the result stream keyed by `uuid`; acceptance bookkeeping + state machine unchanged.
3. **No materialization** in the sampler (the Worker maps `u→x` and materializes).
4. **RNG**: master `SeedSequence` + child streams for Worker-count-independent reproducibility
   ([checkpoint.md](checkpoint.md)).

Nothing in the per-algorithm math changes — that is the basis of golden parity.

---

## 3. Bug fixes to carry (invariant #14, when these files are touched)

| File | Fix |
|------|-----|
| `sampler.py:447` | `ax.axes("off")` → `ax.axis("off")` |
| `bridson.py:397` | `np.sum(..., axis=ndim)` → `axis=-1` |
| `dnn.py:244-246` | `Classifier.save()` uses uninitialized `self.path` |
| `dnn.py:556` | `train_model(..., self.dataset.valid)` likely wrong target |
| `sample_archive.py:179-180` | non-atomic check-then-add race (also [datarecorder.md](datarecorder.md)) |
| spelling | `SamplingVirtial` kept (do **not** rename, invariant #12); `"initializaing"` → `"initializing"` log strings |

---

## 4. Concurrency / determinism / failure semantics

- Samplers run in the **control process** (proposal + checkpoint); execution is the Worker's.
- Batch proposers (`Random/Grid/Bridson`) emit `_submit_group` (one Redis pipeline); MCMC drains
  ready chains into a group per iteration. `batch_size` affects **submission pipelining only**, not
  execution (one Sample per Worker).
- Seeded `u_coords` sequences are **identical to V1** (parity); the MCMC state machine round-trips
  through checkpoints unchanged.
- Backpressure: the base throttles submission on `hep:task_queue` high-water.

---

## 5. Interfaces

- **sampler base** (`SamplingVirtial`) → `_submit`/`_submit_group`, result feedback, checkpoint.
- **distributor** → selects the sampler class by name (`ToyMCMC`→`MCMC`, `DREAM-lite`, etc.).
- **RedisQueue** → task submit + (MCMC) result stream.
- **checkpoint** → `export/import_runtime_state`, `SeedSequence`.

---

## 6. Tests (`tests/test_samplers_catalog.py`)

Unit (per family, `fakeredis`):
1. **Seeded parity** — `u_coords` sequences byte-identical to V1 goldens for Random/Grid/Bridson
   and a representative MCMC sampler.
2. **Group submission** — batch proposers emit one pipeline of the right size; order preserved
   (Bridson).
3. **MCMC feedback** — result stream drives the same acceptance as the V1 fixture; checkpoint
   round-trip green.
4. **Distributor dispatch** — every registered name (incl. `ToyMCMC`, `DREAM-lite`) resolves to
   the right class.
5. **Carried bug fixes** — regression tests for `bridson.py:397`, `sampler.py:447`, the `dnn.py`
   issues (where touched).

Verification logic: test 1/3 keep the science identical across all families (the whole reuse
premise); test 5 lands the known sampler bug fixes.

---

## 7. Open questions

- Real gradient engines (`mala/hmc/nuts`) — implement beyond placeholders (separate track).
- Tight-MCMC result-feedback latency over Redis vs. a dedicated low-latency channel
  ([sampler.md](sampler.md) §8).
