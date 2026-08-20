# V1 Lightweight Expression-Function Migration

**Status:** implemented and compatibility-tested  
**Date:** 2026-07-13  
**Source audited:** clean `Jarvis-HEP/jarvishep/inner_func.py` from the local V1 line
(`pyproject.toml` declares 1.7.5)  
**V2 implementation:** `Jarvis-HEP-v2/jarvishep2/inner_func.py` + `expression.py`

## Decision

V2 keeps a dependency-light internal Expression Core. The complete V1 built-in lightweight
surface is now available to every `ExpressionContext` consumer: Likelihood, Operas inputs,
Calculator/Portal Dump variables, sampler Selection, and AdaptiveLevelSet targets.

Jarvis-Operas remains the extension layer. Functions that V1 discovered dynamically through
`jarvis_operas.build_sympy_dicts()` are deliberately not copied into HEP core. D11.4d now
discovers and snapshots them into the shared Worker `ExpressionContext`; see
[`OPERAS_DYNAMIC_FUNCTION_DISCOVERY_2026-07-13.md`](OPERAS_DYNAMIC_FUNCTION_DISCOVERY_2026-07-13.md).

## Migrated inventory

| Group | Names |
|---|---|
| Log/exponential | `log`, `exp`, `ln` |
| Trigonometric | `sin`, `cos`, `tan`, `sec`, `csc`, `cot`, `sinc` |
| Inverse trigonometric | `asin`, `acos`, `atan`, `asec`, `acsc`, `acot`, `atan2` |
| Hyperbolic | `sinh`, `cosh`, `tanh`, `sech`, `csch`, `coth` |
| Inverse hyperbolic | `asinh`, `acosh`, `atanh`, `acoth`, `asech`, `acsch` |
| General math | `sqrt`, `Min`, `Max`, `root`, `Abs`, `Heaviside` |
| HEP probability helpers | `Gauss`, `Normal`, `LogGauss` |
| V1 constants | `Pi`, `E`, `Inf` |
| Retained V2 aliases | `pi`, `PI` |

Total: **38 function names**, all in `inner_func.NUMERIC_MODULES`, plus five accepted constant
spellings. `EXPRESSION_FUNCTION_NAMES` and `EXPRESSION_CONSTANTS` are the inspectable catalogs.

### Probability definitions

```text
Gauss(x, μ, σ)    = exp[-0.5 ((x-μ)/σ)^2]              # unnormalised, as in V1
Normal(x, μ, σ)   = Gauss(x, μ, σ) / (σ sqrt(2π))
LogGauss(x, μ, σ) = -0.5 ((x-μ)/σ)^2                   # no normalisation term
Heaviside(0)      = 0.5
```

The numerical implementations accept Python/NumPy scalars and NumPy arrays. Scalar inputs
return scalar values; array inputs remain vectorized.

## Compatibility evidence

- Catalog test requires exact equality with the frozen V1 38-name function surface.
- Every migrated name is compiled and numerically evaluated through `ExpressionContext`.
- A source-to-source numerical audit loaded V1 `inner_func.py` and compared 37 numerical
  functions at valid scalar points: maximum absolute difference **4.44e-16**.
- `Heaviside(0) = 0.5` is tested separately.
- Vector `LogGauss` behavior is tested.
- The released GMFit expression using nested `Heaviside`, `Max`, and `Abs` is tested both
  directly and through `LogLikelihoodEvaluator`.
- Targeted expression/consumer regression: **71 passed, 38 function subtests passed**.
- Complete V2 regression: **270 passed, 1 skipped, 38 function subtests passed in 274.30 s**.
- The unmodified released-style Eggbox Bridson/Operas YAML completed a real Redis run after
  migration: **10,049 / 10,049**, zero failed, 16.9410 s, with exact `z` and `LogL_Z`
  recomputation errors `0.0`. Isolated output:

  ```text
  /tmp/jarvis-hep-v2-v1-functions-20260713/outputs/EggBox_Bridson_05/
  ```

## Deliberate exclusions

- The V1 `sympy` module object inserted into `update_funcs()` is not a lightweight function and
  is not exposed as an expression value.
- Jarvis-Operas registry functions are not duplicated into the HEP core.
- File-backed interpolation helpers are not pure built-ins; they belong in the Operas extension
  namespace and require explicit startup registration.
- No YAML key or expression spelling changed.

## Extension status

D11.4d added the dedicated Jarvis-Operas expression-function snapshot API. External functions
remain namespaced by their registered full name, so core names cannot be silently shadowed and no
new HEP YAML registration surface is required.
