# Component — Benchmark mode (`jarvishep2/benchmark.py`)

**Role**: the `Jarvis2 … --benchmark [s]` throughput harness and hot-path stage timers. Produces
the numbers the slow-regime acceptance gates (D7) are measured against, plus the
`parallelism_efficiency` metric.
**Status**: ⚠️ **NOT BUILT AS A MODULE** (as-built @ `jarvis2` `d0de31a`).

> **As-built reality:** there is **no `jarvishep2/benchmark.py`** and **no `--benchmark` CLI flag**.
> The closest shipped functionality is `Jarvis2Core.run_check_modules()` / `check_modules()` (a
> fixed-point 10-sample smoke through the real distributed path, see [core.md](core.md)) and the
> slow-regime acceptance JSON checked in at `docs/benchmarks/d7_1_acceptance.json` (produced by
> `tests/test_distributed_acceptance.py`). There is **no `parallelism_efficiency` code module** and
> no stage-timer harness. The design below is retained for history.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §1, §12;
[V2_DISTRIBUTED_PLAN.md](../V2_DISTRIBUTED_PLAN.md) §4 (D7.1), [monitor.md](monitor.md).
**Reuses V1**: `benchmark.py` — `record_stage`/`snapshot_stage_timers`, `BenchmarkDeadline`,
`collect_benchmark_report`/`write_benchmark_report`. Retargeted to the distributed path + new
metrics.

---

## 1. Responsibilities

1. Run a scan for a fixed wall time and report **throughput** (samples/s), submitted/completed/
   failed, and the run_summary fields.
2. **Stage timers**: zero-overhead-when-off accumulators for the hot path — repurposed for the
   distributed path (submit, Redis round-trip, worker execute, archive).
3. Emit `parallelism_efficiency` (per dispatch layer: per-Worker subprocess fan-out, calculator
   free-pool saturation) into `benchmark.json`.
4. Back the **D7 gates**: worker scaling, calculator throughput, archive-latency/backlog bounds.

---

## 2. Structure (reused + retargeted)

```python
def enable_stage_timers(reset=True) -> None: ...
def record_stage(stage: str, elapsed: float) -> None: ...        # guarded; no-op when off
def snapshot_stage_timers(...) -> dict: ...

class BenchmarkDeadline:                  # wrap the sampler loop, stop after N seconds
    def __init__(self, sampler, seconds): ...
    def __enter__(self): ...; def __exit__(self, *exc): ...
    def elapsed_seconds(self) -> float: ...

def collect_benchmark_report(...) -> dict: ...
def write_benchmark_report(report, output_dir) -> str: ...        # benchmark.json
```

V2 stages (replacing the in-process ones): `propose+push`, `redis_roundtrip`, `worker_execute`,
`archive`, plus per-layer `parallelism_efficiency`.

---

## 3. Member functions

| Function | Signature | Behavior |
|----------|-----------|----------|
| `enable_stage_timers` | `(reset) -> None` | Turn on stage accumulation (module-level flag — zero cost when off). |
| `record_stage` | `(stage, elapsed) -> None` | Add to a stage accumulator (guarded by the flag). |
| `BenchmarkDeadline` | context mgr | Wrap the run; stop the sampler after `seconds`; expose elapsed. |
| `collect_benchmark_report` | `(...) -> dict` | Assemble throughput + stages + efficiency + run_summary fields. |
| `write_benchmark_report` | `(report, dir) -> path` | Write `benchmark.json`. |
| `compute_parallelism_efficiency` | `(coordination/Redis) -> dict` | `peak_active / configured` per layer (calculator pool, per-Worker subprocess). |

---

## 4. Distributed benchmark flow

```
Jarvis2 <task>.yaml --benchmark 30:
   enable_stage_timers()
   with BenchmarkDeadline(sampler, 30): run()        # real Redis + Workers + Archiver
   report = collect_benchmark_report(...)            # samples/s, stages, efficiency
   write_benchmark_report(report)                    # benchmark.json
   WARN if calculator-bearing workflow efficiency < 0.6   # starvation early-warning
```

Efficiency is the key slow-regime health figure: `peak_active / configured` per dispatch layer; a
value well below 1.0 means Workers/calculator slots are starved (the failure the retired
throughput-core missed).

---

## 5. Concurrency / overhead / failure semantics

- **Zero overhead when off**: stage timers are guarded by a module-level flag, not per-call config
  lookups (carried from M0).
- Benchmark mode runs the **real distributed path** (no mocks) so numbers reflect production.
- Efficiency is a **pure projection** of coordination/Redis counters (no new hot-path bookkeeping).
- A benchmark run that can't reach a gate is reported honestly with measured numbers (the plan
  forbids silently lowering a gate).

---

## 6. Interfaces

- **core.benchmark** → `Jarvis2 … --benchmark`.
- **monitor / run_summary** → shares the efficiency + throughput projection ([monitor.md](monitor.md)).
- **subprocess_scheduler / Redis** → `peak_active` sources for efficiency.

---

## 7. Tests (`tests/test_benchmark_mode.py`, `test_distributed_acceptance.py`)

Unit / integration:
1. **Runs + reports** — `--benchmark 5` on a fixture completes, `benchmark.json` exists,
   `samples_per_sec > 0`.
2. **Zero overhead off** — benchmark-off path shows no measurable regression vs. a baseline.
3. **Efficiency metric** — a saturated calculator fixture reports efficiency ≈ 1.0; a deliberately
   starved one reports < 0.6 and emits the WARNING.
4. **Worker scaling (D7)** — throughput scales ~linearly 1→2→N Workers on a calculator fixture.
5. **Archive backlog (D7)** — staging/archive-queue stays bounded under the Worker output rate.
6. **Contract** — `benchmark.json` schema is stable and documented.

Verification logic: test 3 is the standing slow-regime guard (the metric the retired design
lacked); tests 4/5 are the D7 acceptance gates.

---

## 8. Open questions

- Auto-tuning `Runtime.workers` / calculator `make_paraller` from the efficiency figure (note as
  future; do not implement in the harness).
- A reference NAS fixture for archive-latency gating (machine-relative).
