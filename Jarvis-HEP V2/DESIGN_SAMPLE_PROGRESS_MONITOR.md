# DESIGN — Active Sample Progress for Monitor (low overhead)

**Status**: design proposal (not implemented)  
**Date**: 2026-07-16  
**Goal**: Let the monitor show *which* sample is active and *which workflow phase* it is in (calculator / opera / likelihood / archive handoff), **without** slowing the scan hot path.

---

## 1. Problem

Today Redis monitor data is:

| Source | What monitor sees |
|--------|-------------------|
| `hep:sample:stats` | global `running` / `completed` / `failed` only |
| `hep:worker:status:{id}` | `current_sample`, full `current_task`, `held_calc_packs`, PIDs |
| queues | task / archive lengths |

There is **no first-class phase** such as “this uuid is on calculator EggBox” vs “on Likelihood”.  
Operators can only *infer* from held packs / busy status. That is enough for recovery, not for a pipeline UI.

---

## 2. Non-goals

- Per-sample **history** in Redis (no timeline of every past sample).
- Storing observables / full task again under a second key.
- Sub-millisecond phase accuracy.
- Any dependency of scan correctness on this channel (best-effort only).

---

## 3. Design principle

> **Piggyback on the existing Worker heartbeat.**  
> Do **not** open a new per-sample Redis key space that must be scanned at 120 Hz.

Rationale:

- Heartbeat already runs (~5 s by default) and already `HSET`s `hep:worker:status:{id}`.
- One Worker ≡ one in-flight sample (V2 invariant).
- Monitor already refreshes worker heartbeats when `hep:worker:op_count` moves.
- Extra fields on the same hash ≈ **zero extra round-trips** on the steady state.

Phase updates are **telemetry**, not control plane: failure to write must never fail the sample.

---

## 4. Recommended shape

### 4.1 Worker-local progress (source of truth while running)

In `Worker` (memory only):

```text
_sample_progress: {
  uuid: str
  phase: "idle" | "start" | "calculator" | "opera" | "likelihood" | "submit" | "done"
  module: str | ""          # e.g. "EggBox", "LogL", ""
  layer: int | null
  step_index: int | null    # 0-based in execution_plan
  step_total: int | null
  pack_id: str | ""         # if holding a calc slot
  updated_ts: float
}
```

Update **only at coarse boundaries** (cheap; pure memory):

| When | `phase` | `module` |
|------|---------|----------|
| `process_task` begins | `start` | `""` |
| before `_run_calculator_step` | `calculator` | calc name |
| before `_run_opera_step` | `opera` | opera name |
| before `_run_likelihood` | `likelihood` | `"LogL"` or expression name |
| before `_stage_and_submit` | `submit` | `""` |
| `finally` after clear | `done` / clear struct | |

Do **not** update on every Portal Dump / every subprocess log line.

### 4.2 Heartbeat fields (Redis export)

Add small string fields to the existing heartbeat `HSET` (same call as today):

| Field | Example | Notes |
|-------|---------|--------|
| `sample_phase` | `calculator` | enum string |
| `sample_module` | `EggBox` | current step name |
| `sample_layer` | `1` | optional |
| `sample_step` | `2/5` | compact `index/total` |
| `sample_pack` | `001` | optional duplicate of held pack |

Keep **`current_sample`** as uuid (already present).  
Do **not** re-encode full `current_task` more often than today.

Steady-state cost: **0 extra Redis commands** — only a few more hash fields on the periodic heartbeat.

### 4.3 Optional “snappy” write (still cheap)

If monitor should update **within ~100 ms of a phase change** instead of waiting for the next 5 s beat:

- On phase change only: `HSET hep:worker:status:{id}` with the progress fields **only** (not full task).
- Still best-effort; swallow errors.
- Budget: **O(steps per sample)** Redis writes, typically **&lt; 10 / sample**.  
  Vs. one external calculator subprocess: negligible.

Default recommendation:

- **v1 implement: heartbeat-only** (simplest, zero extra RTT).
- **v1.1 optional**: phase-change HSET behind `Runtime.Factory.monitor.progress_eager: true`.

### 4.4 What we deliberately do *not* do

| Approach | Why skip |
|----------|----------|
| `hep:sample:active:{uuid}` + `SCAN` | SCAN is O(N); key churn; monitor 120 Hz unfriendly |
| Write phase on every layer tick / expression eval | Hot-path spam |
| Store full observables in progress hash | Size + encode cost |
| Block sample pipeline on monitor Redis errors | Violates “telemetry only” |
| Archiver per-uuid phase in Redis | Archive is already on `archive_queue` length + Archiver logger |

Queued samples remain visible only as **task_queue_length** (payload still on the list; no per-uuid phase until a Worker pulls).

---

## 5. Monitor / dashboard surface

Extend `MonitorView` / `format_monitor_view` (and any TUI later):

```text
worker 3: busy sample=005e6ab3… phase=calculator module=EggBox step=1/3 pack=001
worker 5: busy sample=a91c…     phase=likelihood module=LogL step=3/3
worker 1: idle
sample_stats: running=2 completed=1204 failed=0
```

Implementation notes:

- Read path: already fetches worker heartbeats; just **parse new fields**.
- No new Redis read pattern required.
- Snapshot rate stays gated by existing `op_count` logic.

---

## 6. Performance budget

| Event | Extra cost |
|-------|------------|
| Phase boundary (memory) | nanoseconds |
| Heartbeat (default 5 s) | +~100–200 bytes hash fields; same 1 pipeline as today |
| Eager phase HSET (optional) | 1 `HSET` per step ≈ µs on local Redis |
| Monitor refresh | unchanged (same keys) |

Scan path (subprocess, Portal IO, FileOperation, likelihood) is **not** on this critical path beyond a dict assign + optional tiny HSET.

---

## 7. Lifecycle & consistency

```
pull_task → running++ 
  Worker sets progress=start
  … calculator / opera / likelihood …
  submit_result → archive_queue; running--
  progress cleared; heartbeat shows idle / no current_sample
```

After sample ends:

- Progress fields empty or `phase=idle`.
- **No** leftover per-uuid key (because we never created one).
- Global stats + archive queue remain the post-run signals.

Stale progress: if Worker crashes, watchdog already uses heartbeat TTL; progress dies with the heartbeat hash refresh / worker death.

---

## 8. Config (optional)

```yaml
# EnvReqs.V2 / Runtime.Factory (sketch)
Factory:
  monitor:
    # default: only attach progress on existing heartbeats
    sample_progress: true
    # optional snappy updates on phase change
    progress_eager: false
```

`sample_progress: false` → zero behavior change (no extra fields).

---

## 9. Implementation sketch (small PR)

1. `Worker`: `_sample_progress` + `_set_sample_progress(...)` at layer boundaries.  
2. `_heartbeat`: include progress fields.  
3. `dashboard` / `MonitorView`: display `phase` / `module` / `step`.  
4. Tests: unit test that heartbeat mapping contains phase after fake step; no Redis load test required for v1.  
5. Docs: one paragraph in `components/monitor.md` + `worker.md`.

Optional follow-up: `progress_eager` HSET helper on RedisQueue.

---

## 10. Answer to “能不能做到？”

**Yes.**  
Enough fidelity for monitor UI; **heartbeat piggyback** keeps scan performance effectively unchanged.  
Do **not** build a full per-sample Redis state machine unless later product needs multi-consumer live tracking beyond “one sample per worker”.
