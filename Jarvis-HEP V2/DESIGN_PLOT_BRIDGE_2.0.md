# DESIGN — Jarvis-PLOT CLI Bridge (V2)

**Status**: **implemented — render-only bridge**; scan-driven plotting remains D11.5
**Date**: 2026-07-10 · as-built review 2026-07-13
**Scope**: thin HEP → Jarvis-PLOT handoff so plotting features evolve by upgrading **JarvisPLOT**, not HEP.

---

## 1. Problem

At the time of design V2 had no `--plot` path. The thin delegation path is now implemented.
V1 also embeds `jarvishep/plot.py`, which *emits* JarvisPLOT YAML from a scan, and renders a
workflow flowchart. Those scan-driven semantics are not implemented in V2.

## 2. Goals

1. **HEP does not own plot algorithms** — all figure/layer code stays in Jarvis-PLOT.
2. **`Jarvis2 <yaml> --plot`** invokes the installed JarvisPLOT engine.
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

> **As-built UX warning:** the positional YAML is a JarvisPLOT scene, not a Jarvis-HEP scan
> task. This differs from V1's scan-driven `Jarvis ... --plot` intent. D11.5 must remove the
> ambiguity with an explicit `Jarvis2 plot PLOT_YAML` command and a separate from-run path.

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

As-built verification on 2026-07-13 used installed `JarvisPLOT 1.4.2` and produced a real PNG
with exit 0. Unit coverage lives in `tests/test_plot_bridge.py` (8 tests).

---

## 6. Follow-ups

- `emit_jplot_from_scan(task_yaml, out_dir)` using DATABASE paths.
- `Jarvis2 <task.yaml> --plot` after successful run.
- Flowchart via `jarvisplot.flowchart` during workflow init.
- Standard overlay scene for D10 `levelset.json`.
