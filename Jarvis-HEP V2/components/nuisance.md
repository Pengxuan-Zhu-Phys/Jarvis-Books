# Component — Nuisance handling (`jarvishep2/Module/nuisance.py`, `profile1d.py`)

**Role**: nuisance-parameter profiling and pass-condition gating on the **Worker**.
The sampler proposes interest parameters only; the `nuisance_optimize` execution-plan
step runs Profile1D (golden-section) over a free nuisance variable, evaluates
expression LogL / pass conditions (compile-once via shared `ExpressionContext`),
and merges the best nuisance state into the Sample.
**Status**: ✅ **Implemented** (D13.4, 2026-07-17) — Profile1D path.
**Design refs**: [`../DESIGN_SAMPLERS_2.0.md`](../DESIGN_SAMPLERS_2.0.md) §3.5,
[worker.md](worker.md), [likelihood.md](likelihood.md), [expression.md](expression.md),
[flowchart.md](../components/README.md).
**Reuses V1 science (no import)**: `NuisanceExpressionRegistry`, `NuisancePassCondition`,
`Profile1D` golden-section from `Sampling/Source/Nuisance/profile1d.py`.

---

## 1. YAML surface

Under `Sampling.Nuisance` (or top-level `Nuisance`):

```yaml
Sampling:
  Nuisance:
    Method: Profile1D
    Variables:
      - name: ratio
        distribution: {type: Flat, parameters: {min: -3, max: -2}}
    LogLikelihood:
      - {name: "LogLNP_wN2", expression: "Abs(log(WN2) + 40.3)"}
    TargetMode: min          # or max
    MaxAttempt: 100
    re_run_physics: true     # default true (V1 parity); false = expression-only
    PassCondition:
      - {name: "N1LSP", expression: "Abs(mN1) < Abs(mC1)"}
```

When present, Core inserts `ExecutionStep(type=nuisance_optimize)` **before**
likelihood in the Worker plan and stamps `nuisance_config` into the Worker blueprint.

---

## 2. Modules

| Module | Role |
|--------|------|
| `Module/nuisance.py` | `NuisanceExpressionRegistry`, `NuisancePassConditionRegistry`, config helpers |
| `Module/profile1d.py` | `Profile1DProfiler.optimize(evaluate)` golden-section loop |
| `worker.py` | `_run_nuisance_optimize`, optional `_rerun_physics_pipeline` |
| `workflow.py` | `include_nuisance` on `build_execution_plan` / template |
| `flowchart.py` | draws `NuisanceOptimize` when config has Nuisance |

---

## 3. Worker loop

```
for layer in execution_plan:
  calculators / operas as usual
  if step.type == nuisance_optimize:
      for probe z in Profile1D:
          sample.params[var] = z
          if re_run_physics: re-run calc + opera layers
          eval nuisance LogL terms + pass conditions (compiled once)
      merge best z, NuisanceLogL, pass flag into observables/info["nuisance"]
      if not pass: sample.status = Failed
  likelihood …
```

Expression evaluation does **not** recompile per probe. **`re_run_physics`
defaults to `true`** (D13.7c): V1 Profile1D re-ran the full sample pipeline per
`NAttempt` probe; V2 keeps that cost model so nuisances that feed calculators
(HinoLLP-style `ratio`) stay correct. Set `false` for pure expression-only
objectives that never touch calculators/Operas.

---

## 4. Outputs

- `observables[var]`, `observables["NuisanceLogL"]`, `observables["nuisance_pass"]`
- Named nuisance LogL terms written into observables
- `info["nuisance"]`: best, attempts, history, pass_terms, status
- Sample logger lines under `(Nuisance:Profile1D)`
- Flowchart node `NuisanceOptimize`

---

## 5. Tests

`tests/test_nuisance_optimize.py` — registry unit tests, Profile1D golden-section,
workflow plan ordering, flowchart node, in-process Worker profile, Core Bridson e2e.

---

## 6. Non-goals (D13.4)

- Multi-variable nuisance (only first Variable used, V1 Profile1D).
- Other nuisance Methods beyond Profile1D.
- Full HinoLLP SUSYHIT golden parity (D13.6 progressive).
