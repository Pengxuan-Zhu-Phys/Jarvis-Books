# Component — Monitor / Dashboard (`jarvishep2/dashboard.py`, `jarvishep2/monitoring/run_summary.py`)

**Role**: a read-only view of a running scan and the run-summary writer. Reads the Factory's
op_count snapshot (or Redis directly) into a `MonitorView`, formats it, and writes
`run_summary.{json,csv,txt}` at shutdown.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `dashboard.py` 126 + `monitoring/run_summary.py` 218
lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §6; `factory_design.md`.
**Reuses V1**: none by import — the run-summary schema is frozen.

> **As-built drift:** text snapshot only (`format_monitor_view`); **no Textual TUI / `Dashboard`
> class**. The run-summary builder lives in `monitoring/run_summary.py`
> (`build_run_summary`/`RunSummaryRenderer`), not in `dashboard.py`.

---

## 1. `dashboard.py`

### `MonitorView` — `@dataclass`
**Fields:** `sampler`, `factory`, `workers`, `calculators`, `samples`, `resources`, `queues`,
`op_counts`, `timestamp`.
**Methods:** `has_active_scan()` (workers / non-empty queues / sample stats), `from_snapshot(snapshot)`
(`@classmethod`, map a Factory/Redis snapshot into the view).

### `SnapshotReader`
**Attributes:** `_source` (`TaskFactory` | `RedisQueue`). **Method:** `read() -> MonitorView`
(factory `get_monitor_snapshot()` or `redis.snapshot_raw()`; **no writes**).

### Module functions
`attach_reader(*, factory=None, redis=None) -> SnapshotReader`; `format_monitor_view(view) -> str`
(text snapshot: workers alive, queue lengths, sample stats, calculator status, op_counts,
per-worker lines); `_coerce_float`.

## 2. `monitoring/run_summary.py`

Frozen field order `RUN_SUMMARY_FIELD_ORDER` (25 fields: run_id … open_file_peak).

| Symbol | Behavior |
|--------|----------|
| helpers `_utc_iso_from_epoch`, `_coerce_float/_int`, `_safe_div`, `_mean`, `_median` | numeric/projection helpers. |
| `validate_run_summary(summary)` | raise if any frozen-contract field is missing. |
| `build_run_summary(*, factory_metrics=None, project_name, sampler_name, run_id, start_epoch, end_epoch, configured_workers, resource_summary, external_tool_time_sec, crashed_subtasks, skipped_subtasks) -> dict` | project Factory/Redis counters into the frozen schema (success_rate, throughput, framework_overhead_fraction, …); validates before returning. |
| `RunSummaryRenderer` | `render(summary) -> str` (`[Run Summary]` block) + `write_outputs(summary, output_dir, *, rendered_text=None) -> dict` (writes `run_summary.{json,csv,txt}` in field order). |

---

## 3. Data flow

```
Factory thread → op_count-gated _snapshot (in-memory)
SnapshotReader.read() → MonitorView → format_monitor_view (Jarvis2 --monitor)
core.shutdown → build_run_summary(factory.get_run_metrics()) → RunSummaryRenderer.write_outputs
```

Because the view is reconstructable from Redis, the monitor can attach out-of-process
(`Jarvis2 --monitor`) by reading the same keys.

---

## 4. Concurrency / failure semantics

- **Strictly read-only** — a monitor session issues zero Redis writes.
- In-process reads are memory (fast path); out-of-process is a bounded set of cheap Redis reads.
- No active scan → clear "No active scan found." message + non-zero exit.
- Monitor failure never affects the scan.

---

## 5. Interfaces / collaborators

- **TaskFactory.get_monitor_snapshot()** ([factory.md](factory.md)) /
  **RedisQueue.snapshot_raw()** ([redis_queue.md](redis_queue.md)).
- **client.py** ([cli.md](cli.md)) → `Jarvis2 --monitor`.
- **core.write_run_summary** ([core.md](core.md)) → `build_run_summary` + `RunSummaryRenderer`.

---

## 6. Tests

`tests/test_dashboard_reader.py` (6), `tests/test_monitor_snapshot.py` (5): MonitorView mapping,
read-only invariant, advancing snapshot, run_summary contract field-for-field, no-scan path.
