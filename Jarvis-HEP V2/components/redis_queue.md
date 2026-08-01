# Component — RedisQueue & Schema (`jarvishep2/redis_queue.py`, `jarvishep2/calculator_pools.py`)

**Role**: the single cross-process broker. Task queue, calculator free-pools, status hashes,
results, and `op_count` change counters — all through one thin, tested access layer.
**Status**: **As-built** @ `jarvis2` **`7c482ec`** (D1.1 sample `op_count` contract clarified in
`2c8351f` + later bucket/isolation work).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §7; discussions
`factory_design.md` §4/§5, `worker_design.md` §8.
**Reuses V1**: none (new). Replaces the in-process queues/locks of the V1 thread runtime.

---

## 1. Modules

- `redis_queue.py` — key namespace, codecs, payload validators, and the `RedisQueue` class.
- `calculator_pools.py` — two helpers that translate Worker config into calc-pool registration.

---

## 2. Key namespace (module constants)

```python
TASK_QUEUE     = "hep:task_queue"          # List   Sampler rpush / Worker blpop
CALC_FREE      = "calc:free:{name}"        # List   exclusive PackID slots ("001"…"N")
CALC_BUSY_PACKS= "calc:busy:{name}"        # Hash   pack_id -> "running"
RESULTS        = "hep:results:{uuid}"      # Hash   optional direct result handoff
ARCHIVE_QUEUE  = "hep:archive_queue"       # List   Worker → Archiver
FEEDBACK_QUEUE = "hep:feedback"            # List   sampler barriers (D10); projected payload (D13.8)
WORKER_STATUS  = "hep:worker:status:{id}"  # Hash   heartbeat fields
CALC_STATUS    = "hep:calculator:status"   # Hash   "<name>:free":n / "<name>:busy":n
SAMPLE_STATS   = "hep:sample:stats"        # Hash   running/completed/failed
OP_COUNT       = "hep:{kind}:op_count"     # Str    kind ∈ {worker,calculator,sample,task}
CONTROL_LOCK   = "hep:control:lock"        # Str    single control-process lease
# SAMPLE buckets (V1 sample_directory parity)
BUCKET_META    = "hep:sample:bucket:meta"  # Hash   current/count/limit/width/sample_root/pack
BUCKET_STATE   = "hep:sample:bucket:state:{id}"  # Hash active/completed/assigned/archived/sealed/packing/packed
BUCKET_READY   = "hep:sample:bucket:ready" # List   sealed buckets ready to tar
BUCKET_LOCK    = "hep:sample:bucket:lock"  # short mutex for allocate/finish/note
```

Validation sets: `_VALID_OP_KINDS`, `_VALID_SAMPLE_ARTIFACTS` (auto/always/never),
`_VALID_RESULT_STATUSES` (Created/Init/Running/Completed/Failed), `_VALID_EXECUTION_STEP_TYPES`.

---

## 3. Classes defined

### 3.1 `CodecError(RuntimeError)` — payload encode/decode failure.
### 3.2 `TaskValidationError(ValueError)` — task/result schema validation failure.

### 3.3 `RedisQueue`
Redis broker for tasks, calculator pools, results, and monitor counters.

**Attributes:** `config: dict` (connection settings), `_codec: str` (`json`|`msgpack`), `r`
(the live redis client, or `None` until `connect()` / injection).

**Member functions:**

