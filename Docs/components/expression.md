# Component — Expression Engine (字母运算) (`jarvishep2/expression.py`)

**Role**: the symbolic/formula evaluation subsystem. Compiles and evaluates user-written
expressions — selection cuts, likelihood/observable formulas, variable transforms — over a
context of custom functions, constants, and named symbols.
**Status**: design — auxiliary; used from D0.1 (variables/selection) onward.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §11 (reused
subsystem); [likelihood.md](likelihood.md), [workflow.md](workflow.md), [umapper.md](umapper.md).
**Reuses V1**: `inner_func.py` (`Gauss/Normal/LogGauss`, `update_funcs`, `update_const`,
`build_expression_context`, `_extract_operas_full_name_map`) and `SamplingVirtial.evaluate_selection`
(sympy `sympify`/`subs`/`bool`). V2 **consolidates** these scattered call sites into one engine.

---

## 1. Responsibilities

1. Provide one **expression context** = custom functions ∪ constants ∪ operas-name map ∪
   per-evaluation symbols.
2. **Compile** an expression once (cache the parsed form) and **evaluate** it many times against
   different value dicts — the hot path (per Sample) must not re-parse.
3. Serve all four expression sites with one engine:
   - **selection cuts** → strict `bool`,
   - **likelihood / observable formulas** → `float`,
   - **variable transforms** (where expressions appear in distributions),
   - **derived observables** declared in YAML.
4. Be **picklable/rebuildable under spawn** so Workers build the same context as the control
   process (identical results — parity).
5. Fail **loudly and specifically** (no silent `-inf`/`True`): a bad expression names itself.

---

## 2. Structure

```python
class ExpressionContext:
    funcs:  dict[str, Callable]     # Gauss, Normal, LogGauss, interp, custom user funcs
    consts: dict[str, float]        # pi, e, user constants, operas-name map
    def symbols(self, names: Iterable[str]) -> dict[str, sympy.Symbol]: ...
    def build(self) -> dict: ...    # funcs ∪ consts (the sympify locals base)

class CompiledExpression:
    raw: str
    expr: "sympy.Expr"              # parsed once
    free_symbols: frozenset[str]
    def eval_bool(self, values: dict) -> bool: ...
    def eval_float(self, values: dict) -> float: ...

class ExpressionEngine:
    def __init__(self, context: ExpressionContext): ...
    def compile(self, raw: str) -> CompiledExpression: ...   # cached by raw text
    def eval_bool(self, raw: str, values: dict) -> bool: ...
    def eval_float(self, raw: str, values: dict) -> float: ...
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `update_funcs` | `(funcs={}) -> dict` | (V1) merge built-in math funcs (`Gauss/Normal/LogGauss`, interpolators) + user `Utils` funcs. |
| `update_const` | `(values={}) -> dict` | (V1) merge constants (`pi`,`e`, user constants, operas full-name map). |
| `ExpressionContext.build` | `() -> dict` | The `sympify` `locals` base (funcs ∪ consts). |
| `ExpressionEngine.compile` | `(raw) -> CompiledExpression` | `sympy.sympify(raw, locals=context.build() ∪ symbols)`; cache by `raw`. Raises a named error on parse failure. |
| `CompiledExpression.eval_bool` | `(values) -> bool` | `subs(values)` → strict `bool`; raise `BoolConversionError` if not boolean-coercible. |
| `CompiledExpression.eval_float` | `(values) -> float` | `subs(values)` → `float`; on non-finite, return `-inf` **only** where the call site asks for a likelihood sentinel, else raise. |
| `free_symbols` | property | The variable names the expression needs (for validation against available observables). |
| `signature` | `() -> str` | Stable hash of (context + expression set) for checkpoint integrity / parity. |

---

## 4. The four call sites

```
selection cut        eval_bool(expr, {obs_name: value, ...})   → keep/drop the Sample
likelihood formula   eval_float(LogL_expr, observables)        → LogL  (likelihood.md)
derived observable   eval_float(obs_expr, observables)         → new observable column
variable transform   (UMapper uses compiled transforms)        → u→x   (umapper.md)
```

All four go through one engine, so a custom function defined once in `Utils` is visible
everywhere, and behavior is consistent.

---

## 5. Concurrency / determinism / failure semantics

- **Compile-once / evaluate-many**: the parse (`sympify`) is the cost; it runs once per unique
  expression (cache), never per Sample. Hot-path is `subs` only.
- **Determinism**: same context + same values ⇒ same result, in the control process and in any
  Worker (the basis of parity). Custom funcs ship as **importable references** (picklable), not
  lambdas, so the Worker rebuilds an identical context.
- **Validation up front**: at config load, every expression is compiled once and its
  `free_symbols` checked against declared observables/variables — typos fail at boot, not after
  hours of scanning.
- **No silent failure**: parse/coercion errors raise `ExpressionError`/`BoolConversionError`
  naming the offending expression. The likelihood `-inf` sentinel is **opt-in** per call site,
  never a swallowed exception (avoids the V1 `log(0) → -inf` silent trap).

---

## 6. Interfaces

- **config loader** → compile-and-validate all expressions at boot (`free_symbols` check).
- **Sampler** → `eval_bool` for selection cuts (replaces `evaluate_selection`).
- **LogLikelihood** → `eval_float` for the LogL formula.
- **Workflow / observables** → `eval_float` for derived observables.
- **UMapper** → compiled variable transforms.

---

## 7. Tests (`tests/test_expression_engine.py`)

Unit:
1. **V1 parity** — `eval_bool`/`eval_float` reproduce `evaluate_selection` / V1 formula values
   for a battery of expressions + value dicts (golden).
2. **Custom funcs/consts** — `Gauss/Normal/LogGauss`, interpolators, and a user-defined `Utils`
   function are all callable inside an expression.
3. **Compile-once** — a repeated expression parses once (cache hit; assert via counter); `subs`
   runs each time.
4. **Boot validation** — an expression referencing an undeclared symbol fails at compile with a
   message naming the symbol.
5. **Strict bool** — non-boolean selection result raises `BoolConversionError` (no truthy
   coercion of arbitrary numbers unless intended).
6. **No silent -inf** — `log(0)` raises unless the call site explicitly requests the likelihood
   sentinel.
7. **Spawn parity** — context rebuilt from importable refs in a child process yields identical
   results.

Verification logic: test 1 is the science-parity gate (cuts/likelihoods identical to V1); tests
4/6 close the V1 silent-failure traps (typos and `log(0)`).

---

## 8. Open questions

- Optional fast numeric backend (`lambdify`/numexpr) for hot derived-observable formulas vs.
  `subs` (default: `subs`; add `lambdify` only if measured to matter — slow regime rarely cares).
- Whether to freeze a curated allowed-function list for safety (default: yes, the `Utils` +
  built-in set; no arbitrary `eval`).
