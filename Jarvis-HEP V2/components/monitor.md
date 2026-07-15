# Component — Monitor / Dashboard (`jarvishep2/dashboard.py`, `jarvishep2/monitoring/run_summary.py`)

**Role**: a read-only view of a running scan and the run-summary writer. Reads the Factory's
op_count snapshot (or Redis directly) into a `MonitorView`, formats it, and writes
`run_summary.{json,csv,txt}` at shutdown. Emits a console **[Scan Performance]** block.
**Status**: **As-built** @ `jarvis2` **`4e562e0`** (performance metrics + log).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §6;
[`../DESIGN_SAMPLE_PROGRESS_MONITOR.md`](../DESIGN_SAMPLE_PROGRESS_MONITOR.md) (proposed sample phase telemetry).
**Reuses V1**: none by import — the run-summary schema is versioned via `RUN_SUMMARY_FIELD_ORDER`.

> **As-built drift:** text snapshot only (`format_monitor_view`); **no Textual TUI / `Dashboard`
> class**. The run-summary builder lives in `monitoring/run_summary.py`
> (`build_run_summary`/`RunSummaryRenderer`/`format_scan_performance_log`), not in `dashboard.py`.

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

### What Redis exposes for “active samples” today

| Source | Content |
|--------|---------|
| `hep:sample:stats` | global `running` / `completed` / `failed` only |
| `hep:worker:status:{id}` | `current_sample`, full `current_task`, `held_calc_packs`, subprocess PIDs, status |
| queues | task / archive lengths |

There is **no** per-uuid phase state machine in Redis yet. See
[`DESIGN_SAMPLE_PROGRESS_MONITOR.md`](../DESIGN_SAMPLE_PROGRESS_MONITOR.md) for the proposed
heartbeat-piggybacked `sample_phase` / `sample_module` fields (zero hot-path cost).

---

## 2. `monitoring/run_summary.py`

Frozen field order `RUN_SUMMARY_FIELD_ORDER` (includes throughput + latency fields).

| Symbol | Behavior |
|--------|----------|
| helpers `_utc_iso_from_epoch`, `_coerce_*`, `_safe_div`, `_mean`, `_median` | numeric/projection helpers. |
| `validate_run_summary(summary)` | raise if any frozen-contract field is missing. |
| `build_run_summary(...)` | project Factory/Redis counters into the schema. |
| `format_scan_performance_log(summary) -> str` | human `[Scan Performance]` two-column block. |
| `RunSummaryRenderer` | `render` + `write_outputs` → `run_summary.{json,csv,txt}`. |

### Throughput / latency fields (as-built 2026-07-16)

| Field | Meaning |
|-------|---------|
| `wall_time_sec` | end − start |
| `samples_per_sec` | finished / wall_time |
| `samples_per_min` | finished × 60 / wall_time |
| `throughput_points_per_min` | alias of `samples_per_min` (compat) |
| `avg_sample_sec` | wall_time / finished (**amortized** system latency) |
| `avg_point_eval_sec` / `median_point_eval_sec` | from Worker duration list when present; else = `avg_sample_sec` |

`core.write_run_summary` logs `format_scan_performance_log` at WARNING on the main
`Jarvis-HEP` logger and INFO-prints the written file paths.

---

## 3. Data flow

```
Factory thread → op_count-gated _snapshot (in-memory)
SnapshotReader.read() → MonitorView → format_monitor_view (Jarvis2 monitor)
core.shutdown → write_run_summary:
  build_run_summary(factory.get_run_metrics())
  → log [Scan Performance]
  → RunSummaryRenderer.write_outputs → outputs/<scan>/run_summary.{json,csv,txt}
```

Because the view is reconstructable from Redis, the monitor can attach out-of-process
(`Jarvis2 monitor`) by reading the same keys.

---

## 4. Plot CSV (related output)

`plot_scene.export_samples_csv_from_hdf5` writes `DATABASE/samples.csv` as a **full** export of
HDF5 JSON records (all columns; nested values as JSON text). Not a thin x/y/LogL-only table.
jplot may still select `x`/`y`/`LogL` via expressions.

---

## 5. Concurrency / failure semantics

- **Strictly read-only** for live monitor — zero Redis writes.
- In-process reads are memory (fast path); out-of-process is a bounded set of cheap Redis reads.
- No active scan → clear "No active scan found." message + non-zero exit.
- Monitor / run_summary failure never aborts sample correctness (best-effort log).

---

## 6. Interfaces / collaborators

- **TaskFactory.get_monitor_snapshot()** ([factory.md](factory.md)) /
  **RedisQueue.snapshot_raw()** ([redis_queue.md](redis_queue.md)).
- **client.py** ([cli.md](cli.md)) → `Jarvis2 monitor`.
- **core.write_run_summary** ([core.md](core.md)) → performance log + files.

---

## 7. Tests

`tests/test_dashboard_reader.py`: MonitorView mapping, run_summary contract (incl.
`samples_per_sec` / `avg_sample_sec`), performance block in txt, no-scan path.
`tests/test_monitor_snapshot.py`: advancing snapshot / op_count gating.