| Method | Signature | Behavior |
|--------|-----------|----------|
| `__init__` | `(config=None, *, client=None)` | store config + codec; client injectable for tests. |
| `connect` | `() -> None` | build a `redis.Redis` from `url` or `host/port/db`; `decode_responses` when codec is json; no-op if a client is injected. |
| `push_task` | `(task) -> None` | validate, `rpush(TASK_QUEUE)` + `incr` **task** op_count in one transaction. |
| `pull_task` | `(timeout=5) -> dict\|None` | `blpop(TASK_QUEUE)`; decode to dict or `None`. On success: **`hincrby(SAMPLE_STATS, "running", 1)` only** — **does not** bump sample `op_count` (D1.1). |
| `drain_task_queue` | `() -> int` | discard all queued tasks (resume path D6.2); returns count. |
| `push_many_tasks` | `(tasks) -> None` | validate all, pipeline `rpush` + single `incrby` (**task** op_count). |
| `register_calc_pool` | `(name, n) -> None` | reset `CALC_FREE`/`CALC_BUSY_PACKS`, seed **stable PackIDs `001`…`N`** (`format_calc_pack_id`), set status free=n busy=0. |
| `acquire_calc` | `(name, timeout=30) -> str\|None` | `blpop` a slot → the popped value **is** the stable PackID (no minting; legacy `ready` junk discarded), mark busy, flip free/busy counters, bump **calculator** op_count. |
| `release_calc` | `(name, pack_id) -> None` | Lua-atomic `HDEL` guard + `RPUSH` + free/busy counter updates + calculator op count; a second release is reported as already free, while Redis errors propagate. Worker held-pack state is removed only after confirmed release. |
| `submit_result` | `(info) -> None` | validate, `rpush(ARCHIVE_QUEUE)`, update `SAMPLE_STATS` (completed/failed + running−1), **exactly one** `incr` **sample** op_count per sample (D1.1). |
| `pull_result` | `(timeout=1) -> dict\|None` | `blpop(ARCHIVE_QUEUE)` for the Archiver. |
| `init_sample_buckets` | `(sample_root, limit, width, …)` | reset bucket meta for a run. |
| `allocate_sample_bucket` | `() -> dict\|None` | assign `SAMPLE/<bucket>/`, active++/assigned++; may seal previous bucket. |
| `finish_sample_bucket` | `(bucket_id) -> bool` | Worker finished sample (active--, completed++); pack only if also fully archived. |
| `note_sample_archived` | `(bucket_id) -> bool` | Archiver wrote DATABASE row (archived++); may enqueue pack. |
| `seal_current_sample_bucket` | `() -> bool` | end-of-run seal of open bucket. |
| `pull_ready_bucket` / `mark_bucket_packed` | pack lifecycle for Archiver. |
| `acquire_control_lock` / `release_control_lock` / `reset_run_ephemeral_keys` | multi-instance isolation. |
| `heartbeat` | `(worker_id, **fields) -> None` | `hset(WORKER_STATUS)` (encodes non-scalars to JSON, derives `last_heartbeat`), bump worker op_count. |
| `get_op_count` | `(kind) -> int` | read one op_count (validates kind). |
| `get_all_op_counts` | `() -> dict` | all four op_counts in one pipeline. |
| `get_queue_lengths` | `() -> dict` | task + archive queue lengths in one pipeline. |
| `fetch_calculator_status` | `() -> dict` | read `CALC_STATUS` hash (numeric-coerced). |
| `fetch_sample_stats` | `() -> dict` | read `SAMPLE_STATS` hash. |
| `fetch_worker_status` | `(worker_ids) -> dict` | pipeline `hgetall` each `WORKER_STATUS`. |
| `sweep_held_calc_slots` | `(held_packs) -> int` | release calc slots held by a dead Worker (D6.1); returns count. |
| `encode_task_for_heartbeat` | `(task) -> str` | serialize an in-flight task for the heartbeat hash. |
| `decode_heartbeat_task` | `(heartbeat) -> dict\|None` | decode `current_task` from a heartbeat. |
| `decode_heartbeat_held_packs` | `(heartbeat) -> dict` | decode `held_calc_packs`. |
| `incr_op` | `(kind, amount=1) -> int` | `incrby` an op_count. |
| `snapshot_raw` | `() -> dict` | queue lengths + calc status + sample stats + op_counts (Factory monitor read). |
| `store_result_hash` | `(uuid, info) -> None` | optional per-uuid `RESULTS` hash for debug/sync. |
| `connection_config` | `() -> dict` | picklable settings for spawn children. |
| `extract_connection_config` | `@staticmethod (source) -> dict` | normalize a live queue or mapping into settings. |
| `close` | `() -> None` | best-effort close + drop the handle. |
| `_require_client` | `() -> None` | raise if `r is None`. |

---

## 4. Module-level functions

