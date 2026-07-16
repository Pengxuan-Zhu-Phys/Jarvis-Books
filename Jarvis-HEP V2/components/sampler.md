# Component — Sampler base (`jarvishep2/Sampling/sampler.py`)

**Role**: the sampler base in the control process. Builds light `Sample` task dicts (with the
execution-plan template) and submits them to Redis. Concrete algorithms subclass it.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `sampler.py` 84 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §11.
**Reuses V1**: none by import — the name `SamplingVirtial` is kept (invariant #12).

> **As-built drift:** the design framed this as extending V1's `SamplingVirtial(Base)` with MCMC
> result-feedback, `set_factory`, `loglike`, etc. **Shipped is a minimal base** with only
> config/redis/plan + submit; the checkpoint/resume machinery lives in
> [`CheckpointedSampler`](samplers_catalog.md). Feedback-driven methods (AdaptiveLevelSet, D13
> MCMC) sit on [`FeedbackSampler`](feedback_sampler.md) above that — not on this file.

---

## 1. Class defined — `SamplingVirtial`

**Attributes** (from `__init__`): `config`, `info`, `redis: RedisQueue|None`, `runtime_mode`
(`auto`/`redis`), `_execution_plan_template`, `_opera_modules`, `_calculator_modules`,
`_sample_artifacts`.

| Method | Behavior |
|--------|----------|
| `set_config(config_info)` | adopt config; read `Runtime.mode` + `sample_artifacts`. |
| `set_redis(redis)` | inject the `RedisQueue` submission target. |
| `set_execution_plan_template(opera_modules=None, *, calculator_modules=None, include_likelihood=True)` | build the JSON execution-plan template (via [`workflow.execution_plan_template`](workflow.md)). |
| `_build_sample(u_coords=None) -> Sample` | new `Sample` with fresh uuid, the draw, `sample_artifacts`, and the plan rebuilt from the template. **No materialization, no `u→x`.** |
| `_submit(sample)` | `redis.push_task(sample.to_task_dict())`; requires `Runtime.mode == redis`. |
| `_submit_group(samples)` | `redis.push_many_tasks(...)` — one pipeline for a batch. |

---

## 2. Submission flow

```
draw u → _build_sample(uuid, u_coords, plan) → redis.push_task(to_task_dict)
         (u→x mapping + materialization happen in the Worker)
```

Both `_submit` paths raise unless `Runtime.mode == "redis"` and a Redis queue is configured.

---

## 3. Interfaces / collaborators

- **Concrete samplers** ([samplers_catalog.md](samplers_catalog.md)) subclass this (via
  `CheckpointedSampler`).
- **workflow** ([workflow.md](workflow.md)) supplies the execution-plan template.
- **RedisQueue** ([redis_queue.md](redis_queue.md)) is the submission target.
- **core** ([core.md](core.md)) wires `set_redis` / `set_config` / `set_execution_plan_template`.

---

## 4. Tests

Exercised by `tests/test_samplers_catalog.py` (14) and `tests/test_sample_taskdict.py` (16)
(task-dict shape, no logger on wire) and the distributed acceptance/resume suites.
