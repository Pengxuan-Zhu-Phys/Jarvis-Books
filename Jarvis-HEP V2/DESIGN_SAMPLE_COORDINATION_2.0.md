# DESIGN — Sample Coordination over Redis (V2)

**Status**: **accepted design** (v1.0)  
**Date**: 2026-07-16  
**Principle**: **Occam’s razor** — Redis is the core coordination fabric of Jarvis;  
**do not add entities** that are not required for a concrete consumer.  
**Identity**: **`uuid` is the only cross-system primary key** for a Sample.

---

## 1. Goal

Define the **minimum** Redis surface for coordinating Samples among Sampler, Worker,
Archiver, and (optional) Monitor — without turning Redis into a second archive or a
full observables bus.

---

## 2. Non-goals

- Storing full `observables`, SLHA, file paths, or product lists in Redis for sampler/monitor.
- A permanent per-uuid sample catalog for ~1M points.
- A parallel “Sample Registry” stack (Hash + Stream + ZSET + stage sets) as a default.
- Replacing `hep:archive_queue` / Archiver / SAMPLE / HDF5.
- Always-on status or always-on logL writes (both are **opt-in**).

---

## 3. Occam partition (three concerns only)

| Concern | Channel | Payload | When written |
|---------|---------|---------|--------------|
| **Propose** | existing task queue | **`uuid` + `u_coords`** (+ plan metadata already on the task) | always (sampler → worker) |
| **Sampler feedback** | existing feedback queue (or equivalent single list/stream) | **`uuid` + `logL` only** | **only if** that sampler requires feedback |
| **Monitor stage** | worker heartbeat fields *or* one tiny status key — **not** feedback | **`uuid` + stage** (via current worker slot) | **only if** monitor switch is on |
| **Persist** | existing archive queue | full sample (`to_info_dict` / observables) | always on completion |

No fourth bus. No observables on feedback. No `status` on feedback. No `u_coords` on feedback
(sampler already issued them with the task and can map `uuid → u` locally).

---

## 4. Identity and matching

```text
uuid  ── unique Sample id across Sampler / Worker / Archiver / Monitor / disk
```

| Direction | Identity + data |
|-----------|-----------------|
| Sampler → Worker (task) | **`uuid` + `u_coords`** (+ execution_plan / metadata as today) |
| Worker → Sampler (feedback) | **`uuid` + `logL`** |
| Worker → Monitor (optional) | **`uuid`** implied by worker’s current sample + **stage** |
| Worker → Archiver | **`uuid`** + full cold payload |

Join rule everywhere: **match on `uuid` only**. Do not re-ship `u_coords` or observables to
“help” the sampler; the sampler already owns the map for proposals it submitted.

---

## 5. Channel contracts

### 5.1 Task (existing) — always on

- Key: `hep:task_queue` (current design).
- Minimum logical fields for coordination: **`uuid`**, **`u_coords`**.
- Rest of the task dict (execution_plan, sample_artifacts, …) stays as today.

### 5.2 Feedback (existing queue, **schema-hardened**) — opt-in

- Key: reuse **`hep:feedback`** (do **not** invent a second stream for the same purpose).
- Enable only when the active sampler is feedback-driven (e.g. MCMC, AdaptiveLevelSet).
- **Payload (hard max)**:

```json
{ "uuid": "<string>", "logL": <float> }
```

- **Forbidden** on feedback: `status`, `observables`, `u_coords`, paths, stage, worker_id.
- **Failed samples**: do **not** put `status` on feedback. Sampler handles loss via its own
  pending-`uuid` set + timeout / generation barrier (as ALS already does with timeouts).
  Optionally later: a separate minimal failed-uuid signal — **only if** a concrete sampler
  proves timeout is insufficient. Default: no extra entity.
- **logL source**: Worker Likelihood step → `observables["LogL"]` or `sample_info["likelihood"]`,
  projected to the scalar field only.
- Worker write order: **`submit_result` (archive) first**; feedback is best-effort and must
  never block or replace archive.

### 5.3 Monitor sample stage — opt-in, separate from feedback

- **Not** part of feedback.
- **Default off** (zero cost on the hot path).
- Switch (one of, prefer single source of truth):
  - Redis flag e.g. `hep:monitor:sample_stage` = `0|1`, or
  - YAML `Runtime.Monitor.sample_stage: false` stamped into worker config at start.
- When **off**: Worker does not write per-sample stage to Redis.
- When **on**: Worker updates stage only at **coarse** boundaries
  (e.g. start / calculator / opera / likelihood / submit).
- **Preferred storage (Occam)**: piggyback on existing **`hep:worker:status:{id}`**
  heartbeat fields:

```text
current_sample = <uuid>
sample_stage   = start|calculator|opera|likelihood|submit|...
```

  One Worker ≡ one in-flight sample ⇒ **no new `hep:sample:{uuid}` key space** for the
  default monitor path.

- **Avoid** unless proven necessary: `hep:sample:status:{uuid}` Hash per sample (key churn
  at high QPS). If ever added, TTL + monitor-off must delete/stop writes.

### 5.4 Archive (existing) — always on for finished work

