# Component — DataRecorder / Archiver (`jarvishep2/archiver.py`, `jarvishep2/database.py`, `jarvishep2/archive_handoff.py`, `jarvishep2/file_ops.py`, `jarvishep2/sample_bucket.py`)

**Role**: the final persistence layer (Layer 2). Consumes finished Samples from
`hep:archive_queue`, writes DATABASE rows, and (default) packs sealed SAMPLE buckets into
`*.tar.gz` only after every sample in the bucket is archived.
**Status**: **As-built** @ `jarvis2` **`67d760d`** (own Archiver logger) + pack-after-archive.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §5.
**Reuses V1**: none by import (V1 `BucketAllocator` / `SampleArchiveManager` parity reimplemented).

> **As-built drift:** `hdf5writer.py` → `database.py` (`SimpleHDF5Writer`, JSON-rows-in-HDF5). No
> `GlobalHDF5Writer`/`observable_io`, and no `--convert`. Calculator **format** I/O is Portal;
> **SAMPLE save/copy** is HEP `FileOperationService` (see [io_system.md](io_system.md)).
>
> **Defaults (2026-07-14+):** `Archiver.mode=process`, `handoff=direct` / `Cleanup.strategy=direct`
> (staging is **optional**), `pack_buckets=true` with Redis bucket lifecycle.
>
> **Logger (2026-07-16):** process mode calls `setup_jarvis_logging(role="archiver")` and binds
> `module=Archiver` → console + `logs/<scan>/jarvis_archiver.log`. Pack/flush/drain lines use
> the **`·•· Archiver`** prefix (not the control `Jarvis-HEP` ‰ drain heartbeats).

---

## 1. `archiver.py`

### 1.1 `ArchiveProcessor`
Consume archive payloads: prefer existing `save_dir` / `SAMPLE/<bucket>/<uuid>/`, write DATABASE.
- **Attributes:** `writer` (`SimpleHDF5Writer`), `sample_root`, `delete_method`, `strategy`
  (`move`/`copy`), `delete_after_archive`, `batch_size`, `flush_interval_sec`, `records_written`,
  `acked_uuids`, `_batch`, `_last_flushed`, `_lock`.
- **Methods:** `ingest(result)`, `flush_batch()`, `take_last_flushed()`, `_archive_one(result)`
  (idempotent by uuid; writes observables even if the on-disk tree was already packed),
  `persistence_state()`.

### 1.2 `SimpleArchiver`
Background **thread** draining `hep:archive_queue` + packing ready buckets.
- After each flush: `redis.note_sample_archived(bucket_id)` for each written row.
- `_pack_ready_buckets()` tars only buckets Redis marked ready (`archived >= assigned`).
- Default `pack_buckets=true`.
- Optional `logger=` (module=`Archiver`); logs DATABASE row progress (~every 50), pack success/fail, drain/stop.

### 1.3 `ArchiverProcess(Process)`
Default Archiver mode. Own OS process (`Jarvis2-Archiver:<scan>` via `setproctitle`).
`run()` configures Archiver logging (`log_path=logs/<scan>/jarvis_archiver.log` when provided),
hosts a `SimpleArchiver` loop, and mirrors `records_written` into a shared `Value`.

## 2. `database.py` — `SimpleHDF5Writer`
Append **full** JSON-encoded `observables` dicts (not params-only) to `samples.hdf5` `records[]`.
Archiver may add `product_list` / `bucket_dir` / `bucket_id` onto the same row.

Post-run plot emit also writes `DATABASE/samples.csv` as a **full-column** export of those
records (nested values JSON-encoded); see [monitor](monitor.md) / `plot_scene.export_samples_csv_from_hdf5`.

### 2.1 Sampler diagnostic files (D13.6, additive)

Under the same `DATABASE/` directory (never changes the frozen `run_summary.*` schema):

