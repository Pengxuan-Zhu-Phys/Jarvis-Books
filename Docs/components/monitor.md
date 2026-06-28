# Component — Monitor / Dashboard (`jarvishep2/dashboard.py`)

**Role**: a read-only, independent view of a running scan. Reads the Factory's
`op_count`-driven snapshot (or Redis directly) and renders Sampler / Factory / Workers /
Calculators / Samples / Resources at up to 60 Hz. Also the source of `run_summary`.
**Status**: design — plan WP-D5.2 (depends on D5.1 snapshot).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §6;
discussion `factory_design.md` §6/§8, Blueprint §6 (independent monitor process).
**Reuses V1**: the run-summary contract (`docs/specs/RUN_SUMMARY_METRICS.md`, frozen) and,
optionally, the existing Textual layout patterns.

---

## 1. Responsibilities

1. Attach to a running scan **without perturbing it** — read-only, no Redis writes.
2. Render live panels from `get_monitor_snapshot()` (in-process) or Redis keys (out-of-process
   via `Jarvis2 --monitor --pid N`).
3. Compute `run_summary.{json,csv,txt}` from Redis counters at shutdown, **field-for-field
   equal** to the frozen schema.
4. Exit cleanly with a clear message when no scan is running.

---

## 2. Structure

```python
class SnapshotReader:                      # pure data, unit-testable, no UI
    def __init__(self, source): ...        # source: TaskFactory | RedisQueue
    def read(self) -> MonitorView: ...

@dataclass
class MonitorView:
    sampler: dict        # proposed, accepted, queue_len
    factory: dict        # workers alive, peak, respawns
    workers: list[dict]  # per-worker status/current_sample/heartbeat
    calculators: dict    # per-calc free/busy
    samples: dict        # running/completed/failed
    resources: dict      # cpu/mem/fd (sampled locally)

class Dashboard:                            # Textual app (thin UI over SnapshotReader)
    def compose(self): ...
    def on_mount(self): self.set_interval(1/10, self.refresh_view)
    def refresh_view(self): ...
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `SnapshotReader.read` | `() -> MonitorView` | Build a `MonitorView` from `get_monitor_snapshot()` or `redis.snapshot_raw()`. No writes. |
| `Dashboard.refresh_view` | `() -> None` | Pull a `MonitorView`, update panels at 10 FPS. |
| `Dashboard.run` | `() -> None` | Launch the Textual app (optional extra `Jarvis-HEP2[monitor]`). |
| `attach` | `(pid) -> SnapshotReader` | Out-of-process attach: read the same Redis keys the Factory uses (Blueprint §6). |
| `build_run_summary` | `(redis) -> dict` | Project `hep:sample:stats` + factory timing into the frozen run-summary contract. |
| `validate_run_summary` | `(summary) -> None` | Assert the field set matches `RUN_SUMMARY_METRICS.md`. |

---

## 4. Data flow

```
Factory background thread → _snapshot (op_count-gated, in-memory)
        │ get_monitor_snapshot()  (in-process, <0.5 ms)
SnapshotReader ──────────────► MonitorView ──► Dashboard panels (10 FPS)
        │ redis.snapshot_raw()    (out-of-process: Jarvis2 --monitor --pid N)
run_summary (shutdown) ◄── build_run_summary(redis)  → run_summary.{json,csv,txt}
```

Because the view is reconstructable purely from Redis, the dashboard can run as a **separate
process on another host** — the in-process snapshot is just a fast path.

---

## 5. Concurrency / lifecycle / failure semantics

- **Strictly read-only**: a spy-client test asserts zero writes during a monitor session.
- 60 Hz-safe: in-process it reads memory (`get_monitor_snapshot`); out-of-process it does a
  bounded set of cheap Redis reads (LLEN + a few HGETALL gated by op_count).
- No running scan / stale snapshot → clear "no active scan" message, non-zero exit.
- Monitor failure never affects the scan (separate process / daemon thread).

---

## 6. Interfaces

- **TaskFactory.get_monitor_snapshot()** (in-process) or **RedisQueue.snapshot_raw()**
  (out-of-process).
- **client.py** → `Jarvis2 <task>.yaml --monitor` (attached) / `--monitor --pid N` (detached).
- **run_summary** consumers (frozen contract) at shutdown.

---

## 7. Tests (`tests/test_dashboard_reader.py`, `fakeredis`)

Unit (no TUI screenshots — reader logic only):
1. **MonitorView mapping** — given seeded Redis keys, `SnapshotReader.read()` returns the
   expected `MonitorView` fields.
2. **Read-only** — a monitor session issues zero Redis writes (spy client).
3. **Advancing** — bumping `op_count` + stats is reflected on the next read.
4. **run_summary contract** — `build_run_summary` output is field-for-field equal to
   `RUN_SUMMARY_METRICS.md`; `validate_run_summary` passes on a reference scan and fails on a
   missing field.
5. **No-scan** — attaching with no snapshot/keys yields the clear error path.

Verification logic: test 2 (read-only) protects the very property the retired throughput-core
monitor violated (it sat on the hot path); test 4 keeps the frozen output contract.

---

## 8. Open questions

- Particle-flow / queue-depth widget fed by `task_queue_length` history (nice-to-have).
- Prometheus / web exporter (future; Redis keys make it straightforward).
