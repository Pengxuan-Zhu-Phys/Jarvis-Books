# Component — Library / LibDeps (`jarvishep2/library.py`)

**Role**: minimal LibDeps helper — symlink concurrency-safe tools into Sample dirs (zero-copy) and
expose the Phase-1 `registered_executables` map.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `library.py` 41 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §8.
**Reuses V1**: none by import.

> **As-built drift:** the design described a fuller `LibraryManager` (`install_all`, `dep_path`,
> `InstalledDep`, a `LibraryModule` workflow type). **Shipped is a thin helper** — symlinking +
> reading the registered map. Boot-time LibDeps install is handled inside
> [`CommandParser.register`](command_parser.md) (which creates the symlinks / validates sources);
> per-pack physical installs are done by [`CalculatorModule.ensure_shadow_installed`](calculator.md).

---

## 1. Class defined — `LibraryManager`

**Attributes:** `config: dict`, `task_root: str`.

| Method | Behavior |
|--------|----------|
| `__init__(config=None, *, task_root=None)` | store config + task root (defaults to cwd). |
| `requires_shadow(clone_shadow)` (`@staticmethod`) | `bool(clone_shadow)`. |
| `link_into_sample(source_path, sample_dir, link_name) -> str` | symlink a safe tool into a Sample dir (idempotent; returns the link path); raise if source missing. |
| `resolve_registered(parser) -> dict[str,ResolvedExecutable]` | copy of a `CommandParser`'s registered map. |

---

## 2. Isolation policy

```
concurrency-safe tool   → symlink into each Sample dir          (zero-copy, link_into_sample)
non-safe tool           → clone_shadow per-pack physical copy   (CalculatorModule)
```

A `CalculatorModule` with `clone_shadow=False` and a `source` calls `link_into_sample` via
`ensure_symlink_runtime`; otherwise it physically installs per pack.

---

## 3. Interfaces / collaborators

- **CalculatorModule** ([calculator.md](calculator.md)) holds a `LibraryManager` and calls
  `link_into_sample` from `ensure_symlink_runtime`.
- **CommandParser** ([command_parser.md](command_parser.md)) owns the `registered` map this reads.

---

## 4. Tests

`tests/test_library.py` (5): zero-copy symlink into a Sample dir, idempotent re-link, registered
map resolution, missing-source error. Also exercised via `tests/test_clone_shadow.py` (5).
