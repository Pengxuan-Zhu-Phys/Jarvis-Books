# Component — Logger (two-layer) (`jarvishep2/logging/`, `jarvishep2/sample_logger.py`, `jarvishep2/log_kv.py`)

**Role**: lightweight, multiprocess-safe logging in two layers — top-level process logs and
per-Sample detailed logs — over the standard library (no loguru).
**Status**: **As-built** @ `jarvis2` **D12.2** — V1 visual formatter (`·•·` / `Ϡ`), scan-scoped
main log path, logo banner; QueueListener architecture unchanged (D0.3). Remaining gaps: V1
check-modules console mirroring and calculator command-block ownership (this doc §3–§4) are not
yet fully landed in V2 `sample_logger.py` / `Sample.sample_console`.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §9;
[`../PROTOTYPE_CLOSEOUT_REVIEW_2026-07-14.md`](../PROTOTYPE_CLOSEOUT_REVIEW_2026-07-14.md) §5.1.
**Reuses V1**: none by import — top-level rendering and per-Sample sink/replay are **fresh
reimplementations** of the V1 visual contract. Logo template lives at `jarvishep2/card/logo`
with `jarvishep2/versioning.py`.

---

## 1. Modules

```
jarvishep2/logging/
├── __init__.py     # re-exports both layers
├── toplevel.py     # process-level logging (adapter + setup over stdlib)
└── sample.py       # thin re-export of jarvishep2.sample_logger
jarvishep2/sample_logger.py   # per-Sample file + buffered loggers (the real implementation)
jarvishep2/log_kv.py          # two-column key/value formatting helper
```

> `logging/sample.py` simply re-exports from `jarvishep2/sample_logger.py`; the package
> `__init__.py` exports `BufferedSampleLogger`, `SampleLogEvent`, `SampleLogger`,
> `replay_sample_log_events`, `get_jarvis_logger`, `setup_jarvis_logging`,
> `shutdown_jarvis_logging`.

---

## 2. Top layer — `logging/toplevel.py`

Module state: `_state` dict (configured/listener/log_queue/log_path), `JARVIS_HEP_LOG_DOMAIN =
"jarvis_hep"`, `_RECORD_SKIP_KEYS` (stdlib record fields to skip when rendering context).

### Classes
| Class | Base | Purpose / key methods |
|-------|------|-----------------------|
| `JarvisContextFormatter` | `logging.Formatter` | V1 visual layout: `\n·•· {module}\n\t-> {MM-DD HH:mm:ss.SSS} - [LEVEL] >>>\n{message}` (+ trailing `key=value` context). `raw=True` → message passthrough. `Ϡ` bullet when module is `Jarvis-HEP.hdf5-Writter`. Optional ANSI colorize on TTY. |
| `JarvisLoggerAdapter` | `logging.LoggerAdapter` | `process()` merges bound `extra`; remaps V1 `module=` onto non-reserved `jarvis_module` (stdlib forbids overwriting `LogRecord.module`). `bind(**ctx)` returns a new adapter. |

### Functions
| Function | Behavior |
|----------|----------|
| `format_record_context(record) -> str` | render non-standard record fields as sorted `key=value` (skips reserved / presentation keys). |
| `_resolve_level(level) -> int` | name/int → logging level (default INFO). |
| `_make_console_handler(*, level) -> Handler` | `StreamHandler(stderr)` + V1 formatter with colorize. |
| `_make_file_handler(*, log_path, level, max_bytes, backup_count) -> RotatingFileHandler` | rotating file sink; default max **5 MiB** (V1 rotation). |
| `setup_jarvis_logging(*, level="INFO", console=True, role="jarvis", log_dir="logs", log_path=None, max_bytes=5MiB, backup_count=7, use_queue=True) -> str` | configure the `jarvis_hep` domain **once per process**: console + rotating file; `log_path` if set else `logs/jarvis_<role>_<pid>.log`; QueueHandler+QueueListener when `use_queue`; returns the log path. |
| `shutdown_jarvis_logging() -> None` | stop the queue listener, drain, reset `_state`. |
| `get_jarvis_logger(name="jarvis") -> JarvisLoggerAdapter` | qualified adapter under the `jarvis_hep` domain, bound with `jarvis_module=name`. |

