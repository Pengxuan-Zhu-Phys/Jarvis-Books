# Component — Library / LibDeps (`jarvishep2/library.py`)

**Role**: install and resolve shared library dependencies (`LibDeps`) and back the
`registered_executables` mechanism. Distinguishes concurrency-safe tools (symlink/direct-path)
from non-safe tools (`clone_shadow`).
**Status**: design — plan WP-D2.3 (isolation) + D3.1 (registered_executables).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §8;
discussion `Jarvis-HEP_Discussion_Summary_2026-06-21.md` §4/§5.
**Reuses V1**: `jarvishep/library.py` install logic **and** `jarvishep/Module/library.py`
(`LibraryModule` — the in-workflow library-install module: `check_installed`, `install`,
`run_install_commands`). Pairs with [CommandParser](command_parser.md) (which resolves executable
paths) and [CalculatorModule](calculator.md) (which uses them).

> **Two roles, one doc.** (1) The top-level **`LibraryManager`** installs `LibDeps` once at boot
> and backs `registered_executables`. (2) **`LibraryModule`** (`Module/library.py`) is a workflow
> module type that installs a library as part of a Sample's `required_modules` chain. Both are
> control-side/boot install paths in V2 (Workers only *use* installed deps); the manager owns the
> isolation policy below, the module is the per-required-module installer.

---

## 1. Responsibilities

1. **Install `LibDeps`** once per run (build/fetch into the deps area; idempotent).
2. Back `registered_executables`: produce a name → path map (`direct_path` default, `symlink`
   optional) consumed by the CommandParser.
3. Encode the **isolation policy**: concurrency-safe tools are shared (symlink/direct-path into
   the Sample dir); non-safe tools are flagged `clone_shadow` so the Worker copies them
   per-Sample.

---

## 2. Structure

```python
class LibraryManager:
    def __init__(self, config, project_root): ...
    deps: dict[str, InstalledDep]
    registered: dict[str, ResolvedExecutable]   # shared with CommandParser

@dataclass
class InstalledDep:
    name: str
    path: str
    concurrency_safe: bool        # True → symlink/direct; False → clone_shadow required
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `install_all` | `() -> None` | Install every `LibDeps` entry once (idempotent; skip if present). |
| `resolve_registered` | `() -> dict[str, ResolvedExecutable]` | Build the `registered_executables` map (`direct_path`/`symlink`). |
| `link_into_sample` | `(name, sample_dir) -> str` | For a safe tool, create a symlink (or return the direct path) inside a Sample dir — zero-copy. |
| `requires_shadow` | `(name) -> bool` | True for non-concurrency-safe tools (`clone_shadow`). |
| `dep_path` | `(name) -> str` | Resolved absolute path of an installed dep. |

---

## 4. Isolation policy (the core decision)

```
concurrency-safe tool   → LibDeps + symlink/direct_path into each Sample dir   (no copy)
non-safe tool (MadGraph)→ clone_shadow: per-Sample physical copy                (unavoidable)
```

This is the explicit, YAML-declared split from the discussion (§4): pay the copy cost **only**
where a tool genuinely can't be shared across concurrent processes.

---

## 5. Concurrency / lifecycle / failure semantics

- **Install once** at boot (control process), before Workers start; Workers only *use* the
  installed deps (no re-install on the hot path).
- `direct_path` registered executables have **zero cleanup**; `symlink` links are created at
  install/use and cleaned with the Sample dir.
- `clone_shadow` copies are owned by the Worker per Sample and removed via the
  staging/Archiver cleanup path (`delete_method`).
- A missing/failed dep install raises at boot with a clear message (no silent half-install).

---

## 6. Interfaces

- **core.init_libraries** → `install_all` + `resolve_registered` at boot.
- **CommandParser** → consumes `registered` for Phase-1 resolution.
- **Worker / CalculatorModule** → `link_into_sample` (safe) or `requires_shadow` →
  `decode_shadow_*` (non-safe).

---

## 7. Tests (`tests/test_library.py`)

Unit:
1. **Idempotent install** — `install_all` twice installs once (second is a no-op; assert via a
   sentinel).
2. **Registered resolution** — `direct_path` → absolute path; `symlink` → a link that resolves
   to the source.
3. **Safe vs shadow** — a safe tool links into a Sample dir with no copy; a `clone_shadow` tool
   reports `requires_shadow True`.
4. **Concurrent link** — N Samples linking the same safe tool don't race (each gets a valid
   link; no partial state).
5. **Failure** — a missing source raises a clear boot error.

Verification logic: test 3 encodes the isolation policy that bounds the copy cost (the
slow-regime efficiency lever); test 4 ensures shared tools are safe under Worker concurrency.

---

## 8. Open questions

- Optional `isolation: full_shadow | per_sample_dir` field (design §4 — not added yet).
- Whether registered executables are validated for existence at boot vs. first use (default:
  boot, fail fast).
