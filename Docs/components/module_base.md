# Component — Module base contract + pool dissolution (`jarvishep2/Module/module.py`)

**Role**: the abstract base every module backend shares (`OperasModule`, `CalculatorModule`,
`LogLikelihood`, `Parameters`). Defines a **uniform `execute` signature**, the selection-gate
contract, and the expression-funcs binding. Also documents how V1's `ModulePool` (live-object
checkout) **dissolves** into the Redis free-pool.
**Status**: design — plan WP-D1.1 (base) / D2.1 (pool → Redis).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §9; CLAUDE.md
"Inheritance Problems"; [worker.md](worker.md), [redis_queue.md](redis_queue.md).
**Reuses V1**: `Module(Base)` (`jarvishep/Module/module.py`) selection compile/eval; replaces
`ModulePool` (`jarvishep/modulePool.py`).

---

## 1. Responsibilities

1. One **ABC** with a **consistent `execute` signature** across backends — V1's inconsistency
   (`CalculatorModule.execute(input_data, sample_info)` vs `OperasModule.execute(observables,
   sample_info)`) is unified.
2. The **selection gate** contract: compile a `selection` expression once; evaluate per Sample
   → strict bool (via the [expression engine](expression.md)).
3. Bind the **expression-funcs context** (`set_funcs`).
4. Drop the 7 unused V1 imports and the not-an-`@abstractmethod` problem.

---

## 2. Structure (V2 base)

```python
class Module(Base, abc.ABC):
    name: str
    selection: str | None
    _selection_compiled: "CompiledExpression | None"

    @abc.abstractmethod
    def execute(self, observables: dict, sample_info: dict) -> dict: ...   # UNIFORM signature

    def set_funcs(self, funcs) -> None: ...
    def set_selection(self, expression: str | None) -> None: ...
    def selection_checker(self, available_keys) -> tuple[bool, set[str]]: ...
    def evaluate_selection(self, values) -> bool: ...      # via ExpressionEngine
```

Uniform contract: **`execute(observables, sample_info) -> dict`** for every backend.
`CalculatorModule` adapts its `input_data` from `observables`/`sample_info` internally, so callers
(the Worker) treat all modules the same.

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `execute` | `(observables, sample_info) -> dict` | **abstract**; each backend returns new observables. |
| `set_selection` | `(expr) -> None` | Store + compile the selection cut once. |
| `selection_checker` | `(available_keys) -> (bool, set)` | Can the cut be evaluated yet? Which keys are missing? |
| `evaluate_selection` | `(values) -> bool` | Strict bool via the expression engine (no truthy coercion). |
| `set_funcs` | `(funcs) -> None` | Bind the expression-funcs context. |

---

## 4. ModulePool → Redis free-pool (the dissolution)

V1 `ModulePool` checked out **live module instances** (`create_instance`, `get_available_instance`,
`is_busy`, `installation_event`, instance list, PackID counter) on the control process. V2 removes
all of it:

| V1 `ModulePool` mechanism | V2 replacement |
|---------------------------|----------------|
| live-instance checkout | each **Worker holds one instance per type** (no checkout) |
| `is_busy` / availability | **Redis `calc:free:<name>`** slot pool (cross-process cap) |
| `installation_event` | idempotent install keyed by slot dir |
| `PackID` counter | `pack_id` minted per `acquire_calc` ([redis_queue.md](redis_queue.md)) |
| `installed_instances` JSON | `installed_slots` in the checkpoint blueprint |
| `selection_checker`/`evaluate_selection` | move onto `Module` base (above) |

There is **no `ModulePool` class** in V2 — concurrency is the Redis free-pool, instances are
Worker-held.

---

## 5. Concurrency / lifecycle / failure semantics

- Modules are **stateless between Samples** (except a calculator's installed program) → safe to
  reuse across Samples within a Worker.
- Selection cuts compiled once (per Worker), evaluated per Sample.
- The uniform `execute` signature removes the V1 call-site branching and lets the Worker loop over
  `execution_plan` steps uniformly.

---

## 6. Interfaces

- **Worker** → calls `execute(observables, sample_info)` on every backend uniformly.
- **operas / calculator / likelihood / parameters** → subclass `Module`.
- **expression engine** → selection compile/eval.
- **Redis free-pool** → replaces the pool's availability role.

---

## 7. Tests (`tests/test_module_base.py`)

Unit:
1. **Uniform execute** — all backends implement `execute(observables, sample_info) -> dict`; a
   calculator adapts internally (golden parity with V1 results).
2. **Abstract enforced** — instantiating a `Module` subclass without `execute` raises `TypeError`.
3. **Selection** — `evaluate_selection` strict-bools; `selection_checker` reports missing keys;
   parity with V1.
4. **No ModulePool** — there is no live-instance pool/singleton; concurrency goes through the
   Redis free-pool (assert structurally).
5. **Stateless reuse** — two Samples through one instance produce independent outputs.

Verification logic: test 1 unifies the call surface the Worker depends on; test 4 confirms the
pool is genuinely dissolved into Redis.

---

## 8. Open questions

- Whether `Parameters`/`LogLikelihood` truly fit the `(observables, sample_info) -> dict`
  contract or need a thin adapter (default: adapt internally, keep one signature).
