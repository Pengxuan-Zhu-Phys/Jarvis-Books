# Component — Module contract & spawn context (`jarvishep2/operas.py`, `jarvishep2/Module/calculator.py`, `jarvishep2/mp_context.py`)

**Role**: document the (as-built) module-execution contract and the shared spawn convention. Also
records how V1's `ModulePool` (live-object checkout) **dissolved** into the Redis free-pool.
**Status**: **As-built** @ `jarvis2` `d0de31a`.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §9;
[worker.md](worker.md), [redis_queue.md](redis_queue.md).

> **As-built drift (important):** the design proposed a shared `Module(Base, abc.ABC)` at
> `jarvishep2/Module/module.py` with a uniform `execute(observables, sample_info)` and a selection
> gate. **That base class was never built and there is no `Module/module.py`.** The two backends
> are **independent standalone classes** with *different* `execute` signatures; the Worker branches
> on `ExecutionStep.type` instead of calling a common ABC.

---

## 1. The two backends (no shared base)

| Backend | Module | `execute` signature | Worker call site |
|---------|--------|---------------------|------------------|
| `CalculatorModule` | `Module/calculator.py` | `execute(sample_info, *, runtime_prepared=False) -> dict` | `_run_calculator_step` |
| `OperasModule` | `operas.py` | `execute(observables, sample_info) -> dict` | `_run_opera_step` |
| `LogLikelihoodEvaluator` | `likelihood.py` | `calculate(sample_info) -> float` | `_run_likelihood` |

There is **no abstract method, no `selection`/`evaluate_selection` on a base class** — selection
evaluation, where used, is the free function
[`sampling_utils.evaluate_selection`](expression.md). See [calculator.md](calculator.md),
[operas.md](operas.md), [likelihood.md](likelihood.md) for each class in full.

---

## 2. Spawn context — `mp_context.py`

| Symbol | Behavior |
|--------|----------|
| `get_spawn_context() -> mp.context.BaseContext` | process-local, lazily-initialized `mp.get_context("spawn")` (invariant #10). |

`Worker(Process)` derives `Process` from this context; the Archiver process does the same. This is
the single place the spawn rule is enforced — no `fork`, anywhere.

---

## 3. ModulePool → Redis free-pool (the dissolution)

V1's `ModulePool` checked out **live module instances** on the control process. V2 removes all of
it — there is **no `ModulePool` class**:

| V1 `ModulePool` mechanism | V2 replacement |
|---------------------------|----------------|
| live-instance checkout | each **Worker holds one instance per type** (no checkout) |
| `is_busy` / availability | **Redis `calc:free:<name>`** slot pool (cross-process cap) |
| `installation_event` | idempotent install keyed by pack_id (`_installed_shadows`) |
| `PackID` counter | `pack_id` minted per `acquire_calc` ([redis_queue.md](redis_queue.md)) |

Concurrency is the Redis free-pool; instances are Worker-held and reused across Samples
(stateless except a calculator's installed program).

---

## 4. Interfaces / collaborators

- **Worker** ([worker.md](worker.md)) holds the instances and dispatches by step type.
- **RedisQueue** ([redis_queue.md](redis_queue.md)) free-pool replaces the pool's availability role.
- **calculator_pools** ([redis_queue.md](redis_queue.md)) seeds the per-calculator slots.

---

## 5. Tests

Backend behavior is covered by the per-backend tests ([calculator.md](calculator.md),
[operas.md](operas.md)); the free-pool dissolution by `tests/test_worker_pool.py` (6) and
`tests/test_redis_queue.py` (16). Spawn safety is asserted across the worker/factory suites.
