# Component — Sample (`jarvishep2/sample.py`)

**Role**: the unit of work. Carries a parameter point from the Sampler, across Redis, into a
Worker, through the workflow, and out to the Archiver.
**Status**: **As-built** @ `jarvis2` `d0de31a`. 643 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §3; discussions
`worker_design.md` §3/§5.
**Reuses V1**: none by import — V2 is a **fresh dataclass reimplementation** (V1 `jarvishep/` is
never imported). It keeps V1-compatible *semantics* for `materialize()`, `create_info()`,
`logger_name` metadata, `@SampleID/@Sdir/@PackID` tokens, and buffered failure-replay.

---

## 1. Module & line count

`jarvishep2/sample.py` (643 lines). Exports (`__all__`): `ExecutionStep`, `Sample`,
`UMapperProtocol`, `VALID_EXECUTION_STEP_TYPES`, `ensure_sample_materialized`,
`materialize_failure_artifacts`. Depends on `runtime_config.should_eager_materialize` /
`should_materialize_on_failure` and `sample_logger.{BufferedSampleLogger,SampleLogger}`.

Module constants: `VALID_EXECUTION_STEP_TYPES = {"calculator","opera","likelihood",
"nuisance_optimize"}`; `_INFO_PROJECTION_KEYS` (save_dir, run_log, logger_name, pack_id, NAttempt,
worker_id, staging_path, product_list); `_TIMING_KEYS` (elapsed_s, started_at, finished_at).

---

## 2. Classes defined

### 2.1 `UMapperProtocol` — `@runtime_checkable Protocol`
The Worker-held `u → x` mapper contract. One method `map(u_coords: np.ndarray) -> Mapping[str,
Any]`. Any object exposing `map()` (see [umapper.md](umapper.md)) satisfies it.

### 2.2 `ExecutionStep` — `@dataclass`
One workflow step in the execution plan (JSON-serializable).

| Attribute | Type | Meaning |
|-----------|------|---------|
| `type` | `str` | one of `VALID_EXECUTION_STEP_TYPES` |
| `name` | `str` | module / calculator / opera name |
| `layer` | `int` | DAG layer index (same layer ⇒ may run concurrently) |
| `params` | `dict` | per-step params (default `{}`) |

| Method | Signature | Behavior |
|--------|-----------|----------|
| `to_dict` | `() -> dict` | JSON-able dict (copies `params`). |
| `from_dict` | `@classmethod (data) -> ExecutionStep` | Validates `type` via `_validate_execution_step_type`, coerces `layer` to int. |

### 2.3 `Sample` — `@dataclass`
The unit of work: identity + `u_coords` cross Redis; everything else is Worker-local.

**Attributes** (dataclass fields):

| Field | Type / default | Crosses Redis? | Meaning |
|-------|----------------|----------------|---------|
| `uuid` | `str` | ✅ | UUID4 primary key |
| `u_coords` | `np.ndarray` (empty float64) | ✅ | normalized draw — the only heavy wire field |
| `execution_plan` | `list[ExecutionStep]` | ✅ | which steps in which layer |
| `opera_params` | `dict` | ✅ | extra params for opera steps |
| `sample_artifacts` | `str = "auto"` | ✅ | `auto` / `always` / `never` |
| `priority` | `int = 0` | ✅ | queue priority hint |
| `created_at` | `str \| None` | ✅ | ISO timestamp |
| `params` | `dict` (`repr=False`) | ❌ | x-space params (Worker-local) |
| `info` | `dict` (`repr=False`) | ❌ | V1-compat bag (save_dir, logger, handlers…) |
| `observables` | `dict` (`repr=False`) | ❌ | computed observables |
| `status` | `str = "Created"` | ❌ | Created/Init/Running/Completed/Failed |
| `_materialized` | `bool` | ❌ | dirs/logger created on disk |
| `_logger` | `SampleLogger \| BufferedSampleLogger \| None` | ❌ | active per-Sample logger |
| `_with_nuisance` | `bool` | ❌ | nuisance card combined |
| `_likelihood` | `Any` | ❌ | backing field for the `likelihood` property |
| `_params_bound` | `bool` | ❌ | `bind_params` has run |

**Member functions:**