| File | Producer | Contents |
|---|---|---|
| `samples.hdf5` / `samples.csv` | Archiver / plot_scene | Per-Sample observables. MCMC proposals stamp `chain_id`, `step`, `stage` on the proposal row **before** Metropolis absorb. |
| `sampler_summary.json` | MCMC / nested finish + Core `write_run_summary` mirror | Method summary: accept rates, R-hat (`rhat_logl`), per-chain stats, nested `logz`/`logzerr`/`ncall`. |
| `chain_history.csv` | MCMC/ensemble finish (`diagnostics_export`) | One row per completed Metropolis iteration: `chain_id`, `step`, `accepted` (0/1), `weight` (1 if accepted else 0), `logl`, `temperature`, `state`. |
| `dynesty_result.csv` / `multinest_result.csv` | Dynesty/MultiNest finalize | Nested dead-point table for Jarvis-PLOT `dynesty_runplot` (`log_weight`, `log_Like`, `log_Evidence`, …). |

**Contract notes**

* Accept/reject is **not** rewritten onto the archived HDF5 proposal row (archive is Worker-side before control-side absorb). Use `chain_history.csv` for accepted/weight.
* Nested importance weights are `log_weight` in `*_result.csv`, not `chain_history.csv`.
* Columns are **additive** (hard invariant 3): missing files mean the method did not run or export failed with a warning.

## 3. `archive_handoff.py` — optional staging helpers
`normalize_move_strategy`, `resolve_staging_dir`, `move_tree`, `stage_sample_dir`,
`archive_staging_to_sample`, `list_product_names`. Used when `Cleanup.strategy=mv_to_staging`;
**not** on the default direct path.

## 4. `sample_bucket.py` — SAMPLE layout + tar
`normalize_sample_directory`, `bucket_dir_path`, `pack_bucket_dir` (tar.gz + prune),
`format_bucket_name`. Layout: `SAMPLE/000001/<uuid>/…` → `SAMPLE/000001.tar.gz`.

## 5. `file_ops.py` — deletion + SAMPLE save helpers
`DEFAULT_DELETE_METHOD="shutil"`; `normalize_delete_method`, `delete_path` / `delete_paths`.
Also: `save_io_copy` / `apply_io_save_policy` (SAMPLE landing for YAML `save:`) — executed by
Worker-side `FileOperationService`, not by the Archiver process. See [io_system.md](io_system.md).

## 6. Calculator Layer-1 format I/O + SAMPLE save

Format R/W: Portal. SAMPLE copy/delete policy: HEP FileOperation. See [io_system.md](io_system.md).

---

## 7. Two-layer I/O flow (default = direct)

```
Worker pull → Redis.allocate_sample_bucket → SAMPLE/<bucket>/<uuid>/
  → calculator on @PackID (or @Sdir) → save:true products into sample dir
  → submit_result → finish_sample_bucket (active--)
Archiver:
  pull_result → ingest/flush → SimpleHDF5Writer (DATABASE)
  note_sample_archived(bucket_id)
  when sealed && active==0 && archived>=assigned → pack_bucket_dir → <bucket>.tar.gz
```

Optional staging path (`Cleanup.strategy: mv_to_staging`):

```
Worker: save_dir → stage_sample_dir → staging/<uuid> ; submit_result(staging_path)
Archiver: staging → SAMPLE/... then DATABASE (legacy hop)
```

---

## 8. Concurrency / failure semantics

- Batch flush on count **or** time.
- **Pack-after-archive invariant:** never tar/prune a bucket until every assigned sample has a
  DATABASE row. Early pack was a production hang (`archived` stuck below `ok`).
- Idempotent uuid acks; observables written even if the sample tree is already gone.
- Stall detection in `wait_for_results`: workers idle + `archive_q=0` + frozen row count → clear error.

---

## 9. Interfaces / collaborators

- **Worker** ([worker.md](worker.md)) materialize + `submit_result` + `finish_sample_bucket`.
- **RedisQueue** ([redis_queue.md](redis_queue.md)) archive queue + bucket meta/state/ready.
- **runtime_config** `ARCHIVER_DEFAULTS` / `CLEANUP_DEFAULTS` / `get_sample_directory_config`.
- **core** Archiver lifecycle + managed Redis + signal shutdown.

---

## 10. Tests

`tests/test_archiver_handoff.py`, `test_archiver_parity.py`, `test_file_ops.py`,
`test_sample_bucket.py` (allocate / finish / note_archived / pack order).