**Control-process layout (D12.2 + component files 2026-07-16):** `Jarvis2Core.init_logger`
writes under ``<task_root>/logs/<scan>/``, prints the logo banner via `render_logo_with_version()`,
and binds `module=Jarvis-HEP`.

### Scan-scoped component files (as-built)

All process logs for a scan live under **`logs/<scan>/`**, one primary file per process
component (Jarvis V1 visual format unchanged: ``·•· <module>`` / timestamp / level):

```text
logs/<scan>/
  core.log           # control process (Jarvis-HEP, Factory, Sampler.* via module bind)
  worker-00.log      # Worker 0
  worker-01.log      # Worker 1
  ...
  archiver.log       # Archiver process
```

| Process | `setup_jarvis_logging` | File | `·•·` module label |
|---------|------------------------|------|---------------------|
| Control | `component=core`, `scan_logs_dir=logs/<scan>` | `core.log` | `Jarvis-HEP` |
| Factory / Sampler | same process as control (no extra file) | lines in `core.log` | `Factory`, `Sampler:<name>` |
| Worker *N* | `component=worker`, `worker_id=N` | `worker-NN.log` | `Worker-N` |
| Archiver | `component=archiver` | `archiver.log` | `Archiver` |

Helpers: `scan_logs_dir(task_root, scan)`, `component_log_path(scan_logs, component, worker_id=…)`.

**Sample layer** remains separate: per-sample detail still goes to
`SAMPLE/.../Sample_running.log` (not mixed into component process logs).

End-of-scan **[Scan Performance]** is logged on the **control** logger (`Jarvis-HEP` →
`core.log`). Archiver pack/flush lines go to `archiver.log`.

---

## 3. Sample layer — `sample_logger.py`

This is the per-Sample detailed sink. It is **not** `loguru` and **not** the top-level stdlib
logger. It preserves the frozen sample-log file format with a minimal logger-shaped API
(`debug/info/warning/error/critical`, `bind`, `close`).

Module constants / helpers:

| Symbol | Role |
|--------|------|
| `DEFAULT_BUFFER_MAX_EVENTS = 2048` | bound for `BufferedSampleLogger` deque. |
| `_CONSOLE_LOCK` | process-wide `threading.Lock` for console mirrors (shared by all sample loggers). |
| `_format_sample_log_text(*, module, level, message, raw, timestamp, has_written, last_ended_newline) -> str` | single formatter for file and console so both sinks render identical text. |

### 3.1 `SampleLogEvent` — `@dataclass(frozen=True)`

One buffered log line: `timestamp`, `module`, `level`, `message`, `raw`.

### 3.2 `_SampleLogBackend`

Sample-local file sink with exact legacy formatting; optional console mirror of the **same**
rendered text.

- **Attributes:** `path`, `_time_provider`, `_lock` (per-backend file lock), `_console`,
  `_console_stream`, `_closed`, `_has_written`, `_last_ended_newline`, `_handle` (append-mode).
- **Constructor:** `__init__(path, *, time_provider=None, console=False, console_stream=None)`.
- **Methods:**
  - `_timestamp()` → `%m-%d %H:%M:%S.mmm`
  - `write(*, module, level, message, raw, timestamp=None)` — format via
    `_format_sample_log_text`, write+flush under `_lock`, then optionally
    `_write_console(text)` under `_CONSOLE_LOCK`
  - `_write_console(text)` — `stream.write` + `flush` (default stream: `sys.stderr`)
  - `close()`

Formatting rules (identical for file and console):

