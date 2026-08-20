# Jarvis-Operas Dynamic Function Discovery — Implementation Report

**Status:** implemented on 2026-07-13  
**Scope:** dynamically discover externally registered Jarvis-Operas functions without changing
the released V1 `Operas` YAML interface.

## 1. YAML contract: V1 `Operas.Modules` unchanged

V2 accepts the released-style YAML directly:

```yaml
Operas:
  # make_paraller: 16     # V1 key is accepted but unused in V2
  Modules:
    - name: EggBox
      operator: helper.eggbox2d
      call_mode: call
      required_modules: []
      input:
        - {name: x, expression: "xx * Pi"}
        - {name: y, expression: "yy * Pi"}
      output:
        - {name: z, entry: z}
```

There is no V2-only `Operas.Functions` block. Register external functions in Jarvis-Operas, then
refer to them by their registered full name:

```yaml
Sampling:
  LogLikelihood:
    - name: LogL
      expression: user.external_loglike(x, y)
```

> Updated 2026-08-04: this example originally used a top-level `Likelihood:` block, which the
> V2 schema rejects (`JV2-SCH-001`). Likelihood terms live under `Sampling.LogLikelihood`.

The same `namespace.function(...)` syntax is available in Operas input expressions,
Calculator/Portal Dump expressions, `Sampling.selection`, and AdaptiveLevelSet targets.

## 2. Loading and runtime lifecycle

1. V2 sees a registered operator in `Operas.Modules[].operator`, or a qualified function call in
   an expression.
2. It initializes the local Jarvis-Operas registry, including persisted user operators, and calls
   `discover_entrypoints()` once for that process registry.
3. For expressions, `build_register_dicts()` and `build_sympy_dicts(..., include_all=True)` make
   a local parsing/numerical snapshot.
4. One Worker-owned `ExpressionContext` is shared by Operas, Calculator/Portal, and Likelihood.
5. Both compiled expressions and `Operas.Modules[].operator` directly invoke snapshotted numerical
   functions; they do not perform a registry lookup or `registry.call()` per Sample.

Discovery is demand-gated: Calculator-only jobs and expressions using only the internal V1
function core do not initialize Jarvis-Operas. `make_paraller` remains accepted in the YAML but
is deliberately not used: V2 concurrency is governed by long-lived Workers and workflow layers.

## 3. Registration paths

External functions are visible to every spawn Worker when supplied through:

- persisted Jarvis-Operas user operators;
- installed `jarvis_operas.core` or `jarvis_operas.user` entry points;
- built-in/catalog and hot-curve registry declarations.

An ad-hoc function registered only in a parent Python process is not spawn-transferred. Persist it
or package it as an entry point so each Worker can rediscover it.

## 4. Verification

- V1 `Operas.Modules` YAML shape is forwarded without a new functions block.
- External registry function, entry-point discovery-once, shared consumers, and zero-registry
  hot paths for both expressions and module operators are covered in `tests/test_operas_functions.py`.
- A real spawn Worker independently discovers `math.add` and evaluates
  `math.add(x, 2)` → 5.0 and `math.add(qualified_sum, 1)` → 6.0 for `x = 3`.
- Complete V2 regression after direct module-callable reuse: **280 passed, 1 skipped,
  38 function subtests**.

## 5. Remaining Operas bridge work

This closes dynamic function discovery without changing the YAML contract. Strict `call_mode`
validation, operator signature/logger alignment, and a thin `Jarvis operas list/info` UI remain
separate work.
