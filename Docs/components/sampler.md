# Component — Sampler binding (`jarvishep2/Sampling/sampler.py`)

**Role**: the "brain" in the control process. Proposes parameter points (`u_coords`), builds
light task dicts, and pushes them to Redis. Keeps all V1 sampling algorithms unchanged; only
the **submission path** moves from `factory.submit_task` to `redis.push_task`.
**Status**: design — plan WP-D0.1 (Sample build) → D1.1 (Redis submit) → D5.1 (op_count).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §11; discussion
`worker_design.md` §1, `…架构升级…` §4.3.
**Reuses V1**: extends `SamplingVirtial(Base)` (`jarvishep/Sampling/sampler.py`) and **all**
concrete samplers (Random, Grid, Bridson, the MCMC family, Dynesty, MultiNest, …) via the
shared base — proposal logic, checkpoint/heartbeat, selection eval are untouched.

---

## 1. Responsibilities

1. Draw `u_coords` exactly as in V1 (RNG, chains, acceptance — no change).
2. Build a `Sample(uuid, u_coords)` and attach the Workflow's `execution_plan_template()`.
3. **Submit** via `redis.push_task(sample.to_task_dict())` (single) or `push_many_tasks`
   (batch of proposals) instead of `factory.submit_task`.
4. Bump `hep:task:op_count` on submit (done inside `push_task`).
5. Keep **checkpoint/resume** from the control process (RNG + chains), unaffected by Redis.
6. Slim down: the Sampler **no longer materializes** Samples or maps `u → x` — that moves to
   the Worker (the accepted "worker-owned sample lifecycle" direction).

What it must **not** do: open per-Sample dirs/logs, run modules, or hold worker state.

---

## 2. Structure (delta over V1 `SamplingVirtial`)

```python
class SamplingVirtial(Base):           # name kept (invariant #12)
    # --- V1, unchanged ---
    runtime checkpoint heartbeat, export/import_runtime_state, evaluate_selection, ...

    # --- V2 binding ---
    redis: RedisQueue | None = None
    workflow: Workflow | None = None
    def set_redis(self, redis: RedisQueue) -> None: ...
    def _build_sample(self, u_coords) -> Sample: ...
    def _submit(self, sample: Sample) -> None: ...
    def _submit_group(self, samples: list[Sample]) -> None: ...
```

The control loop calls `_submit` where V1 called `self.factory.submit_task(sample.info)`.

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `set_redis` | `(redis) -> None` | Inject the `RedisQueue` (V2 submission target). |
| `set_factory` | `(...)` | (V1) retained for the thread line; in V2 the "factory" is `TaskFactory`, which the Sampler does **not** submit through. |
| `_build_sample` | `(u_coords) -> Sample` | New `Sample` with fresh `uuid`, the raw draw, and `execution_plan = workflow.execution_plan_template()`. **No materialization.** |
| `_submit` | `(sample) -> None` | `redis.push_task(sample.to_task_dict())`. |
| `_submit_group` | `(samples) -> None` | `redis.push_many_tasks([s.to_task_dict() for s])` — one pipeline for a batch of proposals (MCMC ready-chain drain, Random/Grid blocks). |
| `export_runtime_state` / `import_runtime_state` | `() ...` | (V1) checkpoint payload — RNG, chains, ready queue. Unchanged. |
| `persist_runtime_checkpoint` | `(force, reason)` | (V1) 30 s heartbeat save at safe barriers. Unchanged. |
| `loglike` / `set_likelihood` | `(...)` | (V1) likelihood wiring — in V2 the LogL is computed in the Worker; the Sampler keeps the interface for samplers that need a value back (e.g. MCMC) via the result stream. |

**MCMC note**: for samplers that need the LogL of a proposal *before* the next proposal
(Metropolis), the control loop consumes the result stream (`hep:results`/Archiver feedback)
keyed by `uuid` — the acceptance bookkeeping and state machine (`MCMC_STATE_MACHINE_DESIGN`)
stay intact; only the transport of "evaluate this point" changes.

---

## 4. Submission flow

```
V1:  draw u → map u→x → build Sample → set_config (materialize!) → factory.submit_task(info)
V2:  draw u → build Sample(uuid,u_coords, plan) → redis.push_task(to_task_dict)
                                                   (mapping + materialize happen in the Worker)
```

Batch proposers (Random/Grid/Bridson) emit `_submit_group`; MCMC drains its ready chains into
one group per iteration. `batch_size` only affects **submission pipelining**, never execution
(invariant #6 — one Sample per Worker).

---

## 5. Determinism / checkpoint / failure semantics

- **RNG draw order is identical to V1** for the same seed (the draw code is unchanged) — this
  is the basis for golden parity. Verified by a seeded-sequence test.
- **Reproducibility across Worker counts**: the Sampler owns the master `SeedSequence`; child
  streams are derived per Sample/Worker (D6.2) so results don't depend on how many Workers run.
- **Resume**: rebuild from the sampler checkpoint; **drain** any stale `hep:task_queue`; never
  restore in-flight; re-propose. (Design §10.)
- **Backpressure**: if `hep:task_queue` grows beyond a bound, the Sampler throttles submission
  (so the queue can't grow unboundedly faster than Workers drain) — a simple high-water check
  on `llen`.

---

## 6. Interfaces

- **Workflow** → `execution_plan_template()` for each Sample.
- **RedisQueue** → `push_task` / `push_many_tasks`; result stream for MCMC feedback.
- **Distributor** (`jarvishep2/distributor.py`) still selects the sampler class by name
  (unchanged dispatch).
- **core** wires `set_redis`, `set_config`, `load_variable`, checkpoint context.

---

## 7. Tests (`tests/test_sampler_submit.py`)

Unit (`fakeredis`):
1. **Seeded equivalence** — Random/Grid/Bridson `u_coords` sequence is **byte-identical** to a
   pre-change V1 golden record for a fixed seed (RNG order unchanged).
2. **Task shape** — each submitted task is a light dict (uuid, u_coords, plan), `json`-able,
   no logger/params.
3. **Group submit** — a batch of proposals pushes via one pipeline; `task:op_count` increments
   by the group size.
4. **No materialization** — submitting leaves `SAMPLE/` empty (the Sampler never opens dirs).
5. **MCMC feedback** — a fake result stream feeds LogL back by `uuid`; acceptance bookkeeping
   matches the V1 state-machine fixture (checkpoint round-trip green).
6. **Resume drain** — on resume, stale queued tasks are dropped and re-proposed; no duplicate
   UUIDs in the final golden set.

Verification logic: test 1 is the parity foundation (same draws ⇒ same science); test 5
guarantees MCMC semantics survive the transport change.

---

## 8. Open questions

- Result-feedback latency for tight MCMC loops (control loop waiting on `hep:results`) — may
  need a dedicated low-latency result channel vs. the Archiver queue.
- Backpressure policy (fixed high-water vs. adaptive to Worker count).
