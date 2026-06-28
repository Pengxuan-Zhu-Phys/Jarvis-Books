# Component — Sample (`jarvishep2/sample.py`)

**Role**: the unit of work. Carries a parameter point from the Sampler, across Redis, into a
Worker, through the workflow, and out to the Archiver.
**Status**: design — plan WP-D0.1.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §3; discussions
`worker_design.md` §3/§5, `…架构升级…` §3.1.
**Reuses V1**: extends the lazy `Sample(Base)` (`jarvishep/sample.py`) — keeps `materialize()`,
`create_info()`, `logger_name` metadata, failure-replay. Does **not** rewrite it.

---

## 1. Responsibilities

1. Hold the **identity + raw draw**: `uuid`, `u_coords` (normalized sampler vector).
2. Be **serializable into a light task dict** for Redis (no live handles, no logger).
3. **Rebuild itself inside the Worker** (`from_task_dict`) and **materialize** dirs/logger
   there (Worker-side only).
4. Carry the **execution plan** (which calculators/operas in which layer).
5. Produce a **result/info dict** for the Archiver + monitor.
6. Resolve `@SampleID` / `@Sdir` / `@PackID` tokens through `info` (compat with V1 modules).

What it must **not** do: map `u → x` (that is the Worker's global `UMapper`), open files on the
control side, or hold a `logger` that crosses processes.

---

## 2. Class structure

```python
@dataclass
class Sample:
    uuid: str                                   # UUID4 string, primary key
    u_coords: np.ndarray                         # normalized draw (the only heavy field)
    execution_plan: list[ExecutionStep]          # filled by the workflow/sampler
    sample_artifacts: str = "auto"               # auto|always|never (Runtime)
    # --- worker-side, NOT serialized ---
    params: dict[str, float] = field(default_factory=dict, repr=False)   # x-space, lazy
    info: dict = field(default_factory=dict, repr=False)                 # V1-compat bag
    observables: dict = field(default_factory=dict, repr=False)
    status: str = "Created"                      # Created|Running|Completed|Failed
    _materialized: bool = field(default=False, repr=False)
    _logger: "SampleLogger | BufferedSampleLogger | None" = field(default=None, repr=False)
```

`ExecutionStep` (light, JSON-able):

```python
@dataclass
class ExecutionStep:
    type: str            # "calculator" | "opera" | "likelihood" | "nuisance_optimize"
    name: str            # module / calculator name
    layer: int           # DAG layer index (same layer ⇒ may run concurrently)
    params: dict = field(default_factory=dict)
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `to_task_dict` | `() -> dict` | Light JSON-able dict: `uuid`, `u_coords` (→ list), `execution_plan`, `sample_artifacts`. **Excludes** `params`, `info`, `_logger`. The wire format for `hep:task_queue`. |
| `from_task_dict` | `(cls, d: dict) -> Sample` | Reconstruct in the Worker. `u_coords` back to `np.ndarray`; `_materialized=False`. |
| `to_info_dict` | `() -> dict` | Result projection for Archiver + monitor: `uuid`, `status`, `params`, `observables`, `likelihood`, timings. No logger/handles. |
| `bind_params` | `(mapper: UMapper) -> None` | Worker-side `u → x`; fills `params` and the V1 `info` bag. Idempotent. |
| `materialize` | `(worker_id: str | None = None, *, failure_message=None) -> str` | Worker-side: create `SAMPLE/<bucket>/<uuid>/`, open per-Sample `SampleLogger`; sets `info["save_dir"]`. Reuses V1 logic. No-op if already materialized. Returns `save_dir`. |
| `materialize_failure_artifacts` | `(error=None) -> str | None` | On failure: materialize + replay the `BufferedSampleLogger` to `Sample_running.log` (V1 contract, invariant #9). |
| `resolve_token` | `(text: str, *, stage, field) -> str` | Replace `@SampleID/@Sdir/@PackID`; triggers `materialize()` on first `@Sdir`. |
| `start` / `close` | `() -> None` | Status transitions + logger open/close (Worker-side). Carries the fixed V1 `==`/`=` status bug. |
| `record` | `() -> dict` | Final record handed to the Archiver (observables + metadata). |

**Properties** (read-only): `uuid`, `u`, `likelihood` (getter/setter), `save_dir` (lazy →
triggers `materialize`).

---

## 4. Serialization contract (the critical surface)

`to_task_dict()` output is the **only** thing that crosses into Redis:

```json
{ "uuid": "…", "u_coords": [0.1, 0.2], "sample_artifacts": "auto",
  "execution_plan": [ {"type":"calculator","name":"DemoCalc","layer":0},
                      {"type":"opera","name":"L","layer":1} ] }
```

Round-trip law: `Sample.from_task_dict(s.to_task_dict())` reproduces `uuid`, `u_coords`
(allclose), and `execution_plan` exactly. `params`/`info`/`observables`/`_logger` are **not**
preserved (they are Worker-local, rebuilt or recomputed).

---

## 5. Lifecycle & failure semantics

```
Sampler:   Sample(uuid,u_coords) → attach execution_plan → to_task_dict → Redis
Worker:    from_task_dict → bind_params(mapper) → materialize(worker_id)
           → run workflow (sets observables/likelihood/status)
           → success: to_info_dict → Archiver ;  buffer discarded (lazy)
           → failure: materialize_failure_artifacts → to_info_dict(status=Failed) → Archiver
           → close (logger closed)
```

- **Lazy artifacts** (M1 carry-over): `sample_artifacts: auto` + opera-only success ⇒ no
  filesystem footprint; non-materialized samples log into a `BufferedSampleLogger`.
- **Never** keep a `_logger` when entering `to_task_dict` (assert in tests).

---

## 6. Interfaces

- **Sampler** builds it (`uuid` + `u_coords`) and attaches `execution_plan` (from Workflow).
- **Worker** owns its full Worker-side lifecycle (`bind_params`, `materialize`, run, `close`).
- **UMapper** (global, Worker-held) does `u → x` via `bind_params`.
- **Archiver** consumes `to_info_dict()` / `record()`.

---

## 7. Tests (`tests/test_sample_taskdict.py`)

Unit:
1. **Round-trip** — `from_task_dict(to_task_dict())` preserves `uuid`, `u_coords` (allclose),
   `execution_plan`; drops `params/info/logger`.
2. **No-logger-on-wire** — after attaching a logger, `to_task_dict()` contains no logger key
   and is `json.dumps`-able.
3. **Lazy materialization** — opera-only sample, `auto`, success → `SAMPLE/` empty (reuse the
   V1 `test_lazy_materialization` assertions).
4. **Failure replay** — force a failure → `Sample_running.log` exists and matches buffered
   events (`docs/specs/LOGGING_CONTRACT.md` format).
5. **Token resolution** — `@Sdir` in a string triggers `materialize()` and resolves under
   `save_dir`.
6. **Status transitions** — `Created→Running→Completed/Failed` (guards the V1 `==`/`=` bug).

Verification logic: round-trip and no-logger are property-style (random `u_coords`, unicode
observable names). Parity: a Worker-run sample's `record()` is **set-equal** to the captured
V1 golden record for the same `u_coords`.

---

## 8. Open questions

- Keep `info` dict as the V1-compat bag indefinitely, or migrate modules to typed accessors?
  (Proposed: keep for D0–D4, revisit.)
- `u_coords` dtype/precision on the wire (float64 vs float32) — pin float64 for parity.
