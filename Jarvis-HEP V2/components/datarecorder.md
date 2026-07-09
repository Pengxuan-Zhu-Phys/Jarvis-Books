# Component — DataRecorder / Archiver (`jarvishep2/archiver.py`, `jarvishep2/database.py`, `jarvishep2/archive_handoff.py`, `jarvishep2/file_ops.py`)

**Role**: the final persistence layer (Layer 2). Consumes finished Samples from
`hep:archive_queue`, moves staged products into `SAMPLE/<uuid>/`, and writes DATABASE rows.
Decoupled from calculator execution.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `archiver.py` 286 + `database.py` 71 +
`archive_handoff.py` 122 + `file_ops.py` 81 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §5.
**Reuses V1**: none by import.

> **As-built drift:** `hdf5writer.py` → `database.py` (`SimpleHDF5Writer`, JSON-rows-in-HDF5). No
> `GlobalHDF5Writer`/`observable_io`, and no `--convert`. Calculator format I/O is owned by
> Jarvis-Portal (see [io_system.md](io_system.md)), not the Archiver. The Archiver writes one
> JSON-encoded observables row per Sample.

---

## 1. `archiver.py`

### 1.1 `ArchiveProcessor`
Consume archive payloads: move staging → `SAMPLE/` and write DATABASE rows.
- **Attributes:** `writer` (`SimpleHDF5Writer`), `sample_root`, `delete_method`, `strategy`
  (`move`/`copy`), `delete_after_archive`, `batch_size`, `flush_interval_sec`, `records_written`,
  `acked_uuids: set`, `_batch`, `_last_flush`, `_lock`.
- **Methods:** `ingest(result)` (queue + threshold flush), `flush_batch()`, `_flush_batch_locked()`,
  `_archive_one(result)` (idempotent by uuid: present `SAMPLE/<uuid>/` short-circuits; else move
  staging/save_dir in, write the observables row, ack), `persistence_state()` (acked_uuids +
  highwater for the checkpoint safe barrier).

### 1.2 `SimpleArchiver`
Background **thread** draining `hep:archive_queue`.
- **Attributes:** `redis`, `poll_timeout`, `processor` (an `ArchiveProcessor`), `_stop_event`,
  `_thread`; property `records_written`.
- **Methods:** `start()`, `_run_loop()`, `drain(*, idle_timeout=2.0)`, `cleanup_staging(paths)`,
  `stop(*, wait=True, drain=True)`, `persistence_state()`.

### 1.3 `ArchiverProcess(Process)`
Spawned Archiver process (same processing, own Redis client + `Value("i")` record counter).
`run()` loops `pull_result` → `ingest`/`flush_batch`; `stop(*, wait, timeout)`.

## 2. `database.py` — `SimpleHDF5Writer`
Append JSON-encoded observables rows to `samples.hdf5`. **Attributes:** `db_path`, `_lock`.
**Methods:** `add_record(observables)` (resize + append to the `records` string dataset),
`read_records()` (decode all rows). Module function `make_json_compatible(value)`.

## 3. `archive_handoff.py` — staging helpers
`normalize_move_strategy`, `resolve_staging_dir`, `move_tree` (rename-preferred move/copy),
`stage_sample_dir` (Worker: work_dir → `staging/<uuid>`), `archive_staging_to_sample` (idempotent
`staging → SAMPLE/<uuid>/`), `list_product_names`, `same_volume_move_preserves_inode`.

## 4. `file_ops.py` — deletion backends
`DEFAULT_DELETE_METHOD="shutil"`; `normalize_delete_method`, `delete_path(*, method, missing_ok)`
(refuses unsafe roots; `shutil` or `rm -rf`), `delete_paths`.

## 5. Calculator Layer-1 format I/O

Not part of the Archiver. See [io_system.md](io_system.md) (`jarvishep2/io_portal.py` → Portal).

---

## 6. Two-layer I/O flow

```
Worker:   work_dir → stage_sample_dir → staging/<uuid> ; submit_result(info_dict)
Archiver: pull_result → ArchiveProcessor.ingest → archive_staging_to_sample → SAMPLE/<uuid>/
                                                 → SimpleHDF5Writer.add_record → DATABASE
```

The Worker stages instantly and moves on; the Archiver absorbs the move + write cost in batches off
the critical path.

---

## 7. Concurrency / failure semantics

- Batch flush on count **or** time. Records keyed by uuid (DATABASE is a set; parity = set-equal).
- **Idempotent restart:** a present `SAMPLE/<uuid>/` or an already-acked uuid short-circuits → no
  dup/missing on resume.
- `delete_method` (`shutil`/`rm`) cleans staging; unsafe paths refused.

---

## 8. Interfaces / collaborators

- **Worker** ([worker.md](worker.md)) stages + `submit_result`.
- **RedisQueue** ([redis_queue.md](redis_queue.md)) `pull_result`.
- **runtime_config** ([config_schema.md](config_schema.md)) `ARCHIVER_DEFAULTS` / archiver config.
- **core** ([core.md](core.md)) Archiver lifecycle; **checkpoint** ([checkpoint.md](checkpoint.md))
  consumes `persistence_state`.

---

## 9. Tests

`tests/test_archiver_handoff.py` (5), `test_archiver_parity.py` (2), `test_file_ops.py` (16):
staging handoff, move-not-copy (inode), batch DATABASE parity, idempotent restart, delete methods.
