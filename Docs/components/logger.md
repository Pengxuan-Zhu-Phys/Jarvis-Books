# Component — Logger (two-layer) (`jarvishep2/logging/`)

**Role**: lightweight, multiprocess-safe logging in two layers — top-level process logs and
per-Sample detailed logs — over the standard library (no loguru).
**Status**: design — plan WP-D0.3.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §9; discussion
`jarvis_hep_logging_design.md`; contract `docs/specs/LOGGING_CONTRACT.md`.
**Reuses V1**: wraps `jarvishep/sample_logger.py` (`SampleLogger`, `BufferedSampleLogger`,
`replay_sample_log_events`) — the per-Sample sink and failure-replay are kept verbatim.

---

## 1. Responsibilities

1. **Top layer (`JarvisLogger`)**: process-level summaries for Sampler / Factory / Worker /
   core → console (color) + `logs/jarvis_<role>_<pid>.log` (rotation). High-level only.
2. **Sample layer (`SampleLogger`)**: one file per Sample, full trace; created and closed
   **inside the Worker**; never crosses a process boundary.
3. **Context binding**: `.bind(worker_id=…, sample_uuid=…, phase=…, calculator=…)`.
4. **Low overhead** under full-CPU load: `QueueHandler` non-blocking files, `isEnabledFor`
   guards on hot paths, no per-call config lookups.
5. **Drop loguru**: remove the global-sink model that fights multiprocess + per-Sample files.

---

## 2. Module structure

```
jarvishep2/logging/
├── __init__.py          # re-exports: setup_jarvis_logging, get_jarvis_logger
├── toplevel.py          # JarvisLogger + adapter + setup
└── sample.py            # thin re-export of V1 SampleLogger / BufferedSampleLogger
```

```python
def setup_jarvis_logging(level="INFO", console=True, role="jarvis",
                         log_dir="logs", max_bytes=100*2**20, backup_count=7,
                         use_queue=True) -> None: ...

class JarvisLoggerAdapter(logging.LoggerAdapter):
    def process(self, msg, kwargs): ...          # merge bound ctx into the record
    def bind(self, **ctx) -> "JarvisLoggerAdapter": ...

def get_jarvis_logger(name="jarvis") -> JarvisLoggerAdapter: ...
```

---

## 3. Member functions

| Function / method | Signature | Behavior |
|-------------------|-----------|----------|
| `setup_jarvis_logging` | see above | Configure the `jarvis` root logger once per process: color console (`colorlog` if present, plain otherwise), rotating file handler wrapped in `QueueHandler` + a `QueueListener`. Idempotent (clears handlers first). |
| `get_jarvis_logger` | `(name) -> adapter` | Return an adapter over `logging.getLogger(name)` with empty ctx. |
| `JarvisLoggerAdapter.bind` | `(**ctx) -> adapter` | New adapter with merged context (worker_id, sample_uuid, phase…). Rendered as `key=value` suffix per `LOGGING_CONTRACT.md`. |
| `SampleLogger.open` | `(path, …)` | (V1) open the per-Sample file backend. |
| `SampleLogger.bind` | `(**extra)` | (V1) child logger with extra fields. |
| `SampleLogger.{info,warning,error,…}` | `(msg, *a, **kw)` | (V1) write one contract-formatted line. |
| `SampleLogger.close` | `()` | (V1) flush + close the file handle. |
| `BufferedSampleLogger.replay_to` | `(path)` | (V1) replay buffered events to a file (failure path). |
| `BufferedSampleLogger.discard` | `()` | (V1) drop the buffer (lazy success path). |

---

## 4. Worker usage pattern

```python
top = get_jarvis_logger("worker").bind(worker_id=self.worker_id)
top.info("start sample", extra={"sample_uuid": uuid})          # summary only

slog = sample.materialize_logger()        # SampleLogger (file) or BufferedSampleLogger (lazy)
slog.info("materialize done", extra={"phase": "materialize"})  # full detail
...
top.info("sample done", extra={"sample_uuid": uuid, "elapsed_s": dt})
```

Rule: **summaries up top, full trace in the Sample log.** A failed sample logs
"failed; see Sample log" at the top layer and the complete trace in its file.

---

## 5. Concurrency / lifecycle / failure semantics

- Each **process** calls `setup_jarvis_logging(role=…)` once; file names carry `pid` so
  multiple Workers never share a file handle.
- Per-Sample loggers are **created/closed within one Worker**; `sample.info["logger"]` is set
  to `None` before any serialization (invariant #8).
- Failure-replay (invariant #9) is unchanged from V1: buffer → replay on failure, discard on
  lazy success.
- `QueueListener` runs on a dedicated thread per process; on shutdown it is stopped and
  drained (no lost tail lines).

---

## 6. Interfaces

- **Worker / Factory / Sampler / core** → `get_jarvis_logger(role).bind(...)`.
- **Sample / Calculator / Likelihood** → per-Sample `SampleLogger` (bound child loggers), as
  in V1; the `logger_name` metadata lets children bind without materialization.

---

## 7. Tests (`tests/test_logging_layers.py`)

Unit:
1. **No loguru** — `grep` guard in the test asserts no `import loguru` on the hot path.
2. **Two layers** — a simulated sample produces (a) a top-level summary line and (b) a
   per-Sample file with the full trace; the summary file does **not** contain the detail.
3. **Contract format** — emitted lines match `LOGGING_CONTRACT.md` fields (timestamp, level,
   name, message, `key=value` context).
4. **Failure replay** — buffered → forced failure → file content equals buffered events
   (reuse the V1 `test_failure_replay_log` assertions).
5. **No cross-process logger** — assert `sample.to_task_dict()` carries no logger and the
   reconstructed sample builds its own.
6. **Non-blocking** — with `QueueHandler`, logging 10⁵ lines does not block the producer
   beyond a bound (timing sanity, not a hard perf gate).

Verification logic: layer separation (test 2) and contract format (test 3) are the gates;
everything an operator needs to debug a Sample is in its own file, and monitoring stays light.

---

## 8. Open questions

- JSON-lines mode for Sample logs (future; default text per contract).
- Optional Redis Stream central sink for multi-host runs (design §9 future).
