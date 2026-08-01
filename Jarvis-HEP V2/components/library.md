# Component — Library / LibDeps (`jarvishep2/library.py`)

**Role**: install and reuse the shared `LibDeps` graph once in the control-process preflight,
then preserve the existing zero-copy symlink helper for concurrency-safe calculators.
**Status**: **As-built** @ `jarvis2` D18 (2026-08-01).
**Design refs**: [`../DESIGN_LIBDEPS_2.0.md`](../DESIGN_LIBDEPS_2.0.md),
[`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §8.
**Reuses V1**: none by import.

`LibraryInstaller` is intentionally independent from per-pack calculator installation:
one shared library path has one control-process writer, while calculator packs have one
stamp per Worker-owned pack.

---

## 1. Shared installation — `LibraryInstaller`

`Jarvis2Core.bootstrap_distributed_runtime()` calls `LibraryInstaller.prepare()` after
environment preflight and before command registration, Redis, Workers, or the Archiver.
It normalizes V1 plain-string commands while preserving V1's sequential standalone `cd`
working-directory rule; dependency layers come from `workflow.resolve_module_layers` and
each layer uses at most `LibDeps.make_paraller` threads.

State files use the calculator D13.11 names and schema:

```text
<LibDeps.path>/jarvis_install.json
<installation.path>/.jarvis_install_stamp.json
```

The fingerprint covers the module name, installation path/source/commands, and source-root
stat. A matching stamp is reused. `jarvis_install.json` has the monotone `reinstall_epoch`;
setting `reinstall: true` rebuilds the whole shared graph next run. V1's `installed:` flag
is accepted by schema but ignored. `--skip-library-installation` does no build, warns, and
requires each declared installation path to exist.

Build output is kept separate per module at `logs/<scan>/library-<name>.log`. Any command
failure raises `LibraryInstallError` with that path before Redis exists.

## 2. Compatibility helper — `LibraryManager`

**Attributes:** `config: dict`, `task_root: str`.

| Method | Behavior |
|--------|----------|
| `__init__(config=None, *, task_root=None)` | store config + task root (defaults to cwd). |
| `requires_shadow(clone_shadow)` (`@staticmethod`) | `bool(clone_shadow)`. |
| `link_into_sample(source_path, sample_dir, link_name) -> str` | symlink a safe tool into a Sample dir (idempotent; returns the link path); raise if source missing. |
| `resolve_registered(parser) -> dict[str,ResolvedExecutable]` | copy of a `CommandParser`'s registered map. |

---

## 3. Isolation policy

```
concurrency-safe tool   → symlink into each Sample dir          (zero-copy, link_into_sample)
non-safe tool           → clone_shadow per-pack physical copy   (CalculatorModule)
```

A `CalculatorModule` with `clone_shadow=False` and a `source` calls `link_into_sample` via
`ensure_symlink_runtime`; otherwise it physically installs per pack.

---

## 4. Interfaces / collaborators

- **CalculatorModule** ([calculator.md](calculator.md)) holds a `LibraryManager` and calls
  `link_into_sample` from `ensure_symlink_runtime`.
- **CommandParser** ([command_parser.md](command_parser.md)) resolves LibDeps / ROOT tokens
  for installation and owns the `registered` map this reads.
- **core** calls `LibraryInstaller` before it creates Redis or Worker processes.

---

## 5. Tests

`tests/test_library.py`: dependency ordering, V1 `cd` normalisation, stamp reuse, operator
reinstall, failure-with-log, skip-path checking, cycle detection, no-Redis-on-build-failure,
plus the existing symlink helper tests. Also exercised via `tests/test_clone_shadow.py`.
