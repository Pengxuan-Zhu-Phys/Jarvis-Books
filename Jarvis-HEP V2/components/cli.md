# Component — CLI parsing & dispatch (命令行解析) (`jarvishep2/client.py`)

**Role**: the `Jarvis2` command-line surface. One-intent subcommands + legacy bare-YAML aliases.
**Status**: **As-built** (2026-07-16) — D11 intents + D12 project tools + **version / ps / kill** +
console logging flags. Entry: `Jarvis2 = jarvishep2.client:main`.
**Design refs**: [core.md](core.md); **project / encrypt-decrypt**: [project_tools.md](project_tools.md);
**logging**: [logger.md](logger.md).

---

## 1. CLI surface (as-built)

| Invocation | Routes to | Notes |
|------------|-----------|-------|
| `Jarvis2 -v` / `--version` | `dispatch_version` | logo + Author + Version (**no** scan init) |
| `Jarvis2 run <task.yaml> [flags]` | `Jarvis2Core.run` | full scan; validates YAML first |
| `Jarvis2 <task.yaml>` | → `run` | legacy alias via `normalize_argv` |
| `Jarvis2 check <task.yaml>` | `check_modules` | smoke: **1 worker**, `SAMPLE/test/<uuid>/` flat, **no tar**; CSV or N draws |
| `Jarvis2 validate <task.yaml> [--strict] [--json]` | `dispatch_validate` | D13.9 config gate only (no Redis) |
| `Jarvis2 monitor` | `dispatch_monitor` | one Redis snapshot |
| `Jarvis2 plot <plot.yaml>` | `plot_bridge` | JarvisPLOT scene |
| `Jarvis2 portal …` | Portal CLI (V2 registry) | same as `jportal` |
| `Jarvis2 operas list\|info` | Operas discovery | |
| `Jarvis2 project …` | scaffold / pack / catalog / encrypt | [project_tools.md](project_tools.md) |
| `Jarvis2 ps` | `list_running_jarvis_cli` | list **running** Jarvis OS processes |
| `Jarvis2 kill [--yes]` | `kill_running_jarvis_cli` | kill after **interactive confirm** |
| `--pid N` | **rejected** | exit usage |

### Version banner (`-v` / `--version`)

Same contract as V1 help text:

> Print Jarvis-HEP logo, author information and runtime package version

Implementation: `dispatch_version()` → `render_logo_with_version()` (`jarvishep2/card/logo` +
package / pyproject version). Prints to stdout and exits **0**. Does **not** touch Redis,
Workers, or task YAML.

### Process inspect / kill (`jarvishep2/process_cleanup.py`)

Useful **during a live scan** (second terminal) or after a hard stop. Titles use `setproctitle`
and start with **`Jarvis2`** or **`Jarvis-Redis`**.

| Command | Behavior |
|---------|----------|
| `Jarvis2 ps` | Print `Running Jarvis processes (N)` + PID + title |
| `Jarvis2 kill` | Show list, prompt `Kill N process(es)? [y/N]`, then SIGTERM → SIGKILL |
| `Jarvis2 kill --yes` / `-y` | Skip prompt (scripts; **required** when stdin is not a TTY) |
| `Jarvis2 kill --no-force` | SIGTERM only |

Kill order: Worker → FileOperation → Archiver → control → Redis. Skips the CLI’s own PID.
Exit `1` if kill fails or processes remain.

There is **no** `cleanup` subcommand (removed); use `ps` / `kill` only.

### Console logging flags (`run` / `check`)

| Flag | Default | Meaning |
|------|---------|---------|
| `--console-level` | `INFO` | stderr level |
| `--log-level` | `INFO` | file level under `logs/<scan>/` |
| `--silence` / `-s` | off | no console output (files still written) |
| `--strict` | off | validation warnings → errors |

Propagated to Worker and Archiver processes. Style templates: `jarvishep2/card/logging.yaml`
(see [logger.md](logger.md)). Config validation: [config_schema.md](config_schema.md) +
[`DESIGN_YAML_VALIDATION_2.0.md`](../DESIGN_YAML_VALIDATION_2.0.md).

### Project subcommands (encrypt/decrypt usage)

| Command | Role |
|---------|------|
| `project list` / `browse` | Catalog table: **Access** + **Key** |
| `project fetch NAME [--key K]` | Download; auto-decrypt restricted packs |
| `project pack PATH --encrypt --key K` | Pack + write `*.jenc` |
| `project encrypt FILE --key K` | Encrypt existing tarball |

Full recipes: [project_tools.md](project_tools.md); `Jarvis-HEP-v2/INSTALL.md` § Project tools.

---

## 2. Module-level functions

| Function | Behavior |
|----------|----------|
| `normalize_argv(argv)` | bare YAML → `run` / legacy flag rewrite |
| `build_parser() -> ArgumentParser` | subcommands + top-level `-v` / legacy flags |
| `resolve_intent(args)` | single intent; mode conflicts → error |
| `dispatch_version()` | logo + version banner, exit 0 |
| `dispatch_monitor(args)` | one monitor snapshot |
| `dispatch_run(...)` | load YAML, logging options, `run` / `check_modules` |
| `dispatch_plot` / `dispatch_portal` / `dispatch_operas` / `dispatch_project` | bridges |
| `dispatch(args)` | version first, then intent routing |
| `main(argv=None)` | process title + parse + dispatch |

---

## 3. Lifecycle / failure semantics

- `main` is a thin synchronous front; process spawning happens in `core`.
- User errors → message + non-zero exit (`1`/`2`); no traceback dump for normal failures.
- Distinct entry `Jarvis2` coexists with V1 `Jarvis`.

### Exit codes (human CLI)

| Code | Meaning |
|------|---------|
| 0 | success (incl. version / empty `ps`) |
| 1 | run failed / partial failure / kill incomplete |
| 2 | usage / invalid args |
| 130 | interrupted |

---

## 4. Interfaces / collaborators

- **Jarvis2Core** ([core.md](core.md)) → `load_task_yaml`, `set_logging_options`, `run`, `check_modules`.
- **versioning** → `render_logo_with_version`.
- **process_cleanup** → `ps` / `kill`.
- **dashboard** ([monitor.md](monitor.md)) → monitor snapshot.
- **project_*** / Portal / Operas / plot_bridge → respective subcommands.

---

## 5. Tests

- `tests/test_cli.py` — normalize, parse, version short-circuit, dispatch routing.
- `tests/test_process_cleanup.py` — title match, kill order, confirm / `--yes`.
