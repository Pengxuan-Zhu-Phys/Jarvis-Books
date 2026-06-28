# Component — CLI parsing & dispatch (命令行解析) (`jarvishep2/client.py`, `jarvishep2/card/argparser.json`)

**Role**: the `Jarvis2` command-line surface. Parse args/subcommands, dispatch to the right run
mode, and stay a **separate entry point** from V1's `Jarvis` (no collision).
**Status**: design — auxiliary; D1.1 (boot) onward.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §0.1 (Jarvis2),
plan invariant #2; [core.md](core.md) (what each mode does).
**Reuses V1**: the data-driven argparser pattern (`card/argparser.json` → argparse) and
`client.py` dispatch; `versioning.py` banner.

---

## 1. Responsibilities

1. Define the `Jarvis2` CLI from a **declarative card** (`card/argparser.json`) — same pattern
   as V1, so help text/flags are data, not code.
2. Dispatch a parsed namespace to the matching `core` entry (run / resume / monitor /
   check-modules / benchmark / convert / project …).
3. Print the version/banner (`Jarvis2 -v`, `--refs`).
4. Be installed as a **distinct console-script** (`Jarvis2`) so V1 and V2 coexist.

---

## 2. Structure

```python
def main(argv=None) -> int:
    args = build_parser().parse_args(argv)
    return dispatch(args)

def build_parser() -> argparse.ArgumentParser:        # built from card/argparser.json
    ...

def dispatch(args) -> int:
    # route to core.* by mode/subcommand
    ...
```

`card/argparser.json` (declarative): top-level options + subparsers, each entry =
`{name, flags, help, default, action, nargs}`.

---

## 3. CLI surface (mirrors V1, additive)

| Command | Routes to | Notes |
|---------|-----------|-------|
| `Jarvis2 <file.yaml>` | `core.run` | distributed run (Redis + Workers + Archiver) |
| `Jarvis2 <file.yaml> --resume` | `core.run` (resume) | drain queue, restore sampler state |
| `Jarvis2 <file.yaml> --monitor [--pid N]` | `core.monitor` | attach dashboard (in/out of process) |
| `Jarvis2 <file.yaml> --check-modules` | `core.check_modules` | 10-sample smoke via the real path |
| `Jarvis2 <file.yaml> --benchmark [s]` | `core.benchmark` | throughput + `parallelism_efficiency` |
| `Jarvis2 <file.yaml> --convert` | `core.convert` | HDF5 → CSV (frozen format) |
| `Jarvis2 <file.yaml> --plot` | `core.plot` | plot config |
| `Jarvis2 worker start` | `worker_cli.start` | **new** — launch a standalone Worker (debug/deploy) |
| `Jarvis2 project create/pack/browse/fetch/info` | `project_tools` | scaffolding/packaging (see [project_tools.md](project_tools.md)) |
| `Jarvis2 -v` / `--refs` | `versioning` | version / full reference |

The **only** additions over V1's surface are `Jarvis2 worker start` and `--monitor --pid`
(plan invariant #2); everything else mirrors `Jarvis` under the new entry point.

---

## 4. Member functions

| Function | Signature | Behavior |
|----------|-----------|----------|
| `main` | `(argv=None) -> int` | Entry point; build parser, parse, dispatch; return an exit code. |
| `build_parser` | `() -> ArgumentParser` | Construct from `card/argparser.json` (subparsers + options). |
| `dispatch` | `(args) -> int` | Map mode/subcommand → `core.*`; handle top-level errors (clear messages, non-zero exit). |
| `print_banner` | `() -> None` | `Jarvis2` logo + version (`versioning.py`). |
| `worker_start` | `(args) -> int` | Boot one standalone `Worker` attached to an existing Redis (deploy/debug). |

---

## 5. Concurrency / lifecycle / failure semantics

- `main` is a thin, synchronous front; all process spawning happens in `core`.
- Argument errors → argparse's own usage message; semantic errors (missing file, bad mode)
  → a clear message + non-zero exit (no traceback dump for user errors).
- `Jarvis2 worker start` lets a Worker join a running scan's Redis from another shell/host —
  useful for scaling out and for debugging a single Worker in isolation.
- Installed via `[project.scripts] Jarvis2 = "jarvishep2.client:main"`; a distinct distribution
  name (`Jarvis-HEP2`) lets `Jarvis` (V1) and `Jarvis2` (V2) install side by side.

---

## 6. Interfaces

- **core** → every run mode.
- **project_tools** → `project` subcommands.
- **dashboard/monitor** → `--monitor`.
- **versioning** → banner/`-v`/`--refs`.

---

## 7. Tests (`tests/test_cli.py`)

Unit:
1. **Parse matrix** — each documented command/flag parses to the expected namespace
   (table-driven over `card/argparser.json`).
2. **Dispatch routing** — each namespace calls the right `core.*` (mock core; assert the call).
3. **Independence** — `Jarvis2` entry point exists and does **not** shadow or import V1's
   `Jarvis` dispatch.
4. **User-error UX** — missing file / bad mode → clear message + non-zero exit, no traceback.
5. **worker start** — `Jarvis2 worker start` boots a Worker against a (fake) Redis and joins the
   task queue.
6. **Version/refs** — `-v` / `--refs` print and exit 0.

Verification logic: tests 1/2 lock the surface (additive-only, invariant #2); test 3 enforces
the V1/V2 non-interference decision.

---

## 8. Open questions

- Final distribution/package name for side-by-side install (design §0.1; D0.2/D1.1).
- Whether `Jarvis2 worker start` needs an auth/namespace token to join a Redis (multi-tenant
  clusters) — defer; single-user default first.