- **Structured** (`raw=False`):
  ```text
  ·•· <module>
  	-> MM-DD HH:mm:ss.mmm - [LEVEL] >>>
  <message>
  ```
  Leading blank-line separation when the previous write did not end in newline.
- **Raw** (`raw=True`): message text as-is (ensure trailing newline; insert a leading newline
  only when needed to avoid gluing onto the previous write).

### 3.3 `SampleLogger`

Per-Sample file logging adapter (and optional console mirror). Used after materialization when
`Sample_running.log` exists.

- **Attributes:** `_backend`, `_extra`, `_options` (V1-compat tuple shape for bound-extra
  introspection).
- **Class method** `open(path, *, module, time_provider=None, extra=None, console=False,
  console_stream=None) -> SampleLogger` — merge `module` into extra, open backend, return
  adapter.
- **Methods:** `bind(**extra)` (shares the backend; new adapter), `close()`,
  `log(level, message, *a, **kw)`, level helpers `debug/info/warning/error/critical`,
  `_normalize_level` / `_render_message` (`@staticmethod`).
- **Extras contract:**
  - `module` — head label (`Sample@<uuid>`, or `Sample@<uuid> (Likelihood)`, or
    `Sample@<uuid> (<Calc>-No.<PackID>)`, …)
  - presence of `raw` in bound extra → raw passthrough write (no Sample head)
  - legacy keys may still appear in `extra` (`to_console`, `Jarvis`, `_log_domain`) for
    V1-compat metadata; **console mirroring is controlled by the `console=` constructor flag**,
    not by `to_console` in extra

### 3.4 `BufferedSampleLogger`

In-memory logger used **before** sample artifacts are materialized (lazy path).

- **Attributes:** `_extra`, `_max_events`, `_events` (bounded `deque`), `_time_provider`,
  `_console`, `_console_stream`, `_console_has_written`, `_console_last_ended_newline`,
  `_closed`, `_options`.
- **Constructor:** `__init__(*, extra=None, events=None, max_events=DEFAULT_BUFFER_MAX_EVENTS,
  time_provider=None, console=False, console_stream=None)`.
- **Methods:**
  - `events` / `event_count` (`@property`)
  - `bind(**extra)` — shares the deque (and console state params); child loggers append to the
    same buffer
  - `close()` / `discard()` (clear + close)
  - `replay_to(path)` — flush buffer to a file via `replay_sample_log_events` (**file only**;
    console is not re-emitted on replay — events already mirrored live if `console=True`)
  - `log(...)` + the five level helpers — append `SampleLogEvent`; if `console=True`, also
    render and write to the console stream immediately (so check-modules stays interactive even
    before materialization)

### 3.5 Module function

`replay_sample_log_events(path, events, *, time_provider=None, console=False,
console_stream=None)` — open a backend, write each buffered event (preserving original
timestamps), close. Used on the failure-materialization path. Production failure replay keeps
`console=False` (failure artifacts are file-backed; interactive console already happened live
during the buffered phase when enabled).

### 3.6 As-built vs target (sample layer)

| Surface | V2 as-built (`d0de31a`) | V2 target (this §3) |
|---------|-------------------------|---------------------|
| File format + structured/raw | ✅ | same |
| `SampleLogEvent` / `replay_sample_log_events` | ✅ | same |
| `BufferedSampleLogger` discard/replay | ✅ | same |
| `_format_sample_log_text` shared helper | ❌ (inlined in backend) | extract for file/console parity |
| `console` / `console_stream` on open/buffer/replay | ❌ | ✅ |
| `_CONSOLE_LOCK` + live console on buffer path | ❌ | ✅ |
| Sample owns `sample_console` → passes `console=` | ❌ | ✅ (see [sample.md](sample.md)) |

---

## 4. Sample command log contract

This section is the behavioral contract for calculator command logging. The subprocess scheduler
executes processes; `SampleLogger` owns formatting and sinks.

