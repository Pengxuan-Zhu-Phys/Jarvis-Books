# Component — Workflow & ExecutionPlan (`jarvishep2/workflow.py`)

**Role**: turn the YAML module graph into (a) a layered DAG, (b) a per-Sample
`execution_plan`, and (c) the semantic flowchart JSON. Decides which modules of one Sample may
run concurrently.
**Status**: design — plan WP-D2.2 (layer derivation), reused from D1.1 onward.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §13.1;
discussion `Jarvis-HEP_Discussion_Summary_2026-06-21.md` §7/§8.
**Reuses V1**: extends `jarvishep/workflow.py` `Workflow(Base)` — keeps
`resolve_layers`, `resolve_dependencies`, `analyze`, `build_flowchart_semantics`,
`export_flowchart_semantics`.

---

## 1. Responsibilities

1. Build the **module DAG** from `required_modules` (existing `resolve_dependencies`).
2. **Layer** the DAG (existing `resolve_layers`) — modules in the same layer have no data
   dependency and **may run concurrently inside one Worker**.
3. Emit a compact, **picklable `execution_plan`** (list of `ExecutionStep`) per Sample type —
   computed once at init, attached to every Sample (it is identical across samples; only the
   data differs).
4. Keep the **flowchart export** (`flowchart.json` + optional PNG) unchanged.
5. Expose **`workflow_has_calculator`** and **`references_sdir`** flags (drive lazy
   materialization, M1 carry-over).
6. **Absorb the (dissolved) ModuleManager registry role**: parse `Operas`/`Calculators` module
   specs (type, config, `selection`, `required_modules`) into the picklable **module blueprint**
   shipped to Workers. V2 has **no `ModuleManager` orchestrator** — module config is data
   (blueprint), module execution is the Worker's job. See
   [V1_TO_V2_MAP.md](V1_TO_V2_MAP.md) (`moduleManager.py` → dissolved).

---

## 2. Structure

```python
class Workflow(Base):
    layers: list[list[Module]]            # resolved DAG layers (V1)
    modules: dict[str, Module]            # by name (V1)

    # --- V2 additions ---
    def build_execution_plan(self) -> list[ExecutionStep]: ...
    def execution_plan_template(self) -> list[ExecutionStep]: ...   # cached
    def concurrency_groups(self) -> list[list[str]]: ...            # names per layer
    @property
    def has_calculator(self) -> bool: ...
    @property
    def references_sdir(self) -> bool: ...
```

`ExecutionStep` is the same light dataclass as in [Sample](sample.md) §2.

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `set_modules` | `(modules) -> None` | (V1) register modules; trigger `resolve_dependencies`. |
| `resolve_dependencies` | `() -> None` | (V1) build the DAG from `required_modules`. |
| `resolve_layers` | `() -> None` | (V1) topological layering; **same-layer = concurrency-eligible**. |
| `build_execution_plan` | `() -> list[ExecutionStep]` | Flatten layers into ordered `ExecutionStep`s with `type` (calculator/opera/likelihood) and `layer`. **Pure, deterministic.** |
| `execution_plan_template` | `() -> list[ExecutionStep]` | Cache of `build_execution_plan()` — attached to every Sample (data-independent). |
| `concurrency_groups` | `() -> list[list[str]]` | Per-layer module-name groups the Worker dispatches concurrently. |
| `build_flowchart_semantics` | `(name=None)` | (V1) semantic flowchart graph. |
| `export_flowchart_semantics` | `async (save_path, name=None)` | (V1) write `flowchart.json`. |
| `has_calculator` / `references_sdir` | property | Computed once; gate lazy materialization. |

V1 `run` / `run_all_samples` (single-Sample, in-process) are **not used** in V2 — execution
moves to the Worker. They stay only for the V1 line.

---

## 4. Execution plan & layering

```
YAML required_modules  →  resolve_dependencies  →  resolve_layers
   layer 0: [GenCalc]                 (1 calculator)
   layer 1: [RivetCalc, FastJetCalc]  (2 calculators, no mutual dep ⇒ concurrent)
   layer 2: [Likelihood]              (in-process opera)
⇒ execution_plan = [
     {calculator,GenCalc,0}, {calculator,RivetCalc,1}, {calculator,FastJetCalc,1},
     {likelihood,Likelihood,2} ]
```

Default layering is **auto** (from `required_modules`). An explicit `parallel:`/`layer:`
override in YAML is a deferred enhancement (design §13.1) — the schema reserves the key but
D2.2 ships auto only.

---

## 5. Concurrency / determinism

- The plan is **deterministic** for a given config (stable module ordering) so checkpoints and
  golden parity are reproducible.
- The plan is **data-independent** — built once, shared by reference across all Samples; the
  Worker reads it, the Sample only carries a pointer/copy.
- Layer order is **sequential**; within a layer the Worker fans out (see [worker.md](worker.md)).

---

## 6. Interfaces

- **Sampler** asks the Workflow for `execution_plan_template()` to attach to each Sample.
- **Worker** consumes `execution_plan` layer by layer; uses `concurrency_groups()` to fan out.
- **Factory** ships the workflow blueprint to Workers at spawn (picklable: names + config, not
  live module instances).
- **core.init_workflow** still exports `flowchart.json`.

---

## 7. Tests (`tests/test_workflow_execution_plan.py`)

Unit:
1. **Plan determinism** — same config → identical `execution_plan` (stable order) across runs.
2. **Layering correctness** — independent modules land in the same layer; dependents in later
   layers; a hand-checked fixture asserts the exact groups.
3. **Concurrency groups** — `concurrency_groups()` matches `resolve_layers()` membership.
4. **Flags** — `has_calculator`/`references_sdir` correct on opera-only vs calculator fixtures
   (drives lazy materialization).
5. **Flowchart parity** — `flowchart.json` byte-identical to the V1 golden for a fixed config
   (frozen output, untouched).
6. **Picklability** — the shipped blueprint pickles under `spawn` (no live module instances).

Verification logic: layering (test 2) is the load-bearing one — it determines what runs
concurrently inside a Sample, which is the V2 source of slow-regime speedup.

---

## 8. Open questions

- Explicit parallel-layer YAML syntax vs pure auto-derivation (design §13.1).
- Whether `likelihood` is always the terminal layer or can interleave (keep terminal for now).
