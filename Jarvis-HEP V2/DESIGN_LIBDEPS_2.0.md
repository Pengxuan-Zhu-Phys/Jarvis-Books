# DESIGN — LibDeps: install-once shared dependencies (V2, D18)

**Status**: design accepted 2026-07-31 (maintainer: *"对齐 V1 的 LibDeps 功能"*); implementation `todo`
**Date**: 2026-07-31
**Scope**: bring V2's `LibDeps` block up to V1 parity — build/install shared libraries
**once**, reuse them across every calculator and every scan, with the operator in control
of when they rebuild.
**Reuses**: the install-control machinery already built for calculators in D13.11
([`DESIGN_CALC_INSTALL_CONTROL_2.0.md`](DESIGN_CALC_INSTALL_CONTROL_2.0.md)).

---

## 1. What exists today (measured, `6e308a4`)

V2 has the **reference half** of LibDeps and none of the **install half**.

| capability | V1 | V2 |
|---|---|---|
| build/install a library from `installation.commands` | `Module/library.py` (159 lines): `install()`, `run_install_commands()`, async subprocess | **none** — no code anywhere executes LibDeps commands |
| "already installed?" detection | `check_installed()` — writes `<name>_config.yaml`, compares it against the current config on the next run | none |
| skip switch | `--skip-library-installation` | none (nothing to skip) |
| per-library install log | `Library_<name>_installation.log` | none |
| dependency ordering | `required_modules` | not consumed for LibDeps |
| `${LibDeps:<Module>}` path token | ✅ | ✅ |
| `${LibDeps:path}` (base dir) | ✅ | ❌ **KeyError** — used **36×** in shipped cards |
| `${LibDeps:make_paraller}` (`make -j…`) | ✅ | ❌ **KeyError** |
| `@{ROOT path}` (CERN ROOT) | ✅ | ❌ no token (but the value *is* already computed — `environment_requirements.py:142` puts it in `report.summary["ROOT path"]`) |

`jarvishep2/library.py` is 41 lines and does only two things: symlink a tool into a Sample
directory (D2.3 clone_shadow) and resolve `registered_executables`. `_build_libdeps_paths`
(`command_parser.py:319`) *reads* `installation.{path,source}` — but only to compute a
string for token substitution. Nothing is ever built.

**Practical consequence**: the user must install every shared dependency by hand before
running, and two of the three tokens their V1 card uses raise `KeyError` at command
resolution. Seven shipped example cards contain a `LibDeps` block.

## 2. What LibDeps *is*, and why it is not a calculator

Both build things once and reuse them, but the shapes differ, and the difference drives
the design:

| | Calculator pack | LibDeps module |
|---|---|---|
| how many copies | **N** (`001`…`N`), one per concurrency slot | **one**, shared by everything |
| who installs | the Worker that first acquires the pack | the **control process**, before Workers spawn |
| when | during the scan | **before** the scan starts |
| why | per-sample isolation (`clone_shadow`) | expensive shared builds (ROOT, Delphes, HepMC) |

So LibDeps is *simpler* than the calculator case — no PackID, no fan-out, no concurrent
writers — which means the D13.11 machinery can be reused with its hardest part removed.

## 3. Design

### 3.1 Install state: reuse D13.11, drop the epoch fan-out

```
<LibDeps.path>/
├── jarvis_install.json          ← control + summary (control process only)
├── Delphes/
│   └── .jarvis_install_stamp.json
└── HepMC/
    └── .jarvis_install_stamp.json
```

Identical file names and schema to the calculator case, so operators learn one mechanism.
The **`reinstall_epoch` fan-out is unnecessary here** (one copy, one writer), but keep the
field for format symmetry; setting `"reinstall": true` rebuilds every module of that
block on the next run.

**Fingerprint** = module name + `installation.{path, source, commands}` + source-root
stat. Same rule and the same deliberate limitation as calculators: editing a file *inside*
a source tree does not trigger a rebuild — the operator asks for it via the control file.
This is exactly what V1's `check_installed()` did by comparing a written config file, so
the semantics carry over.

**V1's interactive prompt is deliberately not carried over.** V1 asks
`Reinstall? (y/n)` on stdin when the config is unchanged. V2's control process must stay
non-interactive (it may run under nohup, a scheduler, or a future cluster submitter), and
the `jarvis_install.json` flag already expresses the same intent ahead of time.

### 3.2 Where it runs

