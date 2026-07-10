# Component — Checkpoint / Resume (`jarvishep2/Sampling/runtime_checkpoint.py`, `jarvishep2/Sampling/checkpointed_sampler.py`)

**Role**: save and restore enough state to resume a scan under the distributed model. The Sampler
(control process) is the source of truth; Redis queues are volatile and re-driven on resume.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `runtime_checkpoint.py` 304 + `checkpointed_sampler.py`
96 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §10.
**Reuses V1**: none by import — frozen UX (30 s heartbeat, atomic write, resume prompt) reimplemented.

---

## 1. `runtime_checkpoint.py`

Constants: `CHECKPOINT_FORMAT = "jarvis-hep.v2-distributed"`, `CHECKPOINT_VERSION = 1`,
`CHECKPOINT_HEARTBEAT_SEC = 30.0`, `RESUME_PROMPT`, and `REFUSAL_MESSAGES` for the **rejected**
formats `jarvis-hep.statesaver` (V1), `jarvis-hep.mcmc-runtime` (V1), `jarvis-hep.v2`
(retired throughput-core).

### Class — `CheckpointHeartbeat`
30 s background save thread. **Attributes:** `_interval_sec`, `_save_callback`, `_stop` (Event),
`_thread`. **Methods:** `start()`, `stop()`, `_loop()` (calls `save_callback(reason=
"checkpoint_heartbeat")` each interval).

### Module-level functions
| Function | Behavior |
|----------|----------|
| `utc_now_iso()` | UTC ISO timestamp. |
| `atomic_pickle_dump(path, payload)` / `pickle_load(path)` | temp-write + `os.replace`; load + type-check. |
| `_json_safe(value)` / `stable_json_hash(payload)` | JSON-safe coercion + sha256 (config hash). |
| `serialize_seed_sequence` / `deserialize_seed_sequence` | round-trip a `np.random.SeedSequence`. |
| `derive_sample_seed(master, sample_index)` | per-Sample child stream (Worker-count independent). |
| `checkpoint_path(*, task_root, scan_name, sampler_name)` | `…/checkpoints/<scan>/<sampler>/state.pkl`. |
| `build_run_spec(...)` | normalized config + scan/task metadata + worker_parallel. |
| `build_payload(*, run_spec, sampler_state, persistence=None, reason="", safe_barrier_confirmed=True)` | the full checkpoint payload (format/version/integrity/config_hash/variable_signature). |
| `validate_checkpoint_payload(payload)` | accept current format/version; **refuse** V1 / throughput-core with actionable messages. |
| `load_checkpoint(path)` / `save_checkpoint(path, payload)` | validate + pickle load/atomic save. |
| `safe_barrier_ready(*, sampler_at_barrier, submitted_uuids, archiver_persistence)` | barrier = sampler at rest **and** all submitted uuids acked by the Archiver. |
| `prepare_resume(redis, *, worker_config)` | drain stale `hep:task_queue`, rebuild calc pools; return drained count. |

---

## 2. `checkpointed_sampler.py` — `CheckpointedSampler(SamplingVirtial)`

Mixes checkpoint heartbeat + resume repropose into samplers. **Attributes:**
`_runtime_checkpoint_save_lock` (RLock), `_submitted_uuids`, `_completed_uuids`, `_checkpoint_file`,
`_checkpoint_heartbeat`, `_save_checkpoint_callback`, `_repropose_after_resume`.

| Method | Behavior |
|--------|----------|
| `configure_checkpoint(*, checkpoint_file, save_callback)` | wire the heartbeat (force-save each tick). |
| `shutdown_checkpointing()` | stop the heartbeat. |
| `submitted_uuids` (`@property`) | frozenset of submitted uuids. |
| `mark_completed(uuid)` | record an archived uuid. |
| `set_resume_repropose_hint(enabled=True)` | enable post-resume repropose. |
| `_checkpoint_control_state` / `_import_checkpoint_control_state` | (de)serialize submitted/completed + control flags. |
| `persist_runtime_checkpoint(*, force=False, reason="", archiver_persistence=None)` | save (gated by `safe_barrier_ready` unless `force`); lock-guarded. |
| `at_safe_barrier()` | abstract — each sampler defines its barrier. |

---

## 3. Safe-barrier model & format refusal

The barrier has **two** conditions: the sampler is at rest **and** the Archiver has acked all
submitted uuids. In-flight tasks are **never** serialized; on resume the sampler re-proposes
whatever wasn't acked (combined with idempotent archiving → no missing / no duplicate). Resume
**refuses** V1 / throughput-core checkpoints with a clear message.

---

## 4. Interfaces / collaborators

- **Samplers** ([samplers_catalog.md](samplers_catalog.md)) implement `at_safe_barrier` +
  `export/import_runtime_state`; `SeededOperaSampler` carries its own copy of these hooks.
- **core** ([core.md](core.md)) `save_runtime_checkpoint` / `_preload_resume_checkpoint` /
  `apply_resume_checkpoint` / `prompt_resume_from_checkpoint`.
- **Archiver** ([datarecorder.md](datarecorder.md)) supplies `persistence_state` (acked uuids).
- **RedisQueue** ([redis_queue.md](redis_queue.md)) drained + pools rebuilt on resume.

---

## 5. Tests

`tests/test_distributed_resume.py` (8): save/resume round-trip, no-dup/no-missing after kill,
Worker-count independence (master SeedSequence), format refusal, two-condition barrier, frozen UX.