| Function | Behavior |
|----------|----------|
| `calc_free_list_key(name)` / `calc_busy_packs_key(name)` | format the per-calculator list/hash keys. |
| `calc_status_free_field(name)` / `calc_status_busy_field(name)` | `"<name>:free"` / `":busy"` hash fields. |
| `_json_default(obj)` | JSON encoder for ndarray/np.generic/UUID/dataclass. |
| `encode_payload(payload, *, codec="json")` / `decode_payload(raw, *, codec)` | json or msgpack (import-guarded → `CodecError`). |
| `_encode_heartbeat_value(value)` | scalars pass through; others JSON-encoded. |
| `_validate_u_coords` / `_validate_execution_plan` / `_validate_task_payload` / `_validate_result_payload` | schema guards raising `TaskValidationError`. |
| `_coerce_numeric(value)` | str → int/float/str for status reads. |
| `make_fakeredis_queue(**config) -> RedisQueue` | **public**: `RedisQueue` backed by `fakeredis.FakeStrictRedis` (unit tests / CI). |

### `calculator_pools.py`
| Function | Behavior |
|----------|----------|
| `resolve_calculator_pools(worker_config) -> dict[str,int]` | explicit `calculator_pools`, else per-module `make_paraller` (fallback `calculator_make_paraller` or 1). |
| `register_calculator_pools(redis, worker_config) -> None` | call `register_calc_pool` for each resolved pool. |

---

## 5. Concurrency & failure semantics

- Multi-key writes that must be atomic use a **pipeline/transaction** (e.g. `acquire_calc` slot +
  status flip + op_count). Single-key ops are naturally atomic.
- `blpop` is the **backpressure** mechanism: empty queue ⇒ Worker blocks (timeout loop), no spin.
- Connection loss → methods **raise**; callers decide retry (no silent swallow).
- `op_count` is **monotonic** (a global change counter, not a gauge). Factory monitor refreshes
  subsystem hashes only when the matching kind advances.

### 5.1 D1.1 sample `op_count` contract (binding)

| Event | `SAMPLE_STATS` | `hep:sample:op_count` |
|-------|----------------|------------------------|
| Task dequeued (`pull_task`) | `running += 1` | **no change** |
| Result submitted (`submit_result`) | `completed` or `failed` += 1; `running -= 1` | **+1 once** |
| Sampler in-flight backpressure | reads `fetch_sample_stats()` | does **not** require op bump |

Rationale: one sample must not double-count (pull + submit). Backpressure and dashboards that
need in-flight load use the stats hash; `op_count` only signals “sample terminal outcome
changed” for op_count-gated monitor refresh.

---

## 6. Interfaces / collaborators

- **Sampler** → `push_task` / `push_many_tasks`.
- **Worker** ([worker.md](worker.md)) → `pull_task`, `acquire_calc`/`release_calc`,
  `submit_result`, `heartbeat`.
- **Factory** ([factory.md](factory.md)) → read-only `snapshot_raw`, `fetch_*`,
  `sweep_held_calc_slots`, `register_calc_pool` via `calculator_pools`.
- **Archiver** ([datarecorder.md](datarecorder.md)) → `pull_result`.
- **Monitor** ([monitor.md](monitor.md)) → `snapshot_raw` / `get_queue_lengths`.

---

## 7. Drift from design

Schema grew: `CALC_BUSY_PACKS` (pack_id ownership) is new and underpins `release_calc` validation
and the D6.1 dead-Worker slot sweep (`sweep_held_calc_slots`). Added: explicit
`CodecError`/`TaskValidationError`, payload validators, `drain_task_queue`, the batched
`get_all_op_counts`/`get_queue_lengths`/`fetch_*` monitor reads, heartbeat task/held-pack
codecs, `connection_config`/`extract_connection_config` (spawn picklability), SAMPLE bucket
keys + control lock / ephemeral reset, and the `calculator_pools.py` helpers.

**Design wording vs as-built:** design §7 says sample `op_count` increments on “sample status
change.” As-built **narrows** that to **terminal** outcomes in `submit_result` only (§5.1);
in-flight (`running`) is a gauge on `SAMPLE_STATS`, not an op_count event.

---

## 8. Tests

`tests/test_redis_queue.py` (16 tests, `fakeredis`): task round-trip, op_count, calc-pool cap
& distinct pack_ids, pipeline atomicity, result handoff, codec swap. Pool registration is also
covered by `tests/test_worker_pool.py` and the distributed acceptance suite.
