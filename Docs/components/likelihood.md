# Component — LogLikelihood (`jarvishep2/Module/likelihood.py`)

**Role**: evaluate the log-likelihood expression(s) for a Sample from its observables. In V2
this runs **inside the Worker** as the terminal step of the workflow, not in the control
process.
**Status**: design — plan WP-D1.1 (opera/likelihood path) onward.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §11;
discussion `worker_design.md` §3.2.
**Reuses V1**: the expression engine + `LogLikelihood` module (`jarvishep/Module/likelihood.py`)
and `OperasModule`. Behavior is unchanged; only the *execution location* moves to the Worker.

---

## 1. Responsibilities

1. Evaluate `LogL` from `sample.info["observables"]` using the configured expression(s).
2. Be a normal **opera-style step** in the `execution_plan` (terminal layer), so the Worker
   runs it like any other in-process module.
3. Return a finite `LogL` (or a sentinel `-inf` for rejected points) and write it into the
   Sample result.
4. Preserve the M1 fix: lazy Samples expose `logger_name` so the likelihood child logger binds
   without forcing materialization (no spurious `LogL=-inf`).

---

## 2. Structure (delta over V1)

```python
class LogLikelihood(Module):
    # V1 kept: expression compilation, calculate(), nuisance handling
    def calculate(self, sample_info: dict) -> float: ...
    def evaluate(self, observables: dict) -> float: ...   # pure: obs → LogL
```

The key V2 refinement is making `evaluate(observables) -> float` a **pure function** (no I/O,
no logger side effects) so it is trivially testable and safe to call in the Worker.

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `compile` | `() -> None` | Compile the LogL expression(s) once (at Worker init, like template preload). |
| `evaluate` | `(observables: dict) -> float` | Pure evaluation `obs → LogL`; finite or `-inf`. |
| `calculate` | `(sample_info) -> float` | Worker-facing: read observables from `sample_info`, call `evaluate`, write `sample_info["likelihood"]`, log to the Sample logger. |
| `set_likelihood` / `loglike` | `(...)` | (V1) wiring for samplers that consume the value (MCMC). |

---

## 4. Placement in the pipeline

```
Worker.process_task:
   run calculator layers → observables populated
   run opera layers
   _run_likelihood(sample):  sample.info["likelihood"] = LogLikelihood.calculate(sample.info)
   status = Completed
result.to_info_dict()["likelihood"] → Archiver + (for MCMC) result feedback to the Sampler
```

For MCMC, the LogL travels back to the control-process sampler via the result stream keyed by
`uuid` (see [sampler.md](sampler.md) §3); acceptance bookkeeping is unchanged.

---

## 5. Concurrency / determinism / failure semantics

- `evaluate` is **pure and deterministic** → identical to V1 for the same observables (parity).
- A non-finite intermediate (e.g. `log(0)`) yields `-inf` (rejected point), logged to the
  Sample log, never crashing the Worker.
- Compiled once per Worker; thread/process-safe (no shared mutable state).
- Lazy-materialization compatible (M1): binds via `logger_name`, never forces a dir.

---

## 6. Interfaces

- **Worker._run_likelihood** → `calculate(sample.info)`.
- **Sample** → reads `observables`, writes `likelihood`.
- **Sampler (MCMC)** → consumes the returned value via the result stream.

---

## 7. Tests (`tests/test_likelihood_v2.py`)

Unit:
1. **Pure parity** — `evaluate(observables)` equals the V1 `LogLikelihood.calculate` value for
   the same observables (golden), across several fixtures.
2. **Rejected point** — observables producing `log(0)`/out-of-domain → `-inf`, no exception.
3. **Compile once** — expression compiled once per Worker (counter), reused across Samples.
4. **Lazy compatibility** — a lazy opera-only Sample gets a finite `LogL` (regression for the
   M1 `logger_name`/`-inf` fix).
5. **MCMC feedback** — the value reaches the sampler via the result stream and drives the same
   acceptance as the V1 fixture.

Verification logic: test 1 keeps the science identical; test 4 guards the specific M1 bug where
lazy samples returned `-inf`.

---

## 8. Open questions

- Vectorized likelihood over many observables (not needed under one-Sample-per-Worker; keep
  scalar).
- Nuisance-profiled likelihood inside the Worker (with the nuisance-in-Worker work).
