# Component — DataRecorder / Archiver (`jarvishep2/archiver.py`, `jarvishep2/hdf5writer.py`)

**Role**: the final persistence layer (Layer 2 I/O). Consumes finished Samples from Workers,
batches them, and writes `outputs/<scan>/SAMPLE/<uuid>/` + `DATABASE/` (HDF5/CSV) with
NAS-optimized file moves. Decoupled from calculator execution.
**Status**: design — plan WP-D4.1 (handoff + skeleton) → D4.2 (batch persistence + parity).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §5; discussions
`Jarvis-HEP_Discussion_Summary_2026-06-21.md` §2/§3, `command_execution_design.md`.
**Reuses V1**: `GlobalHDF5Writer` (`jarvishep/hdf5writer.py`), `observable_io` (schema/CSV), and
**subsumes `SampleArchiveManager`** (`jarvishep/Sampling/sample_archive.py` — per-bucket tar +
`.jsonl` index). The output **format** is frozen (invariant #3); the Archiver changes only the
**transport**.

> **Subsumes `SampleArchiveManager`.** V1's bucket-tarring manager (`_archive_worker`,
> `enqueue_bucket_dir`, tar + jsonl-count dedup) becomes part of the V2 Archiver's batch step:
> staged `SAMPLE/<bucket>/` dirs are tarred into the DATABASE root exactly as in V1. The V1 race
> (known bug #13, `sample_archive.py:179-180` non-atomic check-then-add) is **fixed** here by the
> Archiver's single-consumer model + idempotent `uuid`-keyed dedup (invariant #14).

---

## 1. Responsibilities

1. Consume `info_dict` + staging paths from `hep:archive_queue` (Worker handoff).
2. **Batch** ≈`batch_size` finished Samples (default 200) and persist together.
3. **Move** products `staging/<uuid> → SAMPLE/<uuid>/` (NAS: `os.rename`/`shutil.move`, not
   copy); write `DATABASE/` records via the HDF5 writer; CSV via `observable_io`.
4. **Format**: xlha/slha formatting where configured.
5. **Cleanup**: delete staging after archive (`delete_method`: shutil|rm).
6. Acknowledge persistence so the Sampler's checkpoint safe-barrier can advance (D6.2).

Two pieces:
- **`Archiver(Process)`** — orchestrates batching, moves, cleanup (NAS-aware, async I/O).
- **`hdf5writer`** — the actual HDF5/CSV record writer (reused V1 `GlobalHDF5Writer`, optionally
  run as its own process fed by the Archiver).

---

## 2. Class structure

```python
class Archiver(multiprocessing.Process):
    def __init__(self, redis_config, db_config, archiver_config, name="HEP2-Archiver"):
        # picklable config only (spawn)
        ...
    # built in run():
    redis:  RedisQueue
    writer: GlobalHDF5Writer
    batch:  list[dict]            # pending info_dicts
    staging_paths: list[str]

class ArchiverConfig:             # from YAML Calculators.Archiver
    batch_size: int = 200
    async_io: bool = True
    strategy: str = "move"        # move|copy
    fmt: str = "xlha"
    nas_optimize: bool = True
    delete_after_archive: bool = True
    delete_method: str = "shutil" # shutil|rm  (Runtime.FileOperation)
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `run` | `() -> None` | Process entry: connect Redis, init writer, loop `pull_result`. |
| `_loop` | `() -> None` | `while running:` `pull_result(timeout=1)`; append to `batch`; flush when `len>=batch_size` or a flush interval elapses. |
| `archive_batch` | `(infos, staging) -> None` | Async-move products → `SAMPLE/`, write DATABASE records, format, then delete staging. |
| `_move_products` | `(uuid, staging_path) -> None` | `os.rename`/`shutil.move` (same-volume NAS); fall back to copy+delete across volumes. |
| `_write_records` | `(infos) -> None` | Hand records to `GlobalHDF5Writer.add_data` (+ schema update via `observable_io`). |
| `_format_products` | `(uuid) -> None` | xlha/slha formatting when configured. |
| `_cleanup` | `(staging_paths) -> None` | `delete_path(p, method=delete_method)` for each. |
| `_ack` | `(uuids) -> None` | Mark persisted (advances checkpoint safe barrier). |
| `flush` / `stop` | `() -> None` | Drain pending batch; finalize HDF5 file; close. |

`hdf5writer` (reused, plus a thin process front-end):

| Method | From | Behavior |
|--------|------|----------|
| `add_data(record)` | V1 | enqueue one record to the writer thread/process. |
| `_flush_batch_to_hdf5` / `_rotate_if_needed` | V1 | write + rotate files. |
| `hdf5_to_csv()` | V1 | `--convert` CSV export (frozen format). |
| `start` / `stop` | V1 | lifecycle. |

---

## 4. Two-layer I/O (the design line)

```
Worker (Layer 1, synchronous, per-Sample):   input.json → run calc → output.json
   └─ on completion:  mv work_dir → staging/<uuid>     (fast, metadata-only)
   └─ submit_result(info_dict, staging_path) → hep:archive_queue
Archiver (Layer 2, batched, async):           staging/<uuid> → SAMPLE/<uuid>/ ; DATABASE/
```

The Worker is never blocked on final persistence — it stages (instant) and moves on. The
Archiver absorbs the NAS cost in batches off the critical path.

---

## 5. Concurrency / ordering / failure semantics

- **Batching**: flush on count **or** time (so a trickle of samples still persists promptly).
- **Ordering**: records are keyed by `uuid`; DATABASE is a set, not order-dependent — parity is
  set-equality, matching V1.
- **Atomicity**: move-then-record; if the process dies mid-batch, un-acked staging dirs are
  re-processed on restart (idempotent by `uuid`; a present `SAMPLE/<uuid>/` short-circuits).
- **NAS**: `move` within a volume is ~instant; cross-volume falls back to copy+delete with a
  warning. `delete_method: rm` only for mass-delete on Unix.
- **Writer crash** (design §10): fatal — stop-and-checkpoint; no silent data loss.
- **Backpressure**: if the Archiver lags, `hep:archive_queue`/staging grows; the acceptance
  gate (D7.1) asserts a **bounded** backlog (staging must not grow without bound).

---

## 6. Interfaces

- **Worker** → `submit_result` (info + staging path) to `hep:archive_queue`.
- **RedisQueue** → `pull_result`; `_ack` updates persistence markers.
- **observable_io** → schema inference + CSV fieldnames (frozen contract).
- **core** → Archiver lifecycle (spawn at init, drain+finalize at shutdown).
- **Checkpoint (D6.2)** → safe barrier waits for Archiver ack of consumed batches.

---

## 7. Tests

`tests/test_archiver_handoff.py` (D4.1):
1. **Handoff** — a staged Sample lands in `SAMPLE/<uuid>/` identical to the V1 golden; staging
   emptied; clean shutdown drains the queue.
2. **Move-not-copy** — same-volume products are `rename`d (assert inode preserved / no copy);
   cross-volume falls back without error.

`tests/test_archiver_parity.py` (D4.2):
3. **Batch parity** — DATABASE/SAMPLE/CSV **byte-identical** to the captured V1 golden on a
   fixed scan (`--convert` parity); batch boundaries drop/duplicate no Samples.
4. **Idempotent restart** — kill mid-batch → restart re-archives only un-acked staging; final
   set has no missing/duplicate `uuid`.
5. **delete_method** — both `shutil` and `rm` clean staging; bad method raises.
6. **Bounded backlog** — under a Worker output rate, staging size stays bounded (the D7.1
   archive-latency gate in miniature).

Verification logic: test 3 is the frozen-output gate; tests 4/6 prove the async transport is
safe and self-limiting (the whole point of moving persistence off the Worker path).

---

## 8. Open questions

- HDF5 writer as a **separate process** (one control→writer channel) vs. in-Archiver thread —
  default: in-Archiver thread first; promote to a process if it becomes the bottleneck.
- `aiofiles` vs. thread-pool for async moves (default: `shutil.move` in a thread pool; `aiofiles`
  only if measured to help on the target NAS).
- Per-scan single HDF5 file vs. sharded (keep V1's single-file contract).
