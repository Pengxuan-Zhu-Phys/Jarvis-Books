# Component — Core orchestrator (`jarvishep2/core.py`, `jarvishep2/client.py`)

**Role**: the `Jarvis2` entry point and run orchestrator. Wires config → Redis → Factory →
Workers → Archiver → Sampler, runs the scan, and tears everything down cleanly.
**Status**: design — spans D1.1 (boot a Redis run) → D4/D5/D6 (Archiver, monitor, resume).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §0.1, §2;
discussion `factory_design.md` §7.
**Reuses V1**: the V1 `core.py` init sequence shape (argparse → project → logger → config →
sampler → workflow → libraries → factory → likelihood → database → run_summary), adapted to the
distributed components.

---

## 1. Responsibilities

1. Parse `Jarvis2` CLI (separate entry point from `Jarvis`, invariant #2).
2. Load + validate config (frozen schema; new keys optional); run Phase-1 `CommandParser`.
3. Bring up the distributed runtime in the right order; run the scan; shut down draining all
   queues and finalizing the HDF5 file.
4. Handle `--resume`, `--monitor`, `--check-modules`, `--benchmark`, `--convert` as `Jarvis2`
   subcommands/flags.

---

## 2. Init sequence (distributed)

```
1.  init_argparser()            # Jarvis2 CLI
2.  init_project()              # dirs, &J/ root
3.  init_logger()               # setup_jarvis_logging(role="core")
4.  init_configparser()         # load + validate YAML (frozen schema)
5.  init_command_parser()       # Phase-1 static resolution + registered_executables
6.  init_redis()                # RedisQueue.connect + ensure keys
7.  _preload_resume_checkpoint()# resume? → drain queue, restore sampler state
8.  init_sampler()              # Distributor → sampler; set_redis(); load_variable()
9.  init_workflow()             # DAG + execution_plan_template + flowchart.json
10. init_libraries()            # install LibDeps (registered executables)
11. init_factory()              # TaskFactory: (reuse core Redis or init_redis), start_workers(n), start_monitor   [start_watchdog → D6.1, not yet]
12. init_archiver()             # Archiver process: spawn, attach DB config
13. init_run_summary()          # metrics from coordination/Redis at shutdown
14. run()                       # sampler proposes → Redis → Workers → Archiver
15. shutdown()                  # stop sampler, drain queue, stop workers, flush archiver, finalize HDF5
```

Worker config shipped at step 11 is **picklable** (calculator configs, opera list, workflow
blueprint, mapper config, Phase-1 resolved commands) — no live handles (spawn).

> **`Runtime.mode == "redis"` gate (current code).** The distributed bring-up (steps 11–12) only
> runs when `is_redis_runtime()` is true; otherwise `init_factory` logs *"Runtime.mode != redis;
> skipping TaskFactory bring-up"* and returns `None`. In the D1.1 implementation `init_factory`
> reuses the client created by `init_redis` (step 6) rather than opening a second one, then
> `start_workers(Runtime.workers)` + `start_monitor(update_hz=2.0)`. `shutdown` mirrors this:
> stop the Archiver, `factory.shutdown(wait=…)` (which also closes Redis), else close Redis
> directly.

---

## 3. Member functions (selected)

| Method | Behavior |
|--------|----------|
| `init_redis` | Build `RedisQueue` from `Runtime.redis`; `connect()`. **(current code)** with no `redis` config and no injected client it falls back to `make_fakeredis_queue()` (CI / tests). The same client is later reused by `init_factory`/`init_archiver`. |
| `is_redis_runtime` | `Runtime.mode == "redis"` (case-insensitive; default `"auto"`). Gates `init_factory`. |
| `init_factory` | **(current code)** No-op unless `is_redis_runtime()` (`Runtime.mode == "redis"`, else logs and returns `None`). Obtain `TaskFactory.get_instance(redis_config)`; **reuse the core's Redis client** (`factory.redis = self.redis`) or `factory.init_redis()`; `start_workers(Runtime.workers, **worker_config)` (defaults to 1, clamped ≥ 1); `start_monitor(update_hz=2.0)`. *No `start_watchdog` yet (D6.1).* |
| `init_archiver` | Spawn `Archiver(Process)` with DB + `Calculators.Archiver` config. |
| `run` | Drive the sampler loop: propose → `push_task`/`push_many_tasks`; for MCMC, consume result feedback; throttle on queue high-water. |
| `monitor` | Launch the dashboard (`--monitor [--pid N]`) → `SnapshotReader`. |
| `shutdown` | Ordered teardown: stop proposing → wait queue drained → stop Workers (graceful) → flush Archiver → finalize HDF5 → write run_summary → close Redis. |
| `check_modules` | `Jarvis2 … --check-modules`: 10-sample smoke through the real distributed path (1 Worker). |
| `benchmark` | `Jarvis2 … --benchmark [s]`: timed throughput + `parallelism_efficiency` over the distributed path. |

---

## 4. Shutdown ordering (correctness-critical)

```
sampler.stop_proposing()
wait until hep:task_queue drained AND no Worker has an in-flight Sample
factory.stop_all_workers(graceful=True)        # finish current Sample, release slots
archiver.flush(); archiver.stop()              # drain hep:archive_queue, finalize HDF5
write run_summary; factory.shutdown(); redis.close()
```

Out-of-order teardown is the classic source of lost tail records — the gate is "queue drained
+ no in-flight" before stopping Workers, and "archive queue drained + file finalized" before
exit.

---

## 5. Concurrency / lifecycle / failure semantics

- `spawn` context for all child processes (Workers, Archiver, optional writer).
- `SIGINT`/`SIGTERM` → graceful: stop proposing, drain, finalize, then exit (no corruption).
- A fatal child failure (e.g. writer crash) → stop-and-checkpoint per design §10.
- `--resume` runs step 7 before sampler/factory bring-up: rebuild pools, drain stale tasks,
  restore sampler state.
- Spawn-precondition failure (unpicklable config / unimportable user funcs) → clear error at
  boot (V2 has no thread fallback; this is a hard, early failure, not a silent downgrade).

---

## 6. Interfaces

Ties together every other component: **RedisQueue**, **TaskFactory**, **Worker**(s),
**Archiver**, **Sampler**, **Workflow**, **CommandParser**, **Checkpoint**, **Monitor**,
**run_summary**.

---

## 7. Tests (`tests/test_core_run_distributed.py`, `fakeredis` + real spawn)

Integration:
1. **End-to-end opera** — `Jarvis2 <opera>.yaml` runs to completion; DATABASE set-equal to V1
   golden; clean shutdown, no orphan processes.
2. **End-to-end calculator** — `tests/parity_project` via `Jarvis2`; DATABASE+SAMPLE parity.
3. **Shutdown ordering** — inject a slow Archiver; assert no tail record is lost (queue + file
   fully drained before exit).
4. **Resume** — kill mid-scan, `Jarvis2 --resume`; final golden set complete, no dup.
5. **Boot preconditions** — unpicklable config → clear boot error (no hang, no silent
   downgrade).
6. **CLI surface** — `Jarvis2` exists and is independent of `Jarvis`; `--check-modules`,
   `--benchmark`, `--convert`, `--monitor` route correctly.

Verification logic: tests 1–4 are the system-level gates (parity + crash safety + clean
teardown); test 5 enforces the "fail loud at boot, never silently downgrade" rule that replaces
V1's thread fallback.

---

## 8. Open questions

- Packaging: `Jarvis2` console-script + distribution name (`Jarvis-HEP2`) for side-by-side
  install (design §0.1; resolve in D0.2/D1.1).
- Whether `--check-modules`/`--benchmark` can run without a Redis server via an embedded
  `fakeredis` mode for pure smoke tests (convenient for CI).
