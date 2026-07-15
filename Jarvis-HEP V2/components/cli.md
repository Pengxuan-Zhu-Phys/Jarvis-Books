# Component — CLI parsing & dispatch (命令行解析) (`jarvishep2/client.py`)

**Role**: the `Jarvis2` command-line surface. One-intent subcommands + legacy bare-YAML aliases.
**Status**: **As-built** (D11 + D12.5–D12.6 project tools). Entry:
`Jarvis2 = jarvishep2.client:main`.
**Design refs**: [core.md](core.md); **project / encrypt-decrypt**: [project_tools.md](project_tools.md).

---

## 1. CLI surface (as-built)

| Invocation | Routes to | Notes |
|------------|-----------|-------|
| `Jarvis2 run <task.yaml> [--resume]` | `Jarvis2Core.run` | preferred run intent |
| `Jarvis2 <task.yaml>` | → `run` | legacy alias via `normalize_argv` |
| `Jarvis2 check <task.yaml>` | `check_modules` | fixed-point smoke |
| `Jarvis2 monitor` | `dispatch_monitor` | one snapshot |
| `Jarvis2 plot <plot.yaml>` | `plot_bridge` | JarvisPLOT scene |
| `Jarvis2 portal …` | Portal CLI (V2 registry) | same as `jportal` |
| `Jarvis2 operas list\|info` | Operas discovery | |
| `Jarvis2 project …` | scaffold / pack / catalog / **encrypt-decrypt** | see [project_tools.md](project_tools.md) |
| `--pid N` | **rejected** | exit usage |

### Project subcommands (encrypt/decrypt usage)

| Command | Role |
|---------|------|
| `project list` / `browse` | Catalog table: **Access** + **Key** (no \| required) |
| `project fetch NAME [--key K]` | Download; auto-decrypt restricted packs |
| `project pack PATH --encrypt --key K` | Pack + write `*.jenc` |
| `project encrypt FILE --key K` | Encrypt existing tarball |

Full recipes and catalog schema: [project_tools.md](project_tools.md) §3–4;
install guide: `Jarvis-HEP-v2/INSTALL.md` § Project tools.

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
