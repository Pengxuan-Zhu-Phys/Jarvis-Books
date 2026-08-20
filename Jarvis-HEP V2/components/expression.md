# Component — Shared Expression Runtime (`jarvishep2/expression.py`)

**Role**: one object-oriented compile/evaluate mechanism for every small YAML expression.
**Status**: **As-built** @ `jarvis2` `0a5e85e` (shared `ExpressionContext` + Operas discovery, 2026-07-14).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4;
[likelihood.md](likelihood.md), [operas.md](operas.md), [io_system.md](io_system.md),
[samplers_catalog.md](samplers_catalog.md).
**Reuses V1**: the complete lightweight function/constant contract, reimplemented with
NumPy/SymPy semantics; no V1 runtime imports. Migration audit:
[`../archive/reviews/V1_LIGHTWEIGHT_FUNCTION_MIGRATION_2026-07-13.md`](../archive/reviews/V1_LIGHTWEIGHT_FUNCTION_MIGRATION_2026-07-13.md).

The former state had separate `sympify`/`lambdify` implementations in Likelihood, Operas,
Calculator I/O, sampler selection, and AdaptiveLevelSet. That duplication is removed:
`rg "sympify|lambdify" jarvishep2` now finds the implementation only in `expression.py`.

---

## 1. Object model

### `ExpressionContext`

Owns one expression-language namespace and one bounded, process-local compile cache.

| API | Behavior |
|---|---|
| `ExpressionContext(functions=None, constants=None, parse_locals=None, maxsize=128)` | start with the complete V1 Expression Core namespace; optionally extend numeric functions and qualified parsing namespaces |
| `compile(text, symbols=()) -> CompiledExpression` | normalize the symbol contract, `sympify` + `lambdify` on a cache miss, return the same compiled object on a hit |
| `evaluate(text, values, symbols=None)` | convenience compile/cache + evaluate path |
| `update_functions(mapping)` | extend/replace functions and invalidate old compiled callables |
| `update_namespace(functions=…, parse_locals=…)` | atomically extend both namespaces and invalidate compiled callables |
| `clear_cache()` / `cache_info()` | test/diagnostic cache control and hit/miss counters |

The cache key is `(expression text, sorted explicit symbol names)`. A context is guarded by an
`RLock`; compilation and cache mutation are thread-safe within a process. It is deliberately
not serialized or shared across processes.

`ExpressionContext`, `CompiledExpression`, and `MissingExpressionVariablesError` are also
exported from the package root for extension code:

```python
from jarvishep2 import ExpressionContext
```

### `CompiledExpression`

Immutable compiled value object with:

- original `expression` text;
- sorted `variable_names` dependency tuple;
- hidden numerical callable;
- `evaluate(values)` / `__call__(values)` for repeated binding and evaluation.

Missing dependencies raise `MissingExpressionVariablesError`, which exposes structured
`expression` and `missing` fields. Each consumer may translate it into its established
domain message (`misses observables`, `misses parameters`, or `BoolConversionError`).

### Standard language

- Constants: `Pi`, `pi`, `PI` → π; `E` → Euler's number; `Inf` → positive infinity.
- Log/exponential: `log`, `exp`, `ln`.
- Trigonometric: `sin`, `cos`, `tan`, `sec`, `csc`, `cot`, `sinc`, and all inverse forms
  `asin`, `acos`, `atan`, `asec`, `acsc`, `acot`, `atan2`.
- Hyperbolic: `sinh`, `cosh`, `tanh`, `sech`, `csch`, `coth`, plus `asinh`, `acosh`, `atanh`,
  `acoth`, `asech`, `acsch`.
- General: `sqrt`, `Min`, `Max`, `root`, `Abs`, `Heaviside`.
- Probability: `Gauss`, `Normal`, `LogGauss` with frozen V1 definitions.
- Numerical backend: the 38-name catalog in `inner_func.NUMERIC_MODULES`, then NumPy.

This removes the earlier inconsistency where Operas supported fewer functions than Likelihood.
The same complete catalog is available to every consumer. Jarvis-Operas adds a demand-loaded
extension snapshot of qualified registered functions; no new HEP YAML alias surface is needed.

---

## 2. Consumers and lifetime

| Consumer | Context owner | Compile point | Reuse scope |
|---|---|---|---|
| Operas `input[].expression` | Worker-owned shared context | Worker preload | every Sample handled by that Worker |
| Likelihood expressions | Worker-owned shared context | Worker runtime initialization | every Sample handled by that Worker |
| Calculator/Portal Dump variables | Worker-owned shared context | first expression use | all Calculator modules/Samples in that Worker process |
| `Sampling.selection` | sampler-owned context | first candidate/probe | all candidates in the control process |
| AdaptiveLevelSet target | sampler-owned context | `set_config` / resume rebuild | all feedback generations in the control process |

Compiled SymPy callables and context locks are runtime state only. Checkpoints keep expression
text/config and rebuild compiled objects after resume.

---

## 3. Compatibility wrappers

`Sampling.sampling_utils.evaluate_selection` remains the sampler-facing API so existing code
still gets `None → True` and `BoolConversionError`. `io_portal.evaluate_io_expression` remains
the Portal callback and retains Calculator numeric-string coercion. `LogLikelihoodEvaluator`
retains ordered term dependencies and explicit-vs-summed `LogL` behavior.

No YAML key or expression spelling changed in this refactor.

---

## 4. Known boundary

Explicit `symbols=` protects names known by a consumer, but a name that is neither declared nor
recognized before SymPy parsing can still collide with SymPy built-ins (`I`, `gamma`, …).
The V1 constants `Pi`, `E`, and `Inf` (plus `pi`/`PI`) are intentionally reserved and take
precedence over identically named observables.
Registered Operas namespaces are parse-local objects and similarly take precedence when a
qualified function call uses the same name as an observable.
Future boot validation should pass the complete workflow observable contract to every context;
the shared class now provides the single place to implement that improvement.

---

## 5. Tests

- `tests/test_expression.py`: cache identity/hits, standard language, structured missing values,
  function-update invalidation, exact V1 catalog, all-name numerical evaluation, vector
  Gaussian behavior, released GMFit expression, and Likelihood ordered dependencies.
- Existing Operas, Portal/Calculator, sampler catalog, AdaptiveLevelSet, Worker, and Eggbox
  suites provide consumer-level regression coverage.
- `tests/test_operas_functions.py`: external registration, entry-point discovery-once,
  multi-consumer reuse, demand gating, and a hot-path proof that registry methods are not
  consulted after snapshot creation.

Verification on 2026-07-13 after V1 function migration: complete V2 suite **270 passed,
1 skipped, 38 function subtests passed**; unmodified V1 Eggbox Bridson/Operas YAML
**10,049 / 10,049**, zero failed, exact Opera/Likelihood recomputation parity.

After dynamic Operas function discovery, V1-YAML interface correction, and direct module-callable reuse, the complete suite is **280 passed, 1 skipped,
38 function subtests passed**. A dedicated spawn-Worker test additionally proves that qualified
functions are rediscovered inside the child process rather than inherited from parent memory.
