# Component — LogLikelihood (`jarvishep2/likelihood.py`)

**Role**: compile and evaluate the configured `LogLikelihood` expression(s) for a Sample from its
observables. Runs **inside the Worker** as the terminal step of the execution plan.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `likelihood.py` 81 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §11;
[worker.md](worker.md).
**Reuses V1**: none by import (uses `inner_func.NUMERIC_MODULES`).

> **As-built drift:** moved from `Module/likelihood.py` → top-level `likelihood.py`; **standalone
> class `LogLikelihoodEvaluator`** (no `Module` ABC). Expressions are compiled with sympy
> `lambdify`. No `set_likelihood`/`loglike`/MCMC-feedback hooks (no MCMC in V2).

---

## 1. Class defined — `LogLikelihoodEvaluator`

Compile + evaluate configured LogLikelihood expressions.

**Attributes:** `_compiled: list[tuple[name, var_names, num_expr]]` — each configured expression
sympified + `lambdify`-compiled at construction, using a fixed symbol set
(`x,y,z,shift,calc_z,LogL,LogL_Z`) and the `inner_func.NUMERIC_MODULES` namespace
(`sin/cos/exp/log/sqrt/LogGauss`).

| Method | Behavior |
|--------|----------|
| `__init__(expressions)` | compile each `{name, expression}` entry once (skips empty/non-mapping). |
| `evaluate(observables) -> dict[str,float]` | evaluate each compiled term against observables (raises `KeyError` naming missing observables); accumulate `LogL` (explicit `LogL` term wins; otherwise sum of terms); returns all terms incl. `LogL`. |
| `calculate(sample_info) -> float` | Worker-facing: read `sample_info["observables"]`, evaluate, merge terms back into observables, write `sample_info["likelihood"]`, return the `LogL`. |

---

## 2. Placement in the pipeline

```
Worker.process_task: run calculator layers → operas → _run_likelihood(sample):
    sample._likelihood = LogLikelihoodEvaluator.calculate(sample.info)
to_info_dict()["likelihood"] → Archiver
```

The likelihood step is the terminal `ExecutionStep(type="likelihood")` produced by
[workflow.build_execution_plan](workflow.md).

---

## 3. Concurrency / determinism / failure semantics

- Expressions compiled **once per Worker**; `evaluate` is pure + deterministic (golden parity).
- Missing observables → `KeyError` naming the expression (no silent default).
- Multiple terms sum into `LogL` unless an explicit `LogL` term is configured.

---

## 4. Interfaces / collaborators

- **Worker._run_likelihood** ([worker.md](worker.md)) → `calculate(sample.info)`.
- **inner_func** ([expression.md](expression.md)) supplies the numeric namespace (`LogGauss`, …).
- **Sample** ([sample.md](sample.md)) reads observables, receives `likelihood`.

---

## 5. Tests

Exercised through `tests/test_worker_mvp.py` (17), `tests/test_worker_calculator.py` (8), and the
distributed acceptance suite (LogL written into each archived record; parity vs golden).
