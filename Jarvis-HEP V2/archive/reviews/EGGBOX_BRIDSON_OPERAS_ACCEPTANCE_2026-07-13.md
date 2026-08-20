# Eggbox Bridson + Operas V1-YAML Acceptance

**Status:** passed for one sampling method  
**Date:** 2026-07-13  
**Scope:** unmodified `Jarvis-Examples/Eggbox/bin/Example_Bridson_Operas.yaml` on Jarvis-HEP V2  
**Not claimed:** compatibility of the other Eggbox sampling methods

## Outcome

The released-style V1 YAML completed a real Redis-backed V2 run without editing the YAML:

| Item | Result |
|---|---:|
| Sampling method | `Bridson` |
| Operator | `helper.eggbox2d` from Jarvis-Operas |
| Proposed / finished | 10,034 / 10,034 |
| Failed points | 0 |
| Success rate | 1.0 |
| Wall time | 24.9615 s |
| Reported throughput | 24,118.7 points/min |
| Eggbox `z` max recomputation error | 0.0 |
| `LogGauss(z, 100, 10)` max recomputation error | 0.0 |

The resulting HDF5 rows contain `xx`, `yy`, `z`, `LogL_Z`, `LogL`, `uuid`, and
`product_list`. `run_summary.json` reports 10,034 submitted, 10,034 finished, and zero
failed.

The acceptance run used a temporary project root and a temporary local Redis server. This
kept the existing Eggbox V1 `outputs/` and `checkpoints/` untouched. Output artifacts from
this acceptance are under:

```text
/tmp/jarvis-hep-v2-eggbox-bridson-20260713-1800/
  outputs/EggBox_Bridson_05/
    DATABASE/samples.hdf5
    run_summary.json
    run_summary.csv
    run_summary.txt
```

## Operas update review

The local Jarvis-Operas tree identifies itself as 1.3.8 and currently contains uncommitted
maintainer work around reloading user operator files:

- an operator loaded from the same file may replace its prior definition;
- decorator-based loads now track the names registered during execution;
- loaded operators receive source/path metadata;
- registry and namespace overrides remain context-local.

These changes were not edited from Jarvis-HEP V2. The current Operas worktree passed:

```text
pytest -q tests/test_loading.py tests/test_cli.py tests/test_registry.py
49 passed in 43.17s
```

The built-in `helper.eggbox2d` registry entry resolves successfully and returns a mapping
payload `{z: ...}`, matching the Eggbox YAML output declaration.

## Compatibility fixes made in V2

Three loader/bridge gaps blocked an otherwise valid V1 YAML:

1. A V1 YAML has no `Runtime` block. YAML loading now assigns `Runtime.mode: redis`, the only
   V2 production runtime. An explicitly supplied mode is still preserved.
2. V1 Operas input expressions use `Pi`. The Operas expression context now recognizes
   `Pi`, `pi`, and `PI` as the numeric constant π instead of missing observables.
3. `JARVIS_HEP_TASK_ROOT` / `JHEP_TASK_ROOT` are now honored by project-root inference. This
   makes isolated acceptance and external orchestration possible without modifying the task
   YAML.

Regression coverage includes:

- loader tests for missing/explicit Runtime and task-root environment override;
- a direct `helper.eggbox2d` test using `xx * Pi` and `yy * Pi`;
- a V1-shaped Bridson/Operas YAML through mapper → Worker → Operas → likelihood → Archiver;
- compile-count tests proving both Operas input and sampler-selection expressions are reused.
- the complete V2 suite after the shared-expression refactor: **266 passed, 1 skipped in
  277.58 s**.

## Small-expression hot-path optimization

The first acceptance run exposed a remaining asymmetry: Likelihood expressions were already
compiled once per Worker, but each Operas `{name, expression}` still ran `sympify` and
`lambdify` for every Sample. This has now been aligned:

The follow-up architecture refactor consolidates every path below onto
`ExpressionContext → CompiledExpression`; `jarvishep2/expression.py` is now the only production
module that calls `sympify` or `lambdify`.

| Expression kind | Compile lifecycle | Evaluation lifecycle |
|---|---|---|
| Operas `input[].expression` | once during each Worker's `OperasModule.preload()` | cached numerical callable per Sample |
| Likelihood expression | once during each Worker runtime initialization | cached numerical callable per Sample |
| `Sampling.selection` | once per control process for each `(expression, variable-name set)` | cached numerical callable per candidate |

The unmodified Eggbox YAML was run again after this optimization. It completed **10,026 / 10,026**
with zero failures, exact `z` and `LogGauss` recomputation parity, **14.0147 s** wall time, and
**42,923.4 points/min** reported throughput. Compared with the initial 24.9615 s / 24,118.7
points/min run, this is an observed 43.9% wall-time reduction and 78.0% throughput increase.
It is a useful smoke-performance signal, not a controlled benchmark: the V1 YAML has no `Seed`,
so the two Bridson runs generated slightly different point counts and were executed only once.

After consolidating all consumers onto `ExpressionContext`, a third real Redis run of the same
unmodified YAML completed **10,061 / 10,061**, zero failed, in **16.8064 s** at **35,918.4
points/min**. Exact recomputation again gave maximum `z` error `0.0` and maximum `LogL_Z` error
`0.0`. Its isolated output is:

```text
/tmp/jarvis-hep-v2-expression-context-20260713/outputs/EggBox_Bridson_05/
```

The same no-`Seed` caveat applies; this third run is a compatibility gate, not a benchmark.

After the complete V1 lightweight-function migration (including the vector-capable internal
`LogGauss` replacement), a fourth compatibility run completed **10,049 / 10,049**, zero failed,
in **16.9410 s**. Maximum recomputation errors remained `z = 0.0` and `LogL_Z = 0.0`. The full
suite at this gate is **270 passed, 1 skipped, 38 function subtests passed**. Output:

```text
/tmp/jarvis-hep-v2-v1-functions-20260713/outputs/EggBox_Bridson_05/
```

## Remaining compatibility gaps

This closes exactly one vertical slice. It does not make the full Eggbox catalog V2-ready.

- `Operas.make_paraller: 16` is accepted but does not currently determine V2 Worker count;
  the unmodified YAML therefore ran with one Worker. Mapping V1 concurrency intent requires
  a separate compatibility decision.
- `EnvReqs` and `Utils.interpolations_1D` are present in the YAML but are not needed by this
  Eggbox operator path; their general V1 parity is not established by this run.
- Operas `call_mode` still needs strict `call|acall` validation, signature filtering/logger
  parity, and discovery commands under D11.4.
- Existing V1 checkpoints are deliberately not resumed by V2. Acceptance should continue to
  use an isolated task root or a deliberate fresh-run workflow.
- No claim is made yet for MCMC, MultiNest, Dynesty, Grid, Random, or the other Operas YAMLs.

## Recommended next slice

Keep this Bridson acceptance green as the baseline. Next, implement one additional finite
sampler only after deciding how V1 `make_paraller` maps to V2 `Runtime.workers`; `Random` is
the lowest-risk next candidate because it already exists in the V2 sampler registry and does
not add a new external sampler dependency.
