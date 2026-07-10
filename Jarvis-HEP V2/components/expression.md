# Component — Expression helpers (字母运算) (`jarvishep2/inner_func.py`, `jarvishep2/Sampling/sampling_utils.py`)

**Role**: built-in math functions for likelihood/observable formulas, and the sympy-based
selection-cut evaluator.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `inner_func.py` 29 lines + the expression helpers
in `sampling_utils.py` (68 lines).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §11;
[likelihood.md](likelihood.md), [umapper.md](umapper.md).
**Reuses V1**: none by import — V1-compatible `LogGauss` + selection evaluation reimplemented.

> **As-built drift:** the design proposed a consolidated `ExpressionEngine` /
> `ExpressionContext` / `CompiledExpression` with compile-once caching and boot validation.
> **None of that was built, and there is no `expression.py`.** What shipped is much smaller: one
> built-in function module (`inner_func.py`) and a sympy `evaluate_selection` helper inside
> `Sampling/sampling_utils.py`. Likelihood formulas are evaluated by
> [`LogLikelihoodEvaluator`](likelihood.md), not by a shared engine.

---

## 1. `inner_func.py`

| Symbol | Behavior |
|--------|----------|
| `LogGauss(xx, mean, err) -> float` | V1-compatible `−0.5·((x−m)/e)²`; coerces numpy scalars. |
| `NUMERIC_MODULES: dict` | name → callable map (`sin`, `cos`, `exp`, `log`, `sqrt`, `LogGauss`) used as the evaluation namespace. |

## 2. `Sampling/sampling_utils.py` (expression-related)

| Symbol | Behavior |
|--------|----------|
| `BoolConversionError(ValueError)` | raised when a selection expression can't be coerced to bool. |
| `evaluate_selection(expression, variables) -> bool` | `None` → `True`; otherwise `sympy.sympify(expression, locals=symbols).subs(variables)` → `bool`. Wraps failures in `BoolConversionError` naming the expression. |
| `map_u_to_physical(u_coords, variables) -> dict` | map a `u`-vector through `Variable`s. |
| `map_row_to_physical(row, variables) -> dict` | map a CSV/grid row (with per-variable `length`) through `Variable`s. |
| `row_to_u_coords(row, variables) -> np.ndarray` | inverse: row → normalized `u`. |

---

## 3. Determinism / failure semantics

- `evaluate_selection` is deterministic for a given expression + value dict; parse/coercion
  failures **raise** `BoolConversionError` (no silent truthy coercion).
- `LogGauss` and the numeric helpers are pure.

---

## 4. Interfaces / collaborators

- **LogLikelihoodEvaluator** ([likelihood.md](likelihood.md)) evaluates `LogL` expressions with a
  numpy/`inner_func` namespace.
- **Samplers** ([samplers_catalog.md](samplers_catalog.md)) may apply `evaluate_selection` cuts and
  the `map_*` helpers.
- **Variable** ([parameters_variables.md](parameters_variables.md)) backs the `map_*` helpers.

---

## 5. Tests

`evaluate_selection` and the row/u mapping helpers are exercised through
`tests/test_samplers_catalog.py`; `LogGauss` through the likelihood path in
`tests/test_worker_*` and acceptance runs.