### 4.1 Ownership

Per-sample command logs have **one** canonical owner: `SampleLogger` (file and optional console).

The subprocess scheduler may execute and drain processes, but it must not become a second
sample-log formatter. For calculator commands it emits into the active sample logger
(`SubprocessJob.stream_logger`) and lets `SampleLogger` decide where the rendered text goes.

### 4.2 File vs console policy

Default production scanning is **file-only**:

- sample logs are written to `SAMPLE/.../<uuid>/Sample_running.log`;
- sample command logs are **not** mirrored to the terminal;
- top-level process logs still use the top-level logger (`setup_jarvis_logging`).

`--check-modules` is the exception:

- check-modules test samples set an internal flag (`Sample.sample_console` / `info["sample_console"]`
  = `True`);
- `_make_lazy_sample_logger(..., console=True)` and `SampleLogger.open(..., console=True)` mirror
  the **same** rendered sample-log text to the terminal (default `sys.stderr`);
- console mirroring is implemented **inside** the sample logger (stream write + lock), not through
  loguru or the top-level process logger / queue listener.

This keeps check-modules interactive and verbose while keeping real scans quiet and file-backed.
The flag must **never** cross Redis (`to_task_dict` / wire format).

### 4.3 Command block shape

Calculator command output is a sequence of adjacent log writes, not a single buffered message:

```text
·•· Sample@<uuid> (<Calc>-No.<PackID>)
	-> MM-DD HH:mm:ss.mmm - [INFO] >>>
 Run execution command ->
	./run suspect2_lha.in
 in path ->
	/path/to/calculator
 Screen output ->

<raw stdout/stderr streamed in real time>
 Command Summary -> [execution#00003]
	 rc -> 0 	 dur -> 0.219s 	out -> 185B 	err -> 0B
```

Rules:

- the `Run ... command` header is a **normal formatted** sample-log entry (carries the Sample head);
- stdout/stderr are **raw** entries, streamed as they arrive;
- `Command Summary -> [...]` is also a **raw** entry (no second Sample head);
- the visual result looks like one command block even though output and summary are separate raw
  writes.

Stage labels for the header title map as:

| `meta.stage` | header title fragment |
|--------------|------------------------|
| `install` | `installation command` |
| `initialize` | `initialize command` |
| `execution` | `execution command` |
| (other/empty) | `command` |

### 4.4 Ordering contract

For a single command, the scheduler/worker must emit in this order:

1. formatted command header;
2. raw stdout/stderr stream lines (as produced);
3. raw command summary.

The summary must be emitted **before** the scheduler marks the job complete and before control
returns to the workflow. This prevents later workflow logs from appearing between the command
output and its summary.

The guarantee is **per command**. With concurrent commands from different samples, terminal output
may still interleave according to real execution timing. Each individual write is protected by the
sample logger file lock and/or `_CONSOLE_LOCK` so a rendered entry is not torn mid-write.

### 4.5 Scheduler metadata

Calculator jobs carry sample-command logging metadata on `SubprocessJob`:

| Field / `meta` key | Semantics |
|--------------------|-----------|
| `stream_logger` | the active per-Sample logger (`SampleLogger` or `BufferedSampleLogger`) |
| `meta.command_log_to_stream=True` | route command header/output/summary to `stream_logger` instead of only the scheduler logger |
| `meta.emit_command_summary=True` | scheduler emits a raw summary line after process drain, before returning the result |
| `meta.stage` | `install` / `initialize` / `execution` — labels header + summary |
| `meta.command_index` | integer counter; rendered as `[stage#00001]` |
| `meta.suppress_command_header` | optional; skip the formatted header when true |
| `meta.module` / `meta.pack_id` | used when binding the scheduler-side job logger label |

V2 may rename these fields, but must preserve the same semantics. See
[subprocess_scheduler.md](subprocess_scheduler.md) §4.1.

---

