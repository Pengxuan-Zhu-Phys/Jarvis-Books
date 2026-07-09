# DESIGN — Jarvis-PLOT CLI Bridge (V2)

**Status**: approved for implementation  
**Date**: 2026-07-10  
**Scope**: thin HEP → Jarvis-PLOT handoff so plotting features evolve by upgrading **JarvisPLOT**, not HEP.

---

## 1. Problem

V2 has no `--plot` path. V1 embeds a large `jarvishep/plot.py` that *emits* JarvisPLOT YAML from a scan, then runs the plotter. Re-porting emit logic is a large WP; the immediate gap is **delegation**: users should run plot YAMLs through the V2 CLI without shelling out to a separate mental model.

## 2. Goals

1. **HEP does not own plot algorithms** — all figure/layer code stays in Jarvis-PLOT.
2. **`Jarvis2 --plot <yaml>`** invokes the installed JarvisPLOT engine.
3. **Upgrade JarvisPLOT ⇒ new figure types/methods** without HEP code changes.
4. Optional dependency (`jarvishep2[plot]`).
5. Clear errors when plot extra is missing.

### Non-goals (this WP)

- Full V1 `emit_jplot` (generate plot YAML from Sampling block).
- Flowchart rendering during scan (`--skip-draw-flowchart` parity).
- HDF5→CSV convert.

Those can land later as pure HEP helpers that still *call* PLOT, not reimplement it.

---

## 3. Design

```
Jarvis2 --plot scene.yaml [jplot args…]
        │
        ▼
jarvishep2.plot_bridge.run_plot(yaml, argv_tail)
        │
        ▼
jarvisplot.core.JarvisPLOT().init()   # sys.argv = [jplot, yaml, …]
```

| Module | Role |
|--------|------|
| `jarvishep2/plot_bridge.py` | Import guard, argv injection, return code |
| `jarvishep2/client.py` | `--plot` flag + dispatch before run path |

### API

```python
def run_plot(plot_yaml: str, *, extra_args: list[str] | None = None) -> int:
    """Run JarvisPLOT on plot_yaml. Returns process-style exit code."""

def require_jarvisplot():
    """Import jarvisplot or raise ImportError with install hint."""
```

### CLI

```
Jarvis2 --plot <plot.yaml> [passthrough…]
```

When `--plot` is set:

- `task_yaml` positional (or first free arg) is the **plot YAML**, not a scan task.
- Remaining unknown args are **not** supported in argparse strict mode — keep MVP to `--plot` + one yaml path only. Passthrough via `nargs=argparse.REMAINDER` after `--` if needed.

MVP:

```
Jarvis2 --plot path/to/plot.yaml
```

Exit codes: 0 success, 1 plot failure, 2 usage / missing package.

---

## 4. Failure semantics

| Case | Behavior |
|------|----------|
| JarvisPLOT not installed | exit 2 + `pip install 'jarvishep2[plot]'` |
| Missing yaml path | exit 2 |
| Plot engine raises / sys.exit | map to non-zero return |

---

## 5. Acceptance

1. Design doc + component note (utils/cli).
2. `plot_bridge` unit test with a fake `JarvisPLOT` (no heavy matplotlib run required).
3. CLI `--plot` without package → clear message (or with mock).
4. Optional smoke if jarvisplot installed.
5. Commit.

---

## 6. Follow-ups

- `emit_jplot_from_scan(task_yaml, out_dir)` using DATABASE paths.
- `Jarvis2 <task.yaml> --plot` after successful run.
- Flowchart via `jarvisplot.flowchart` during workflow init.