- Key: `hep:archive_queue`.
- Full cold record (observables, LogL, paths, …) for DATABASE / SAMPLE.
- Unrelated to sampler feedback shape.

### 5.5 Aggregate stats (existing)

- `hep:sample:stats` (`running` / `completed` / `failed`) stays.
- Do not duplicate into a new stats hash for the same counters.

---

## 6. What we explicitly reject (Draft Registry v0.1 cleanup)

| Proposed entity | Decision |
|-----------------|----------|
| Default `jarvis:{project}:sample:{uuid}` Hash with params+logL+stage | **Reject** as default |
| Default ZSET of all buffer logL | **Reject** as default (add only if a sampler needs top-k buffer) |
| Feedback carrying `status` | **Reject** |
| Feedback carrying `u_coords` | **Reject** (task already had them) |
| Feedback / monitor carrying `observables` | **Reject** |
| Parallel Stream *and* List for the same feedback | **Reject** — one queue |
| Namespace multi-tenant keys as first-class now | **Defer** — isolation today is Redis instance + `hep:*` + control lock |

---

## 7. Configuration (YAML sketch)

```yaml
# Conceptual — map into EnvReqs.V2 / Runtime when implemented
Runtime:
  SamplerFeedback:
    enabled: false          # true only for feedback-driven samplers
  Monitor:
    sample_stage: false     # true only while an interactive monitor needs stage
  Checkpoint:
    interval_sec: 30        # allowed range e.g. 30 .. 3600 (or up to 3600*60 if desired)
```

- Bridson / Grid / Random: `SamplerFeedback.enabled=false`, `Monitor.sample_stage=false`.
- MCMC: feedback on; payload still only `{uuid, logL}`.
- Checkpoint interval is independent; checkpoint stores **completed-sample / control-state**
  facts, not Redis hot stage spam.

---

## 8. Lifecycle (minimal)

```text
Sampler:  mint uuid, bind u_coords → push task
Worker:   pull task → compute → Likelihood → logL scalar
          → submit_result (full cold payload)
          → if SamplerFeedback: publish {uuid, logL}
          → if Monitor.sample_stage: heartbeat current_sample + sample_stage
Sampler:  (if feedback) pull {uuid, logL} → join local state by uuid → drop message
Archiver: drain archive → HDF5 / SAMPLE
Monitor:  (if stage on) read worker heartbeats; else only stats/queues
```

---

## 9. Performance implications

| Mode | Extra Redis writes per sample |
|------|-------------------------------|
| Stateless sampler, monitor stage off | **0** beyond task/archive/stats (current path) |
| Feedback on | **+1** small push `{uuid, logL}` |
| Monitor stage on | **amortized** into existing heartbeat (no per-phase mandatory HSET unless eager) |

This preserves Redis as the core fabric without a permanent 1M-key sample graph.

---

## 10. Implementation notes (when coding)

1. Harden `publish_feedback` / ALS consumer to **`{uuid, logL}`** only (migrate ALS target
   evaluation to use local `uuid→u/x` + returned `logL` or keep target on worker-side cold
   path — ALS may need a short follow-up if target ≠ LogL; do not re-expand feedback).
2. Gate feedback with sampler capability / worker_config flag (already `publish_feedback`).
3. Gate stage with monitor flag; prefer heartbeat fields over new keys.
4. Do **not** add `SampleRegistry` class with multi-structure defaults until a sampler
   needs top-k ZSET; then add **one** optional structure, not a suite.
5. Checkpoint interval: YAML `interval_sec` only; content = completed work + sampler state.

### ALS caveat

AdaptiveLevelSet today consumes `observables` from feedback for a **general target
expression**. Under this design, either:

- target is **`LogL`** (or a function of logL only) → fits `{uuid, logL}`, or  
- target needs other symbols → evaluate target **on the Worker** and feedback a single
  scalar `f` (still not a full observables dict), e.g. `{uuid, value}` with a fixed name.

Do not reintroduce full observables “for ALS convenience.”

---

## 11. Relationship to other docs

| Doc | Relation |
|-----|----------|
| [`DESIGN_SAMPLE_PROGRESS_MONITOR.md`](DESIGN_SAMPLE_PROGRESS_MONITOR.md) | Stage-on-heartbeat when monitor needs it — **same** opt-in stage idea; no observables |
| Earlier “Sample Registry” draft | **Superseded** by this document’s Occam partition |
| [`components/redis_queue.md`](components/redis_queue.md) / [`monitor.md`](components/monitor.md) | As-built keys; update when implementing gates |

---

## 12. Acceptance criteria

- [ ] Stateless scans: no feedback writes; no sample-stage writes when switches off.
- [ ] Feedback payload schema is exactly `{uuid, logL}` (types enforced in code).
- [ ] Feedback never includes `status` / `u_coords` / `observables`.
- [ ] Monitor stage never uses the feedback queue.
- [ ] uuid is the only join key documented and used in sampler feedback handling.
- [ ] Full observables only on archive path.

---

## 13. One-line summary

> **Task = `uuid` + `u_coords`; feedback = `uuid` + `logL`; monitor stage = separate opt-in;
> full sample = Archiver only; uuid is the sole identity.**
