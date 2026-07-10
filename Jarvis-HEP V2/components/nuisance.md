# Component — Nuisance handling (`jarvishep2/Module/nuisance_*.py`, `jarvishep2/Sampling/nuisance_sampler.py`, `Source/Nuisance/profile1d.py`)

**Role**: nuisance-parameter support — pass-condition gating, nuisance log-likelihood terms, and
the nuisance optimization/profiling loop. In V2 the **nuisance loop runs inside the Worker** (the
Sampler proposes only interest parameters).
**Status**: ⚠️ **STUBBED ONLY — NOT BUILT** (as-built @ `jarvis2` `d0de31a`).

> **As-built reality:** there is **no `Module/nuisance_*.py`, no `nuisance_sampler.py`, and no
> `profile1d.py`**. Nuisance support is limited to two near-no-op hooks on
> [`Sample`](sample.md) — `combine_nuisance_card()` (merges a `nuisance.active.param` block into
> params when an `info["nuisance"]` card is present) and `gather_nuisance()` — plus the
> `nuisance_optimize` value in `VALID_EXECUTION_STEP_TYPES`. **No nuisance optimization loop, no
> nuisance likelihood registry, and no nuisance sampler ship.** The design below is retained for
> history.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §13.4;
discussion `worker_design.md` §7; [worker.md](worker.md), [likelihood.md](likelihood.md).
**Reuses V1**: `Module/nuisance_LogLikelihood.py` (`NuisanceExpressionRegistry`),
`nuisance_passCondition.py`, `Sampling/nuisance_sampler.py`, `Source/Nuisance/profile1d.py`.

---

## 1. Responsibilities

1. **Pass-condition**: gate whether a Sample's observables satisfy nuisance constraints
   (`nuisance_check`).
2. **Nuisance LogL terms**: compiled expression registry (`NuisanceExpressionRegistry`) adding
   nuisance contributions to the total LogL.
3. **Nuisance optimization loop**: profile/optimize nuisance parameters for a fixed interest point
   — in V2 this loop lives **in the Worker** (it may call calculators multiple times for one
   interest Sample).
4. The Sampler proposes **interest parameters only**; nuisance is the Worker's concern.

---

## 2. Structure (reused + V2 loop)

```python
class NuisanceExpressionRegistry:                  # reused
    def set_config(self, name, expression): ...
    def deps(self) -> tuple[str, ...]: ...
    def can_eval(self, available_keys): ...
    def eval(self, values): ...                    # via the expression engine

# V2: the nuisance loop as a Worker ExecutionStep type
class NuisanceProfiler:
    def __init__(self, registry, pass_condition, strategy="profile_likelihood"): ...
    def optimize(self, sample, worker) -> dict: ...   # returns best nuisance + observables
```

`ExecutionStep(type="nuisance_optimize", strategy=…)` (reserved in [sample.md](sample.md)/[workflow.md](workflow.md))
is where the Worker invokes the profiler.

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `NuisanceExpressionRegistry.set_config` | `(name, expr)` | Register a nuisance LogL expression. |
| `NuisanceExpressionRegistry.eval` | `(values) -> float` | Evaluate the nuisance term (expression engine). |
| `nuisance_check` | `(observables, sample_info) -> bool` | Pass-condition gate (moved onto the Worker). |
| `NuisanceProfiler.optimize` | `(sample, worker) -> dict` | Run the nuisance optimization loop for a fixed interest point — may call calculators repeatedly; return best nuisance + observables + LogL. |

---

## 4. Worker nuisance loop (the new capability)

```
Worker.process_task (interest Sample):
   run interest workflow → observables
   if workflow has nuisance_optimize step:
       best = NuisanceProfiler.optimize(sample, self)      # inner loop: vary nuisance, re-run calc, eval LogL
       sample.info["observables"].update(best.observables)
   LogL = interest_LogL + nuisance_LogL terms
```

The inner loop is **entirely Worker-local**: it can call the Worker's held calculators (acquiring
slots) many times for one interest point, then reports the profiled result. The Sampler never sees
nuisance parameters.

---

## 5. Concurrency / determinism / failure semantics

- The nuisance loop is **within one Sample** on one Worker → respects "one Sample per Worker"
  (invariant #6); it just does more work per Sample.
- Calculator calls inside the loop go through the same Redis free-pool (global cap preserved).
- Nuisance expressions are compiled once (per Worker) via the [expression engine](expression.md);
  evaluation is deterministic (parity).
- A failed pass-condition rejects the Sample (logged); a failed inner calculator fails the Sample
  with its log.

---

## 6. Interfaces

- **Worker** → invokes `NuisanceProfiler.optimize` on a `nuisance_optimize` step; calls
  `nuisance_check`.
- **Parameters** → `add_nuisance` declares nuisance params.
- **expression engine** → nuisance LogL term evaluation.
- **likelihood** → total LogL = interest + nuisance terms.

---

## 7. Tests (`tests/test_nuisance.py`)

Unit:
1. **Registry parity** — `NuisanceExpressionRegistry.eval` matches V1 for the same values
   (golden).
2. **Pass-condition** — `nuisance_check` accepts/rejects correctly; rejection logged.
3. **Worker loop** — a fixed interest point with a `nuisance_optimize` step profiles nuisance,
   calling the calculator multiple times, and returns the best LogL (deterministic on a seeded
   fixture).
4. **One-Sample invariant** — the nuisance loop stays within one Worker/one Sample (no extra
   inflight Samples).
5. **Slot reuse** — repeated calculator calls in the loop acquire/release the free-pool correctly
   (no leak).

Verification logic: test 3 is the new-capability gate (profiling inside the Worker); test 4 keeps
the execution-model invariant intact.

---

## 8. Open questions

- Optimization strategy surface (profile-likelihood vs. marginalization vs. fixed grid) — start
  with `profile_likelihood`, make `strategy` extensible.
- Whether nuisance results need their own checkpoint state (default: derived per interest Sample,
  not separately checkpointed).
