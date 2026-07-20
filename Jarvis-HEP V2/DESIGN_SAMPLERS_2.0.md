# DESIGN — Feedback-Driven Samplers: MCMC / Nested / Nuisance (V2, D13)

**Status**: design accepted 2026-07-16; **D13.1–D13.7 closed** (samplers + nuisance +
Dynesty/MultiNest + diagnostics + review fixes). **Review**:
[`D13_SAMPLERS_REVIEW_2026-07-17.md`](D13_SAMPLERS_REVIEW_2026-07-17.md) — architecture
verdict positive; §2 findings fixed in D13.7 (fail-loud pool dispatch, feedback-drop
logging, `re_run_physics` V1-true default, MCMC ESS).
**Date**: 2026-07-16 (from the post-D12 capability review in this repo's session notes)
**Scope**: migrate the V1 sampler science (MCMC family, nested sampling, nuisance
profiling) onto the V2 distributed execution model. **V1 `Sampling` YAML surface is
preserved verbatim** — a V1 card with `Method: DRAM` must run unmodified.
**Maintainer constraint**: **D8 Agent Bridge stays parked** — nothing here may depend on
D8 verbs, `run_state.json`, or agent stop-ack.

---

## 1. Problem

V2 ships 5 samplers (Bridson, Random, Grid, CSV, AdaptiveLevelSet). V1 ships 25+,
including the entire inference stack users actually need for physics results:

| Family | V1 modules (frozen reference) |
|---|---|
| MCMC core | `mcmc_standard`, `ammcmc`, `robustam`, `dram`, `mala`, `slicemcmc`, `ess` |
| Ensemble | `ensemblemcmc`, `demcmc`, `dream`, `dream_lite`, `pt_ensemble` |
| Nested | `dynesty`, `multinest`, `nested_checkpoint_bridge` |
| Gradient/advanced | `hmc`, `nuts`, `tpmcmc`, `rltpmcmc`, `dnn`, `diver` |
| Nuisance | `nuisance_sampler`, `Module/nuisance_LogLikelihood`, `Module/nuisance_passCondition` |

Without MCMC/nested, V2 can map functions but cannot produce posteriors or evidence —
the runtime is production-grade but the science surface is not.

## 2. Goals

1. **V1 YAML parity**: `Sampling.Method: <name>` + `Sampling.Bounds` keys (incl. V1
   aliases like `dr.steps`) work unmodified; unknown methods keep failing with the
   supported-method list (Distributor registry contract).
2. **One porting pattern, applied N times**: extract the propose → Redis → feedback →
   barrier loop that AdaptiveLevelSet (D10) already implements into a reusable
   `FeedbackSampler` base, then port method by method.
3. **Reuse V1 chain science, replace only transport**: V1's `Sampling/Source/MCMC/`
   package (ChainRegistry / ChainRuntime / multistage state machines / chain engines) is
   already **future-driven** — chain logic is decoupled from evaluation. Copy it into
   `jarvishep2/Sampling/Source/MCMC/` (never import from `jarvishep`) and back the
   futures with the Redis feedback channel instead of the V1 thread pool.
4. **Determinism**: worker-count-independent trajectories via per-chain `SeedSequence`
   spawning (same discipline as D6.2/D10).
5. **Checkpoint/resume at generation barriers** through the existing
   `runtime_checkpoint` payloads; V1's `Source/MCMC/runtime_checkpoint.py` maps onto it.
6. **Honest diagnostics**: acceptance rate, R-hat, ESS per chain in `run_summary` and
   the sampler log; burn-in/thinning recorded per record (`chain_id`, `step`,
   `accepted`, `weight` columns in DATABASE) — never silently dropped rows.

### Non-goals

- RL/DNN/differential-evolution samplers (`rltpmcmc`, `dnn`, `diver`) — later milestone.
- Fortran MultiNest / pymultinest — V1 never used it; V1/V2 ``Method: MultiNest``
  is static dynesty NestedSampler (shipped under D13.5b). True Fortran MultiNest remains out of scope.
- Changing the Archiver/DATABASE output contract (new columns are additive).
- Any D8/agent-facing control surface.

## 3. Execution model

### 3.1 The mapping problem

MCMC is sequential **per chain**; V2 parallelism is **across Samples**. The mapping:

- **Parallel unit = chain.** Run `n_chains ≥ workers` chains; one generation proposes
  one point per chain → one Redis batch → Workers evaluate (calculator/Operas/LogL as
  usual) → feedback drained at the barrier → per-chain accept/reject → next generation.
- **DRAM stage 2**: rejected chains re-propose within the same generation as a
  follow-up mini-batch (V1's multistage state machine already models this —
  `_future_to_stage_idx`). A generation closes only when every chain settled a stage.
- **Ensemble moves** (stretch/DE): half-ensemble updates parallelize naturally; the
  complementary half provides the move vectors, matching emcee's parallel semantics.
- **PT ensemble**: temperature swaps happen control-side at the barrier; each replica
  is an ordinary chain to the runtime.

### 3.2 `FeedbackSampler` base (extracted from AdaptiveLevelSet)

```
class FeedbackSampler(CheckpointedSampler):
    generation loop:
        proposals = self.propose_generation()          # method-specific
        publish batch (Sample uuids ← deterministic per chain/step)
        drain hep:feedback until all uuids seen        # timeout → failure policy
        self.absorb_generation(results)                # method-specific
        checkpoint_at_barrier()
```

AdaptiveLevelSet is refactored to subclass it (behavior-preserving; its tests are the
regression gate). Everything below the barrier loop (feedback drain bookkeeping,
uuid↔chain maps, seed spawning, checkpoint plumbing) lives once in the base.

### 3.3 Sample identity and failure policy

- Sample uuid = deterministic hash of (scan seed, chain id, step, stage) — replays and
  resume re-propose identical points (D6.2 discipline).
- A Failed sample at a barrier: the chain **rejects** that proposal and logs the
  failure uuid (physics-safe default; configurable `on_failure: reject|halt`).
- Watchdog re-queue (D6.1) stays transparent: the barrier waits on uuids, not workers.

### 3.4 Nested sampling (dynesty)

V1 already runs dynesty through an injected pool (`JarvisFactoryAsyncPool`). V2 ships
`RedisEvaluationPool` implementing the same pool interface (`map` / async submit):
dynesty's internals stay stock; every `map` batch becomes a Redis task batch + feedback
drain. Checkpointing wraps dynesty's own `save`/`restore` inside the V2 runtime
checkpoint (V1's `nested_checkpoint_bridge` is the reference). Evidence (`logZ ± err`)
lands in `run_summary` and the DATABASE attrs.

### 3.5 `nuisance_optimize` execution step

The step type is already reserved in `_VALID_EXECUTION_STEP_TYPES` but has no
implementation. Port V1 semantics: per-sample profiling over declared nuisance
variables using the expression-based `NuisanceExpressionRegistry`
(V1 `Module/nuisance_LogLikelihood.py`, compile-once — map onto the shared V2
`ExpressionContext`), plus `nuisance_passCondition` gating. Runs Worker-side as an
execution-plan step between likelihood terms. **Re-run policy (D13.7c):** V1
re-executed each Profile1D probe as a full sample (`NAttempt` cards), so the
V2 default is `re_run_physics: true` (calc+opera per probe). Set `false` for
pure expression-only nuisances. Expression registries still compile once (no
recompile in the inner loop).

## 4. YAML surface (unchanged)

```yaml
Sampling:
  Method: DRAM                # V1 names: MCMC / AM / DRAM / Ensemble / PT / Dynesty …
  Variables: [...]            # unchanged
  Bounds:                     # V1 per-method keys incl. aliases (dr.steps, …)
    chains: 16
    steps: 5000
    dr_steps: 2
    dr_scale_factors: [1.0, 0.5]
  LogLikelihood: [...]        # unchanged
```

`EnvReqs.V2` untouched. New optional keys are additive with V1-equivalent defaults
(hard invariant 1).

## 5. Work packages

| WP | Title | Depends on | Accept |
|---|---|---|---|
| D13.1 | `FeedbackSampler` base extracted from ALS + porting guide (`components/feedback_sampler.md`) | — | **done** 2026-07-16: ALS on base; `test_adaptive_level_set` + `test_feedback_sampler` green; guide documents propose/absorb |
| D13.2 | `Source/MCMC/` chain runtime port + `MCMC`/`AM`/`DRAM` methods | D13.1 | **done** 2026-07-16: engines + `MCMCSampler` on FeedbackSampler; Distributor methods MCMC/AMMCMC/AM/DRAM; worker-count independence tested; feedback loop e2e with mock LogL. Full V1 eggbox golden diag comparison remains progressive under D13.6. |
| D13.3 | Ensemble family: `EnsembleMCMC` (stretch), `DEMCMC`, `PT` | D13.2 | **done** 2026-07-17: half-ensemble barriers + PT exchange; methods EnsembleMCMC/Ensemble/DEMCMC/PTMCMC/PT/PTEnsemble; worker-count trajectory test. Golden moments / wall-clock gate progressive under D13.6. |
| D13.4 | `nuisance_optimize` Worker step + pass-condition | — (parallel to D13.2) | **done** 2026-07-17: Profile1D on Worker; ExpressionContext registries; flowchart + sample log; tests green. Full HinoLLP golden progressive under D13.6. |
| D13.5 | Dynesty bridge (`RedisEvaluationPool`) + checkpoint wrap | D13.1 | **done** 2026-07-17: vendored dynesty 3.0.0 + UUID channel; RedisEvaluationPool; DynestySampler; `DATABASE/dynesty_result.csv` + Jarvis-PLOT `dynesty_runplot` jplot (V1 path, no `dynesty.plotting`). Full card golden / native resume progressive under D13.6. |
| D13.6 | Acceptance & docs closure | D13.2–5 | **done** 2026-07-20: `diagnostics_export.py` + acceptance tests + MCMC/nested wiring committed (`f10889b`); DATABASE `sampler_summary.json` / `chain_history.csv` contract live. |
| D13.7 | Review fixes (see `D13_SAMPLERS_REVIEW_2026-07-17.md` §2) | D13.6 | **done** 2026-07-20: (a) pool map fail-loud on ambiguous dispatch; (b) unmatched feedback warning; (c) `re_run_physics` default **true** (V1 NAttempt parity) + YAML_REFERENCE §6.13; (d) per-chain `ess_logl` / `ess_logl_mean` in MCMC summary. |

**Rollback**: unregistered methods keep erroring with the supported list — each WP only
*adds* Distributor registrations. **Out of scope**: multinest, HMC/NUTS gradients,
RL/DNN/diver, any D8 surface.

## 6. Risks

1. **Throughput inversion**: chains < workers idles workers — document
   `chains ≥ workers` guidance; PT/ensemble raise effective parallelism.
2. **Barrier stragglers**: one slow calculator stalls a generation — same exposure as
   D10; mitigation is the existing watchdog + `on_failure: reject`.
3. **V1 numerical drift**: chain engines copied, not rewritten; goldens compare
   *distributions/diagnostics*, not bitwise trajectories (transport differs by design).
