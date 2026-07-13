# Component — CLI parsing & dispatch (命令行解析) (`jarvishep2/client.py`)

**Role**: the `Jarvis2` command-line surface. Parse args, dispatch to run / resume / monitor /
check-modules, and stay a **separate console-script** from V1's `Jarvis`.
**Status**: **As-built reviewed** @ `jarvis2` `0a5e85e` (Redis CLI flags removed; EnvReqs.V2).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §0.1; [core.md](core.md).
**Reuses V1**: none by import.

> **As-built drift:** the design described a declarative `card/argparser.json` and subcommands for
> `benchmark`/`convert`/`project`/`worker start`. **`--plot` exists** as a thin Jarvis-PLOT bridge
> (`jarvishep2/plot_bridge.py`). Other advanced subcommands are still absent. The CLI is a plain
> `argparse` parser in `client.py` with a single positional + a handful of flags. Entry point:
> `pyproject.toml` → `[project.scripts] Jarvis2 = "jarvishep2.client:main"`.

---

## 1. CLI surface (as-built)

| Invocation | Routes to | Notes |
|------------|-----------|-------|
| `Jarvis2 <task.yaml>` | `Jarvis2Core.run` | distributed run (stateless sampler) |
| `Jarvis2 <task.yaml> --resume` | `core.run(resume=True)` | drain queue, restore sampler state |
| `Jarvis2 <task.yaml> --check-modules` | `core.check_modules` | fixed-point calculator smoke |
| `Jarvis2 --monitor` | `dispatch_monitor` | print one monitor snapshot and exit |
| `Jarvis2 <plot.yaml> --plot` | `plot_bridge.run_plot` | render an existing JarvisPLOT scene; not a scan YAML |
| `--redis-host/--redis-port/--redis-db` | Redis overrides | folded into `Runtime.redis` |
| `--pid N` | **dead/reserved** | parsed but never consumed; must not be presented as working attach |

---

## 2. Module-level functions

| Function | Behavior |
|----------|----------|
| `build_parser() -> ArgumentParser` | define the positional `task_yaml` + flags above. |
| `_redis_cli_overrides(args) -> dict\|None` | non-default Redis flags as a dict (else None). |
| `_apply_redis_overrides(core, args) -> None` | merge overrides into `core.config["Runtime"]["redis"]` + `core.runtime`. |
| `run_monitor(*, factory=None, redis=None) -> int` | read + print a monitor view; 1 if no active scan. |
| `dispatch_monitor(args) -> int` | attach via the in-process factory snapshot, else a fresh `RedisQueue`. |
| `dispatch_run(args) -> int` | load YAML (clear errors on missing/invalid), apply Redis overrides, `check_modules()` or `run(resume=…)`; map exceptions to exit codes. |
| `dispatch(args) -> int` | route `--monitor` vs run; usage error if neither. |
| `main(argv=None) -> int` | `dispatch(build_parser().parse_args(argv))` — the console entry point. |

---

## 3. Lifecycle / failure semantics

- `main` is a thin synchronous front; all process spawning happens in `core`.
- User errors (missing/invalid YAML, unsupported task, runtime failure) → a clear message + a
  non-zero exit code (`1`/`2`), no traceback dump.
- A distinct console-script (`Jarvis2`) lets V1 `Jarvis` and V2 coexist.

### Known interface defects

- Mode flags are not mutually exclusive; dispatch silently prioritizes monitor, then plot, then
  check/run.
- Redis option defaults double as the “not supplied” sentinel, so explicitly passing the default
  host/port/db cannot override a different YAML value.
- `core.run_distributed_scan()` returns submitted count and archiving includes failed records;
  therefore all-sample failure can still produce CLI exit 0.
- Failed sample results do not expose stable `error_type`/`error` fields.
- There is no Portal or Operas discovery command. See
  [`../USER_INTERFACE_INTEGRATION_REVIEW_2026-07-13.md`](../USER_INTERFACE_INTEGRATION_REVIEW_2026-07-13.md)
  and plan D11.

---

## 4. Interfaces / collaborators

- **Jarvis2Core** ([core.md](core.md)) → `load_task_yaml`, `run`, `check_modules`.
- **dashboard** ([monitor.md](monitor.md)) → `attach_reader` / `format_monitor_view`.
- **TaskFactory** ([factory.md](factory.md)) / **RedisQueue** ([redis_queue.md](redis_queue.md)) →
  monitor attach.

---

## 5. Tests

`tests/test_cli.py` (9) and `tests/test_plot_bridge.py` (8): parse matrix, dispatch routing
(run / check-modules / monitor / plot), Redis overrides, user-error UX + exit codes, and PLOT
handoff. The known defects above do not yet have fixed-behavior tests.
