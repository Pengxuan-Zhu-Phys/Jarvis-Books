# Review — D13 Feedback-Driven Samplers (2026-07-17)

**Reviewed at**: `jarvishep2` branch `jarvis2`, committed through `38d4bd9`
("Wire full official dynesty Bounds API for Dynesty and MultiNest"), plus an
uncommitted working-tree tail (see §0).
**Reviewer**: session review requested by the maintainer after
[`DESIGN_SAMPLERS_2.0.md`](DESIGN_SAMPLERS_2.0.md) (D13) went from design to a
fully-"done" ledger in one working day.
**Verdict**: **D13.1–D13.5b are solid and correctly built** — the porting pattern
this design bet on paid off. **Three findings below should be fixed before D13.6
is trusted as closed**, plus one process-hygiene gap. None of the findings block
using D13 today; they are latent-failure and default-value risks, not crashes
observed in testing.

---

## 0. Process note: ledger says "done", tree says "uncommitted"

[`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md) and
[`DESIGN_SAMPLERS_2.0.md`](DESIGN_SAMPLERS_2.0.md) both mark **D13.6** `done`
2026-07-17. At review time the working tree still had this **uncommitted**:

```
 M jarvishep2/Sampling/dynesty_sampler.py   (sampler_summary.json export wiring)
 M jarvishep2/Sampling/mcmc_sampler.py      (sampler_summary.json + chain_history.csv wiring)
 M jarvishep2/core.py                       (DATABASE/sampler_summary.json mirror)
?? jarvishep2/Sampling/diagnostics_export.py (new module — all of the above depends on it)
?? tests/test_d13_acceptance.py             (the acceptance gate D13.6 claims)
```

D13.1–D13.5b's own test files (`test_feedback_sampler.py`,
`test_mcmc_sampler.py`, `test_ensemble_samplers.py`, `test_nuisance_optimize.py`,
`test_multinest_sampler.py`) **are** committed — only the D13.6 diagnostics tail
is not. The uncommitted files were reviewed in place and pass
(`test_d13_acceptance.py`: 8 passed / 12 subtests) — this is not a correctness
problem, it is a **bookkeeping** one: a ledger row dated and marked `done` should
not describe code that only exists in someone's working tree. Had this tree been
discarded (`git clean`, a bad `git checkout`, a crashed session), the ledger
would have been lying.

**Rule going forward**: a WP is not `done` until its diff is committed. If a
future session finds the ledger ahead of `git log`, it should hold the row at
`in-progress` and commit first, not extend the pattern.

---

## 1. What actually shipped (verified, not just read)

| WP | Committed at | What it is |
|---|---|---|
| D13.1 | `e682310` | `FeedbackSampler` base ([feedback_sampler.py](../../../Jarvis-HEP-v2/jarvishep2/Sampling/feedback_sampler.py)) extracted from AdaptiveLevelSet: generation loop (`propose_generation` → publish batch → drain `hep:feedback` until every uuid seen → `absorb_generation` → `checkpoint_at_barrier`). ALS now subclasses it; its full test suite stayed green — the extraction was behavior-preserving, which was the whole point of doing D13.1 before anything else. |
| D13.2 | `56451f1` | V1's `Sampling/Source/MCMC/` chain runtime (ChainRegistry, multistage state machines, DRAM's delayed-rejection stage machine) copied into `jarvishep2/Sampling/Source/MCMC/` — never imports `jarvishep`. Its futures are now backed by the Redis feedback channel instead of V1's thread pool. Methods `MCMC`/`AMMCMC`/`AM`/`DRAM` registered on the Distributor. Worker-count-independent trajectories are tested. |
| D13.3 | `ae362ca` | Ensemble family (`EnsembleChain`, `DEMCMCChain`) — stretch moves, differential evolution, parallel tempering. Half-ensemble batches parallelize (the design's stated mechanism); PT exchanges happen control-side at the barrier, exactly as designed in §3.1. |
| D13.4 | `0e90da4` | `nuisance_optimize` — the execution-plan step type that was reserved since early D-series but had no implementation. `Module/profile1d.py` (golden-section `Profile1DProfiler`) + `Module/nuisance.py`, wired into `worker.py:_run_nuisance_optimize`. |
| D13.5 | `705a0af` | Vendored dynesty 3.0.0 in-tree (`Sampling/Source/Dynesty/`, MIT-licensed, provenance recorded in `VENDOR.md`/`PATCHES.md`) with a UUID channel patch, plus `RedisEvaluationPool` implementing V1's `JarvisFactoryAsyncPool` pool interface (`map`/`submit`) so dynesty's own sampling internals stay stock. |
| D13.5b | `3ed79e5` | `MultiNestSampler` — confirmed **not** a bug: V1's `Method: MultiNest` was already a thin wrapper around static dynesty `NestedSampler`, never true Fortran MultiNest. V2 keeps that exact contract (§ Non-goals). Worth surfacing to users who might assume otherwise. |
| D13.6 | *uncommitted tail* | `DATABASE/sampler_summary.json` + `chain_history.csv` diagnostics export, YAML_REFERENCE §6.10–6.13, samplers_catalog hierarchy. See §0. |

The central architectural bet — "extract one propose/feedback/barrier pattern
from AdaptiveLevelSet, then port N sampler families onto it" — worked. MCMC,
ensemble, and nested sampling all reused the same base with method-specific
`propose_generation`/`absorb_generation` overrides, and none of them needed to
touch Redis, checkpointing, or watchdog integration directly. That was the
design's core risk and it paid off cleanly.

---

## 2. Findings

### 2.1 `RedisEvaluationPool.map` silently falls back to **local** execution on a dispatch miss (severity: medium-high)

**Where**: [`redis_evaluation_pool.py:74-101`](../../../Jarvis-HEP-v2/jarvishep2/Sampling/redis_evaluation_pool.py#L74)

**What it does**: dynesty reuses `pool.map()` for several different jobs
(prior-transform, loglikelihood, internal proposal evolution). The pool has to
guess which one a given call is, because dynesty's own API doesn't tag the call
site:

```python
remote = self.redis is not None and (
    self._is_loglikelihood_callable(func) or looks_uuid_augmented(first)
)
if remote:
    return self._redis_batch_logl(items)
return [func(x) for x in items]          # ← runs func() in THIS process
```

`_is_loglikelihood_callable` checks `type(func).__name__ == "LogLikelihood"` (the
class dynesty itself wraps user loglikelihoods in — reliable *today*) or a
name-substring match on `"logl"`. `looks_uuid_augmented` inspects whether the
first item's trailing element looks like a uuid string rather than a float.

**Why this is a real risk, not theoretical**: both signals are structural
guesses about dynesty's internals, not a contract dynesty publishes. If a future
dynesty upgrade renames its internal wrapper class, or a code path calls
`pool.map()` with unwrapped loglikelihood callables, the guess fails silently —
and instead of erroring, the pool evaluates the **user's raw loglikelihood
function in the control process**, bypassing every Worker, calculator, and
Operas module. Depending on what that raw callable actually needs, this either
crashes with a confusing error deep inside dynesty, or — worse — succeeds with a
value that did not come from the declared physics pipeline, silently producing
wrong evidence/posteriors.

**Recommendation**: invert the default. When the dispatch signals disagree with
"this must be a physics evaluation" (i.e. anything that is not provably a
prior-transform / `SamplerArgument` call), **raise** rather than fall through to
local execution. A hard, immediate error here is strictly better than a nested
sampling run that quietly used the wrong evaluation path — this is exactly the
kind of failure a physicist would not think to suspect.

### 2.2 Feedback records with an unrecognized uuid are dropped with no log line (severity: low-medium)

**Where**: [`redis_evaluation_pool.py:140`](../../../Jarvis-HEP-v2/jarvishep2/Sampling/redis_evaluation_pool.py#L140)

```python
record = self.redis.pull_feedback(timeout=wait)
...
uuid = str(record.get("uuid", ""))
if uuid not in remaining:
    continue
```

`pull_feedback` is a destructive `BLPOP` — the record is gone from
`hep:feedback` the instant it's popped. If its uuid doesn't match anything this
batch is waiting on, it is discarded with zero trace. In the steady state this
should never fire (each dynesty batch owns its uuids exclusively). If it *does*
fire — a uuid-generation edge case, a resume boundary that double-published a
batch, anything — the only symptom is an unexplained missing result later,
because by the time anyone notices, the evidence that would explain it is
already gone. This is the "hardest bug to ever find" pattern.

**Recommendation**: log a `warning` with the stray uuid and the batch's expected
set size before discarding. One line, zero behavior change, but the difference
between a five-minute log-grep and an unreproducible mystery.

### 2.3 `nuisance_optimize`'s default contradicts what this design promised (severity: needs a decision, not obviously a bug)

**Where**: [`Module/profile1d.py:76,85,110,119`](../../../Jarvis-HEP-v2/jarvishep2/Module/profile1d.py#L76)

```python
re_run_physics: bool = True
...
re_run = block.get("re_run_physics", block.get("rerun_physics", True))
```

[`DESIGN_SAMPLERS_2.0.md` §3.5](DESIGN_SAMPLERS_2.0.md) states the intended
contract in this reviewer's own words when the design was written:
*"Runs Worker-side as an execution-plan step between likelihood terms; **no
calculator re-runs inside the inner loop (V1 contract)**."* The shipped default
does the opposite: **every profiling probe re-runs the full
calculator+Operas pipeline** unless a YAML author explicitly sets
`re_run_physics: false`.

This is not flagged as a hard bug because it *is* configurable and it *is*
tested — it's a **design-vs-implementation mismatch that needs a physics
judgment call**, not a coding error. The two candidate truths:

- If nuisance parameters are pure likelihood-only auxiliaries (the common case —
  e.g. a Gaussian-constrained systematic that never touches the physical
  observables), re-running calculators per probe point is pure waste: for a
  slow-regime external calculator, a profile with 20 probe points now costs 20×
  the calculator wall-clock for a value the calculator never needed.
- If some nuisance parameters *do* feed into the physical calculation (a
  profiled theoretical uncertainty, say), skipping the re-run silently produces
  a wrong profile.

**Recommendation**: check what V1's `nuisance_LogLikelihood.py` /
`nuisance_passCondition.py` actually did — the original V1 modules should
settle which case is the common one. If V1 never re-ran the physics pipeline
per probe, `re_run_physics` should default to `false` and this design's §3.5
claim was correct as written; if V1 did re-run it, this design's §3.5 claim was
wrong and should be corrected, not the code. Either way, document the decision
in `YAML_REFERENCE_2.0.md` next to the `nuisance_optimize` key — right now a
user has no way to discover this knob exists without reading the module source.

### 2.4 ESS (effective sample size) diagnostic is missing (severity: low, closes a stated goal)

**Where**: [`mcmc_sampler.py:793` `_build_summary`](../../../Jarvis-HEP-v2/jarvishep2/Sampling/mcmc_sampler.py#L793)

Design goal 6 promised *"acceptance rate, R-hat, **ESS** per chain"*. R-hat
shipped (`_gelman_rubin_rhat`, line 960); ESS did not — `_build_summary` reports
only per-chain `accepted`/`rejected` counts and the R-hat value. Acceptance rate
alone cannot tell a user whether their chain actually explored the posterior
efficiently (a high acceptance rate can still mean a barely-moving chain).

**Recommendation**: add a per-chain ESS estimate (autocorrelation-time-based is
the standard choice) to `chain_history.csv`/`sampler_summary.json` before D13.6
is considered closed against its own stated goal.

---

## 3. What is genuinely out of scope (confirmed, not overlooked)

- **HMC / NUTS / `tpmcmc` / `rltpmcmc` / `dnn` / `diver`** — zero code exists;
  this matches the design's explicit Non-goals list. Not a gap in this review.
- **D14 (cluster execution)** and **D15 (result reuse / analysis)** — untouched,
  as expected; no `worker start`, `--connect`, `cluster submit`, `analyze`, or
  `--warm-start` anywhere in `client.py`.
- **True Fortran MultiNest** — confirmed non-goal, and confirmed V1 never had it
  either (§1, D13.5b).

---

## 4. Action items

Registered as **D13.7** in [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md):

1. Commit the D13.6 diagnostics tail (§0) — pure hygiene, zero new code.
2. Fail loud instead of falling back to local execution in
   `RedisEvaluationPool.map` (§2.1).
3. Log dropped feedback records instead of silently discarding them (§2.2).
4. Resolve the `re_run_physics` default against V1's actual behavior and
   document the key in YAML_REFERENCE (§2.3) — **needs a maintainer/V1-source
   decision, not just a code change**.
5. Add per-chain ESS to the MCMC diagnostics export (§2.4).

None of these block using D13 today for a straightforward scan; they matter
most for (a) users who hit an edge case dynesty maintainers didn't anticipate,
and (b) anyone relying on `nuisance_optimize` for a slow external calculator,
where the default silently costs 10–50× the intended compute.