In the control process, inside the existing preflight phase — **after** environment
requirements (a library build may need CERN ROOT) and **before** Redis/Worker startup, so
a failed build stops the scan before anything is spawned (same contract as D17: fail
before side effects).

Order across modules follows `required_modules` via the existing
`workflow.resolve_module_layers`; modules within a layer may build concurrently, capped by
`LibDeps.make_paraller`.

### 3.3 Command shape and tokens

V1 cards write `installation.commands` as a **list of plain strings**, not
`{cmd, cwd}` mappings — the same V1 shape that D12.1 had to normalize for calculators.
Reuse that normalization (string → `{cmd: <str>, cwd: <installation.path>}`), so the
shipped cards work unmodified.

Tokens that must resolve inside LibDeps commands:

| token | source | today |
|---|---|---|
| `${path}` / `${source}` | the module's own `installation.{path,source}` | reuse `_expand_install_tokens` |
| `${LibDeps:<Module>}` | that module's resolved path | ✅ works |
| `${LibDeps:path}` | `LibDeps.path` (base dir) | **add** — 36 uses |
| `${LibDeps:make_paraller}` | `LibDeps.make_paraller` | **add** — build parallelism |
| `@{ROOT path}` | CERN ROOT prefix | **add** — wire from `environment_requirements`' existing `summary["ROOT path"]`; if ROOT is not configured, fail with a message naming `EnvReqs.CERN_ROOT` |

Generalize `_replace_libdeps_token` from "module names only" to "module names **plus** the
block's own scalar keys", which covers `path` and `make_paraller` in one rule.

### 3.4 CLI and logging

- **`--skip-library-installation`** on `run`/`check` (V1 parity): skip the build phase and
  use whatever is on disk. Warn clearly; if a required path is missing, fail with the
  reason rather than proceeding into a broken scan.
- Per-module log `logs/<scan>/library-<name>.log` (V1 wrote
  `Library_<name>_installation.log`) plus one control-log line per module —
  `Delphes: reusing install from 2026-07-19T21:04Z` / `Delphes: building (operator
  requested)` — same visibility rule as D13.11.

### 3.5 Schema

Declare `LibDeps` in the task-card schema (`path`, `make_paraller`, `Modules[]` with
`name` / `required_modules` / `installed` / `installation.{path,source,commands}`, and
`registered_executables`). Zone: **closed**.

This is also the **precondition for D17.9**: the root schema is currently marked
`delegated` (so a typo'd top-level block is silently dropped) largely because `LibDeps`
was never declared. Declaring it here lets D17.9 close the root.

V1's `installed: false` field appears in real cards; accept it, ignore it, and let the
stamp file be authoritative — the flag was V1's bookkeeping, not user intent.

## 4. Work packages

| WP | Title | Accept |
|---|---|---|
| **D18.1** | LibDeps token completeness (`${LibDeps:path}`, `${LibDeps:make_paraller}`, `@{ROOT path}`) | the 36 `${LibDeps:path}` uses in shipped cards resolve; ROOT token resolves from preflight, or fails naming `EnvReqs.CERN_ROOT` |
| **D18.2** | Declare `LibDeps` in the schema (zone `closed`) | shipped LibDeps cards validate; unblocks **D17.9** |
| **D18.3** | Install engine: string-command normalization, `required_modules` ordering, `make_paraller` concurrency, per-module log, control-process preflight placement | a card whose library is absent builds it once; a second run reuses it; a failed build stops the scan before Redis/Workers start |
| **D18.4** | `jarvis_install.json` control + stamps for LibDeps (reuse D13.11) + `--skip-library-installation` | `"reinstall": true` rebuilds; unchanged config reuses without prompting; skip flag honoured and logged |
| **D18.5** | Docs + skill | YAML_REFERENCE LibDeps section; new skill `shared-libraries.md`, card verified (D16 rule) |

**Rollback**: a card with no `LibDeps` block is unaffected; without D18.3 the block keeps
resolving paths exactly as today.

## 5. Risks

1. **Long builds in the preflight.** A ROOT/Delphes build can take tens of minutes with no
   samples produced yet. Mitigation: the per-module control-log line plus the reuse path
   means it happens once; document that first-run cost in the skill.
2. **Build failures are not physics failures.** They must exit with a clear
   "library build failed" message and the module's log path — never fall through into a
   scan that will fail one sample at a time.
3. **Shared state across concurrent scans.** Two scans of the same project share
   `LibDeps.path`. The control process is the only writer and the operation is
   idempotent-by-fingerprint, so the worst case is one redundant rebuild; note it rather
   than adding locking.
