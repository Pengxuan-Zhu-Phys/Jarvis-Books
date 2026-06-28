# Component — Checkpoint / Resume (`jarvishep2/Sampling/Source/MCMC/runtime_checkpoint.py`)

**Role**: save and restore enough state to resume a scan, under the distributed model. The
Sampler (control process) remains the source of truth; Redis queues are volatile and re-driven
on resume.
**Status**: design — plan WP-D6.2.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §10;
`../../v1/design/DESIGN_CHECKPOINT_1.7.0.md` (payload minimalism, still governs),
`../../v1/design/STATE_SAVER_RESUME_DESIGN.md`.
**Reuses V1**: the checkpoint payload structure + 30 s heartbeat + atomic `os.replace` write +
resume-prompt UX (all frozen, invariant #4).

---

## 1. Responsibilities

1. Persist **sampler state** (RNG/`SeedSequence`, chains, ready queue) at safe barriers — from
   the control process, unaffected by Redis.
2. On resume: rebuild Redis pools from config, **drain** stale `hep:task_queue`, **never**
   restore in-flight Samples, and let the Sampler re-propose.
3. Wait for **Archiver ack** of consumed batches before declaring a safe barrier (so resumed
   runs don't lose or double-write persisted Samples).
4. Keep checkpoint UX frozen: 30 s heartbeat, `state.pkl` location, resume prompt wording,
   `--resume`.
5. Make reproducibility **independent of Worker count** via master `SeedSequence` + child
   streams.

---

## 2. Structure (delta over V1 payload)

```python
PAYLOAD = {
  "format": "jarvis-hep.v2-distributed",
  "version": 1,
  "run_spec": {...},                  # raw yaml, normalized config, scan_name, task_root
  "sampler_state": {                  # control-process truth
     "numpy_random_state": ...,
     "seed_sequence": {...},          # master SeedSequence (worker-count independent)
     "chains": [...], "ready_queue": [...], "control_state": {...},
  },
  "persistence": {                    # NEW: distributed safe-barrier
     "acked_uuids_highwater": ...,    # what the Archiver confirmed persisted
     "next_sample_index": ...,
  },
  "integrity": {"config_hash": ..., "variable_signature": ..., "safe_barrier_confirmed": True},
}
```

No `hep:task_queue` contents are serialized — in-flight is discarded on resume by design.

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `build_payload` | `(sampler, persistence) -> dict` | Assemble the payload from sampler state + Archiver ack high-water. |
| `save_atomic` | `(payload, path) -> None` | Write temp + `os.replace` (atomic; V1). |
| `load` | `(path) -> dict` | Read + validate `format`/`version`/`config_hash`; **refuse V1 + throughput-core formats** with an actionable message. |
| `safe_barrier_ready` | `(sampler, archiver_ack) -> bool` | True when sampler is at a barrier **and** the Archiver has acked all submitted batches. |
| `resume` | `(payload, redis) -> None` | Rebuild pools, drain `hep:task_queue`, restore sampler state, set re-propose hint. |
| `derive_worker_seed` | `(master_seq, index) -> SeedSequence` | Child stream per Sample/Worker — reproducibility decoupled from Worker count. |

---

## 4. Safe barrier under the distributed model

```
heartbeat (30 s) →
   if sampler at barrier AND archiver acked all submitted batches:
        payload = build_payload(...)
        save_atomic(payload, state.pkl)
```

- **In-flight = never serialized** (decision carried from V1): on resume the Sampler re-proposes
  whatever wasn't acked. Combined with idempotent archiving (Sample present ⇒ skip), this gives
  no-missing / no-duplicate UUIDs.
- The barrier now has **two** conditions: sampler-consumed **and** Archiver-acked.

---

## 5. Concurrency / failure semantics

- Save runs under the existing `_runtime_checkpoint_save_lock` (V1, correctly initialized in
  `SamplingVirtial.__init__`).
- A crash between submit and ack → resume re-proposes those Samples; the Archiver's
  present-`SAMPLE/<uuid>/` short-circuit prevents duplicates.
- Worker-count change between save and resume is safe because seeds are derived from the master
  `SeedSequence`, not from Worker indices directly.
- Resume **refuses** V1 (`jarvis-hep.statesaver`) and throughput-core (`jarvis-hep.v2`)
  checkpoints with a clear message — no silent cross-format load.

---

## 6. Interfaces

- **Sampler** → `export_runtime_state` / `import_runtime_state` feed the payload.
- **Archiver** → ack high-water for the safe barrier.
- **RedisQueue** → drained + pools rebuilt on resume.
- **core._preload_resume_checkpoint** → resume orchestration + prompt UX.

---

## 7. Tests (`tests/test_distributed_resume.py`, `fakeredis`)

Unit / integration:
1. **Round-trip** — save at a barrier, resume → identical sampler state; scan continues.
2. **No dup / no missing** — kill mid-scan, resume → final golden UUID set is exactly complete
   (drain + idempotent archive).
3. **Worker-count independence** — seeded rerun with 1 vs 4 Workers reproduces the same chain
   trajectory (master SeedSequence + child streams).
4. **Format refusal** — a V1 / throughput-core `state.pkl` is refused with an actionable error.
5. **Two-condition barrier** — a barrier is not taken until the Archiver acks (assert the save
   waits on ack in a fixture).
6. **UX frozen** — 30 s heartbeat cadence + resume prompt wording unchanged (assert on the
   prompt string).

Verification logic: tests 2/3 are the science-safety gates (no lost/duplicate points,
reproducible regardless of how many Workers ran); test 4 prevents corrupt cross-format resume.

---

## 8. Open questions

- Optional Redis RDB/AOF durability vs. pure file-based re-drive (design §13.5 — default:
  file-based + drain).
- Checkpoint size/time budget on large chains (target ~1–5 MB, ~2–3 s load, as V1).
