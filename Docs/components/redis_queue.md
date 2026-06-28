# Component — RedisQueue & Schema (`jarvishep2/redis_queue.py`)

**Role**: the single cross-process broker. Task queue, calculator free-pools, status hashes,
results, and `op_count` change counters — all through one thin, tested access layer.
**Status**: design — plan WP-D0.2.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §7; discussions
`factory_design.md` §4/§5, `worker_design.md` §8.
**Reuses V1**: none (new). Replaces in-process queues/locks of the V1 thread runtime.

---

## 1. Responsibilities

1. Own the **key namespace** (constants) and **serialization** (JSON now, msgpack hook).
2. Provide **blocking** task and slot primitives (`blpop`/`rpush`).
3. Provide **status writes** (worker/calculator/sample hashes) + **`op_count` INCR**.
4. Be **swappable for `fakeredis`** so unit tests need no server.
5. Hold **no business logic** — Worker/Factory/Sampler decide *what* to push; this layer
   decides *how* it is stored.

---

## 2. Key namespace (constants)

```python
TASK_QUEUE          = "hep:task_queue"            # List   (Sampler rpush, Worker blpop)
CALC_FREE           = "calc:free:{name}"          # List   (slot pool; blpop/rpush)
RESULTS             = "hep:results:{uuid}"        # Hash   (optional direct result handoff)
ARCHIVE_QUEUE       = "hep:archive_queue"         # List   (Worker → Archiver handoff)
WORKER_STATUS       = "hep:worker:status:{id}"    # Hash   {status,current_sample,heartbeat,pid}
CALC_STATUS         = "hep:calculator:status"     # Hash   {"<name>:free":n,"<name>:busy":n}
SAMPLE_STATS        = "hep:sample:stats"          # Hash   {running,completed,failed}
OP_COUNT            = "hep:{kind}:op_count"        # Str    kind ∈ {worker,calculator,sample,task}
```

---

## 3. Class structure

```python
class RedisQueue:
    def __init__(self, config: dict | None = None, *, client=None): ...
    # client injectable so tests pass a fakeredis.FakeStrictRedis
```

Attributes: `r` (the redis client), `config`, `_codec` (json|msgpack).

---

## 4. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `connect` | `() -> None` | Build the client from `config` (`host/port/db/url`); ensure keys exist. |
| `push_task` | `(task: dict) -> None` | `rpush(TASK_QUEUE, encode(task))` then `incr_op(\"task\")`. |
| `pull_task` | `(timeout=5) -> dict | None` | `blpop(TASK_QUEUE, timeout)`; decode or `None` on timeout. |
| `push_many_tasks` | `(tasks: list[dict]) -> None` | Pipeline `rpush` + a single `incrby(task)`. |
| `acquire_calc` | `(name, timeout=30) -> str | None` | `blpop(CALC_FREE.format(name))` → returns a fresh `pack_id`; updates `CALC_STATUS` busy/free + `incr_op("calculator")`. |
| `release_calc` | `(name, pack_id) -> None` | `rpush` the slot back; flip status; `incr_op("calculator")`. |
| `register_calc_pool` | `(name, n) -> None` | Seed `CALC_FREE` with `n` ready tokens (Worker startup). |
| `submit_result` | `(info: dict) -> None` | Push to `ARCHIVE_QUEUE` (+ optionally `RESULTS`); update `SAMPLE_STATS`; `incr_op("sample")`. |
| `pull_result` | `(timeout=1) -> dict | None` | `blpop(ARCHIVE_QUEUE)` for the Archiver. |
| `heartbeat` | `(worker_id, **fields) -> None` | `hset(WORKER_STATUS)`; `incr_op("worker")`. |
| `get_op_count` | `(kind) -> int` | `int(get(OP_COUNT) or 0)`. |
| `incr_op` | `(kind) -> None` | `incr(OP_COUNT.format(kind=kind))`. |
| `snapshot_raw` | `() -> dict` | Cheap reads used by the Factory monitor: queue length, status hashes, stats. |

Encoding: `encode/decode` handle numpy arrays (→ list), UUID (→ str), dataclasses. Pluggable
`_codec`.

---

## 5. Concurrency & failure semantics

- All multi-key writes that must be atomic use a **pipeline/transaction** (e.g. acquire_calc
  status flip + op_count). Single-key ops are naturally atomic.
- `blpop` is the backpressure mechanism: empty queue ⇒ Worker blocks (timeout loop), no spin.
- **Connection loss**: methods raise; callers (Worker/Factory) decide retry/backoff. The
  layer itself does not silently swallow (avoids the V1 bare-`except` traps).
- **`op_count` is monotonic** and never reset during a run — it is a global version number,
  not a gauge.

---

## 6. Interfaces

- **Sampler** → `push_task` / `push_many_tasks` (+ implicit task op_count).
- **Worker** → `pull_task`, `acquire_calc`/`release_calc`, `submit_result`, `heartbeat`.
- **Factory** → read-only `get_op_count` + `snapshot_raw` (never writes during normal run).
- **Archiver** → `pull_result`.

---

## 7. Tests (`tests/test_redis_queue.py`, `fakeredis`)

Unit (no server):
1. **Task round-trip** — `push_task(d); pull_task()==d`; numpy/UUID survive encode/decode.
2. **op_count** — each `push_task` bumps `hep:task:op_count` by exactly 1; idle reads don't.
3. **Calc pool cap** — seed `n=2`; 3 concurrent `acquire_calc` → only 2 succeed until a
   `release_calc`; `CALC_STATUS` free/busy consistent; each acquire yields a distinct
   `pack_id`.
4. **Pipeline atomicity** — acquire flips status + op_count together (no interleaving leaves a
   half state).
5. **Result handoff** — `submit_result` lands on `ARCHIVE_QUEUE`; `SAMPLE_STATS` increments.
6. **Codec swap** — same suite passes under json and msgpack.

Integration (`skipif REDIS_URL` unset): the same suite against a real server; plus a
two-process producer/consumer moving ≥10⁴ tasks with no loss/reorder.

Verification logic: the cap test is the load-bearing one — it proves global calculator
concurrency control works across processes (the Redis form of the Blueprint semaphore).

---

## 8. Open questions

- Durability: rely on Redis RDB/AOF, or treat the queue as volatile and re-drive from the
  Sampler on resume? (Design §13.5 — default: volatile + re-drive.)
- `RESULTS` hash vs `ARCHIVE_QUEUE` list as the canonical result path (default: list to the
  Archiver; `RESULTS` only for synchronous debug).