## 5. KV helper — `log_kv.py`

| Function | Behavior |
|----------|----------|
| `_wrap_cell(text, width) -> list[str]` | textwrap a cell. |
| `format_two_column_log(title, rows, *, max_width=80, min_value_width=16) -> str` | render an aligned two-column `Field \| Value` block for summary log output. |

---

## 6. Concurrency / lifecycle / failure semantics

- Each **process** calls `setup_jarvis_logging(role=…)` once; file names carry `pid` so Workers
  never share a handle.
- Per-Sample loggers are created/closed **within one Worker**; `sample.info["logger"]` is set to
  `None` before serialization (invariant #8). `sample_console` is Worker/control-local only.
- **Lazy success** → `BufferedSampleLogger.discard()`; **failure** → `replay_to(run_log)` then a
  file `SampleLogger` (invariant #9).
- The `QueueListener` runs on a dedicated thread; shutdown stops + drains it (no lost tail).
- Check-modules console mirroring is sample-local, opt-in, and **outside** the top-level queue
  listener. File writes use the per-backend lock; console writes use `_CONSOLE_LOCK`.
- `bind(raw=True)` (or binding any key named `raw`) switches subsequent writes to raw mode on that
  child adapter only.

---

## 7. Interfaces / collaborators

- **core / Worker / Factory / Archiver** → `setup_jarvis_logging(role=…)` +
  `get_jarvis_logger(...).bind(...)`.
- **Sample** ([sample.md](sample.md)) owns the per-Sample logger lifecycle (buffered → file) and
  the check-modules-only `sample_console` flag that is passed as `console=` into both logger
  constructors.
- **Subprocess Scheduler** ([subprocess_scheduler.md](subprocess_scheduler.md)) streams calculator
  command header/output/summary into `SampleLogger` for sample-local command logs.
- **CalculatorModule** ([calculator.md](calculator.md)) sets `stream_logger` + the `meta` keys in
  §4.5 when submitting jobs.
- **run_summary** ([monitor.md](monitor.md)) and various summaries use
  `log_kv.format_two_column_log`.

---

## 8. Drift from design

`shutdown_jarvis_logging` and the `_state`-based `QueueListener` management are new and concrete.
The "Sample layer" is a self-contained reimplementation (`SampleLogEvent`, `_SampleLogBackend`,
`replay_sample_log_events`), not a V1 re-export. `log_kv` is a new helper folded in here.

**Open implementation gap (carry V1 → V2):** as of `d0de31a`, V2 `sample_logger.py` has the file +
buffer path only. It still needs:

1. shared `_format_sample_log_text` + `_CONSOLE_LOCK`;
2. `console` / `console_stream` on `SampleLogger.open`, `BufferedSampleLogger`, and
   `replay_sample_log_events`;
3. live console writes on the buffered path (check-modules before materialize);
4. Sample-side `sample_console` plumbing into `_make_lazy_sample_logger` / `_open_sample_logger`;
5. calculator/scheduler command-block metadata (`command_log_to_stream`, `emit_command_summary`,
   ordering) aligned with §4.

Until those land, production file/replay behavior matches this doc; check-modules interactive
sample mirroring and raw command-summary adjacency do not.

---

## 9. Tests

`tests/test_logging_layers.py` (11 tests): two-layer separation, contract format, failure replay,
no cross-process logger. `tests/test_log_kv.py` (3 tests) covers `format_two_column_log`.

Future / required when §3 console + §4 command blocks land:

- default scan sample logs do **not** mirror to console;
- check-modules sample logs **do** mirror to console through `SampleLogger`, not the top-level
  logger;
- buffered-path console mirror works before materialization;
- failure `replay_to` does not double-print console lines;
- command stdout/stderr remain raw and real-time;
- `Command Summary` is raw and adjacent to the command output (emitted before job completion);
- concurrent sample console writes are not torn mid-entry (`_CONSOLE_LOCK`).
