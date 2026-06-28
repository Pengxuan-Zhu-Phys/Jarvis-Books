# Component — CalculatorModule (V2 changes) (`jarvishep2/Module/calculator.py`)

**Role**: run one external calculator for one Sample. In V2 it is **held long-term by a
Worker** (one instance per type), with templates pre-loaded, executed through the Worker's
scheduler, and concurrency-capped by the Redis free-pool.
**Status**: design — plan WP-D1.2 (`preload_templates`, `execute`) → D2.1 (slots) → D2.3
(clone_shadow) → D3 (command/env).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §8;
discussion `worker_design.md` §3.2/§4.1/§10.
**Reuses V1**: extends `CalculatorModule(Module)` (`jarvishep/Module/calculator.py`) — keeps
`install`, `run_command`, `read_output`, `load_input`, `decode_shadow_*`. **Adds** a few thin
methods; does **not** rewrite execution.

---

## 1. Responsibilities

1. Be **constructed once per Worker** and reused across all Samples that Worker handles.
2. **Pre-load templates** at startup (avoid per-Sample template parse).
3. Provide a **synchronous convenience entry** `execute(sample_info)` over the existing async
   `run_command` machinery.
4. Resolve per-Sample tokens (`@SampleID/@Sdir/@PackID`) at execution time (Phase 2, see
   [command_parser.md](command_parser.md)).
5. Support **clone_shadow** isolation and **env_setup** activation.
6. Stay **stateless between Samples** except for the installed program (the slot), so the same
   instance is safely reused.

---

## 2. Structure (delta over V1)

```python
class CalculatorModule(Module):
    # --- V1 (kept) ---
    install(), run_command(), execute_commands(), read_output(), load_input(),
    decode_shadow_commands(), decode_shadow_path(), set_subprocess_scheduler()

    # --- V2 additions ---
    def preload_templates(self) -> None: ...
    def execute(self, sample_info: dict) -> dict: ...        # sync convenience (replaces V1 execute(input_data, sample_info))
    def bind_env(self, env: dict) -> None: ...               # from env_setup cache
    def acquire_pack_id(self, pack_id: str) -> None: ...      # tag this run (was assign_ID)
```

Carries the V1 bug fix `custom_format(self, record)` (`calculator.py:138`, invariant #14).

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `preload_templates` | `() -> None` | Parse input/command templates once at Worker startup; cache compiled forms. |
| `execute` | `(sample_info) -> dict` | Sync wrapper: `load_input` → `run_command` (via the Worker scheduler, with timeout) → `read_output`; returns observables. Resolves per-Sample tokens. |
| `install` | `async () -> None` | (V1) install/build the program into its slot dir (idempotent; once per slot). |
| `run_command` | `async (command, stage, idx, timeout)` | (V1) async subprocess exec with env + timeout. |
| `bind_env` | `(env) -> None` | Merge captured `env_setup` vars into the subprocess env (cached per Worker). |
| `decode_shadow_path` / `decode_shadow_commands` | `(...)` | (V1) clone_shadow path/command rewriting. |
| `acquire_pack_id` | `(pack_id) -> None` | Tag the current run for traceability (Blueprint `pack_id`). |
| `read_output` / `load_input` | `async (...)` | (V1) IO at the per-Sample save dir. |

---

## 4. Execution within the Worker

```
Worker._run_calculator_step(step, sample):
    pack_id = redis.acquire_calc(step.name)          # blocks on calc:free:<name>
    try:
        calc = self.calculators[step.name]
        calc.acquire_pack_id(pack_id)
        if calc.clone_shadow: calc.decode_shadow_path(...)   # per-Sample physical dir
        calc.execute(sample.info)                    # load_input → run_command → read_output
    finally:
        redis.release_calc(step.name, pack_id)       # always release (no leak)
```

Same-layer calculator steps are dispatched concurrently by the Worker's
`AsyncSubprocessScheduler`; each still acquires its own slot, so global concurrency stays
capped at `make_paraller` across all Workers.

---

## 5. Concurrency / isolation / failure semantics

- **One instance per type per Worker**, reused — but each *run* acquires a fresh slot +
  `pack_id`, so traceability is per-run while initialization is amortized.
- **clone_shadow** tools get a per-Sample copy (`decode_shadow_*`); safe tools symlink in via
  `registered_executables` (zero copy).
- **Timeout**: `Calculators.Modules[].timeout` honored; on timeout the step raises, the slot is
  released, the Sample is marked Failed with a log.
- **env_setup**: `bind_env` applies the cached environment (script sourced once per Worker, see
  [env_setup.md](env_setup.md)).
- **Statelessness**: nothing about Sample N survives into Sample N+1 except the installed
  program — verified by a reuse test.

---

## 6. Interfaces

- **Worker**: constructs, `preload_templates`, `execute`, holds the instance; manages slots.
- **RedisQueue**: free-pool acquire/release (concurrency cap + `pack_id`).
- **AsyncSubprocessScheduler**: actual subprocess execution (per Worker).
- **CommandParser / env_setup**: token resolution (Phase 2) + env injection.

---

## 7. Tests (`tests/test_worker_calculator.py`, `test_calculator_v2.py`)

Unit:
1. **execute() parity** — `execute(sample_info)` produces observables equal to the V1
   `execute(input_data, sample_info)` on the same input (golden).
2. **Template preload once** — `preload_templates` parses once; N executes don't re-parse
   (assert via a counter).
3. **Instance reuse statelessness** — two Samples through one instance produce independent,
   correct outputs (no carryover).
4. **Slot discipline** — slot acquired before run, released in `finally` even on a forced
   exception/timeout (no leak; assert `CALC_STATUS`).
5. **clone_shadow isolation** — concurrent Samples get distinct physical dirs (no file
   cross-talk).
6. **env_setup** — `bind_env` makes sourced vars visible to the subprocess; script sourced once
   per Worker.
7. **pack_id traceability** — each run's outputs/logs carry a unique `pack_id`.

Verification logic: tests 1/3 keep calculator results identical to V1; test 4 is the
anti-leak/anti-serialization guard that protects the slow-regime throughput.

---

## 8. Open questions

- Re-install on config drift mid-run (default: install is idempotent, keyed by slot dir).
- Whether `execute` should return observables or write them into `sample.info` (default:
  return + Worker merges, to keep the function testable in isolation).
