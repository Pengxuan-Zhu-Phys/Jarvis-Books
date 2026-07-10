# Component — Workflow & ExecutionPlan (`jarvishep2/workflow.py`)

**Role**: turn the module list into a layered execution plan — which steps run in which layer,
which may run concurrently inside one Worker — and emit a JSON-serializable plan for Redis.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `workflow.py` 161 lines (all module-level
functions; no class).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §13.1.
**Reuses V1**: none by import.

> **As-built drift:** the design described a `Workflow(Base)` class with `set_modules`,
> `resolve_dependencies`, flowchart export, and `has_calculator`/`references_sdir` properties.
> **Shipped as a set of pure functions, no class, no flowchart export.** The lazy-materialization
> flags moved to [`runtime_config.workflow_has_calculator` / `workflow_references_sdir`](config_schema.md).

---

## 1. Module & line count

`jarvishep2/workflow.py` (161 lines). Depends on [`ExecutionStep`](sample.md). Exports
`build_execution_plan`, `build_opera_execution_plan`, `concurrency_groups`,
`execution_plan_template`, `group_by_layer`, `max_layer_width`, `resolve_module_layers`.

---

## 2. Module-level functions

| Function | Signature | Behavior |
|----------|-----------|----------|
| `_normalize_required_modules` | `(value) -> list[str]` | coerce str/list of deps; ignore mappings. |
| `resolve_module_layers` | `(modules) -> dict[str,int]` | assign layer indices from `required_modules` via iterative topological readiness (same layer = no mutual dep ⇒ concurrent-eligible); cycle-safe fallback runs remaining names together. |
| `max_layer_width` | `(layers) -> int` | widest layer (drives Worker subprocess concurrency). |
| `concurrency_groups` | `(steps) -> list[list[str]]` | module names grouped by execution layer. |
| `build_execution_plan` | `(*, calculator_modules=None, opera_modules=None, include_likelihood=True) -> list[ExecutionStep]` | layered plan **calculators → operas → likelihood**: calculators keep their layers, operas are offset to follow all calculators, a single `LogLikelihood` step is appended as the terminal layer. |
| `build_opera_execution_plan` | `(opera_modules, *, include_likelihood=True) -> list[ExecutionStep]` | opera-only convenience wrapper. |
| `group_by_layer` | `(steps) -> list[list[ExecutionStep]]` | bucket steps by ascending layer (the Worker's per-layer iterator). |
| `execution_plan_template` | `(opera_modules=None, *, calculator_modules=None, include_likelihood=True) -> list[dict]` | JSON-serializable plan (`ExecutionStep.to_dict`) for Redis transport. |

---

## 3. Execution-plan layering

```
required_modules → resolve_module_layers →
  calculators keep layer 0,1,…
  operas start at (max calc layer + 1)
  likelihood = terminal layer (after the last opera / calc layer)
⇒ execution_plan = [{calculator,A,0},{calculator,B,1},{calculator,C,1},
                    {opera,L,2},{likelihood,LogLikelihood,3}]
```

The plan is **deterministic** (stable ordering) and **data-independent** — built once and shipped
in the picklable Worker config / Sample task dict.

---

## 4. Interfaces / collaborators

- **core / sampler** build `execution_plan_template(...)` and attach it to each
  [Sample](sample.md) task.
- **Worker** ([worker.md](worker.md)) consumes it via `group_by_layer` and runs same-layer
  calculators concurrently; `max_layer_width` + `resolve_module_layers` size the Worker's
  subprocess scheduler.

---

## 5. Tests

`tests/test_workflow_execution_plan.py` (6 tests): plan determinism, layering correctness,
concurrency groups, opera-only plan, likelihood placement, JSON template shape. Layer concurrency
is exercised end-to-end in `tests/test_layer_concurrency.py`.