| Method | Signature | Behavior |
|--------|-----------|----------|
| `from_params` | `@classmethod (params) -> Sample` | Control-side seed from a param dict (V1-compatible); mints uuid, copies into `observables`. |
| `u` | `@property -> np.ndarray` | alias of `u_coords`. |
| `likelihood` | `@property` getter/setter | backed by `_likelihood`. |
| `save_dir` | `@property -> str\|None` | from `info["save_dir"]`. |
| `update_uuid` | `(new_uuid) -> None` | rewrite uuid in `observables` + `info`. |
| `to_task_dict` | `() -> dict` | **Wire format** for `hep:task_queue`: uuid, u_coords→list, execution_plan, opera_params, sample_artifacts, priority, created_at. Raises if a forbidden key (`logger/handlers/params/info/observables`) is present (invariant #7/#8). |
| `from_task_dict` | `@classmethod (data) -> Sample` | Reconstruct inside Worker; `u_coords`→ndarray, plan rebuilt; logger/paths **not** restored. |
| `to_info_dict` | `() -> dict` | Result/monitor projection: status, params, observables, likelihood, plan, plus `timings` + selected `_INFO_PROJECTION_KEYS`; strips `logger`. |
| `bind_params` | `(mapper: UMapperProtocol\|None) -> None` | Idempotent `u → x`: `mapper.map(u_coords)` → `params`/`observables`/`info`; sets `_params_bound`. No-op if mapper is None. |
| `set_config` | `(config) -> None` | Adopt `info`, normalize `sample_artifacts`, `create_info()`, combine nuisance card if present, eager-materialize when `should_eager_materialize`. |
| `create_info` | `() -> None` | Build the V1-compat `info` bag: sample_dirs root, logger_name `Sample@<uuid>`, lazy `BufferedSampleLogger`, handlers `{}`, status `Init`. **Target:** pass `console=bool(info.get("sample_console"))` into the lazy logger. |
| `combine_nuisance_card` | `() -> None` | Merge `info["nuisance"]["active"]["param"]` into params; set `NAttempt`. |
| `gather_nuisance` | `() -> None` | Re-stamp uuid into observables (nuisance loop). |
| `_resolve_sample_root` | `@staticmethod (info) -> str` | Pick SAMPLE root: parent of save_dir, else sample_dirs, else `<task_result_dir>/SAMPLE`. |
| `_buffered_logger` | `@staticmethod (info) -> BufferedSampleLogger\|None` | extract a buffered logger from info. |
| `_active_logger` | `() -> logger\|None` | `info["logger"]` or `_logger`. |
| `_sync_logger_handles` | `(logger) -> None` | keep `_logger` and `info["logger"]` aligned. |
| `child_logger` | `(*, module=None) -> logger\|None` | bind a child logger without materializing. |
| `materialize` | `(worker_id=None, *, bucket_parent=None, failure_message=None) -> str` | Worker-side: create `SAMPLE/<bucket>/<uuid>/`, set save_dir/run_log; replay buffered events into `Sample_running.log` then open a file `SampleLogger`. No-op if already materialized. |
| `_open_sample_logger` | `(*, announce_creation=True) -> None` | open a file-backed `SampleLogger`; **target:** pass `console=bool(info.get("sample_console"))` for check-modules mirroring; optionally log "Sample created into the Disk". |
| `materialize_failure_artifacts` | `(error=None) -> str\|None` | gated by `should_materialize_on_failure`; materialize + log failure message. |
| `resolve_token` | `(text, *, stage="runtime", field="") -> str` | Replace `@PackID/@SampleID/@Sdir`; `@Sdir` triggers `materialize()`; raises if pack_id/uuid missing. |
| `start` | `() -> None` | status → Running; log readiness + nuisance attempt banner. |
| `close` | `() -> None` | log close summary (if materialized) then `close_logger`. |
| `_build_close_message` | `() -> str` | format the "Sample SUMMARY" block from observables. |
| `close_logger` | `() -> None` | discard buffered / close file logger; clear handles. |
| `record` | `() -> dict` | Final Archiver record: `{uuid, observables, metadata}` (metadata = selected info keys). |
| `format_summary` | `@staticmethod (values, *, key_width=None, value_width=60, float_precision=6) -> str` | two-column aligned key/value rendering with float/nan/inf formatting. |

---

## 3. Module-level functions

| Function | Behavior |
|----------|----------|
| `_make_lazy_sample_logger(*, module, console=False)` | build a `BufferedSampleLogger` with default extra. **As-built** ignores `console`; **target** forwards `console=` for check-modules (see [logger.md](logger.md) §3). |
| `_utc_now_iso()` | UTC ISO timestamp. |
| `_validate_execution_step_type(step_type)` | guard against unknown step types. |
| `_as_float64_array(value)` | coerce scalar/list/ndarray/None → float64 ndarray. |
| `_attach_sample_for_materialize(sample_info)` | build a throwaway `Sample` bound to an existing `info` bag. |
| `ensure_sample_materialized(sample_info, *, failure_message=None) -> str\|None` | **public**: materialize artifacts on demand (e.g. `@Sdir`); respects `sample_artifacts == "never"`. |
| `materialize_failure_artifacts(sample_info, *, error=None) -> str\|None` | **public**: materialize a failed sample + replay buffered logs. |

---

## 4. Serialization contract (the critical surface)

`to_task_dict()` is the **only** thing crossing into Redis. Round-trip law:
`Sample.from_task_dict(s.to_task_dict())` reproduces `uuid`, `u_coords` (allclose),
`execution_plan`, `opera_params`, `priority`, `sample_artifacts`. `params`/`info`/`observables`/
`_logger` are Worker-local and **not** preserved.

### 4.1 Check-modules console flag (target)

V1 stores `info["sample_console"]` (bool, default `False`). V2 may keep it on `info` and/or as a
dedicated field — either way it is **Worker/control-local only** and must not appear on the wire.

| Policy | Behavior |
|--------|----------|
| Production scan | `sample_console=False` — sample command logs only to `Sample_running.log` |
| `--check-modules` | set `sample_console=True` on test samples; `SampleLogger` / `BufferedSampleLogger` mirror the same rendered text to the terminal |

Ownership of the mirror is entirely inside the sample logger (`console=` constructor flag). The
top-level process logger must not be used as the sample command console sink. Full contract:
[logger.md](logger.md) §3–§4.

---

## 5. Interfaces / collaborators

- **Sampler** ([sampler.md](sampler.md)) builds it (`uuid`+`u_coords`) and attaches the
  `execution_plan`.
- **Worker** ([worker.md](worker.md)) owns the Worker-side lifecycle (`from_task_dict` →
  `bind_params` → `materialize` → run → `record`/`close`).
- **UMapper** ([umapper.md](umapper.md)) supplies `map()` for `bind_params`.
- **Archiver** ([datarecorder.md](datarecorder.md)) consumes `record()` / `to_info_dict()`.
- **runtime_config** ([config_schema.md](config_schema.md)) gates eager/failure materialization.
- **sample_logger** ([logger.md](logger.md)) provides the buffered + file loggers (and optional
  console mirror).
- **check-modules** ([core.md](core.md)) may set `sample_console=True` on test samples. Normal
  sampler-produced tasks must not enable it.

---

## 6. Drift from design

The design doc framed `Sample` as *extending V1's lazy `Sample(Base)`*. **As-built it is a fresh
dataclass** with no V1 import. New vs. the old doc: `opera_params`, `priority`, `created_at`,
`_with_nuisance`, `_params_bound`, `update_uuid`, `set_config`, `create_info`,
`combine_nuisance_card`/`gather_nuisance` (nuisance is **wired but only lightly used** — see
[nuisance.md](nuisance.md)), `child_logger`, `format_summary`, and the module-level
`ensure_sample_materialized` / `materialize_failure_artifacts` helpers.

**Open gap (2026-07-09 V1 sample-command logging):** V2 as-built does not yet plumb
`sample_console` into `_make_lazy_sample_logger` / `_open_sample_logger`, and
`jarvishep2/sample_logger.py` lacks the `console=` path. Target behavior is documented in
[logger.md](logger.md) §3.6 and §4.

---

## 7. Tests

`tests/test_sample_taskdict.py` (16 tests): round-trip, no-logger-on-wire, lazy materialization,
failure replay, token resolution, status transitions. Also exercised end-to-end in
`tests/test_worker_*`, `tests/test_d0_integration.py`, and `tests/test_distributed_acceptance.py`.
