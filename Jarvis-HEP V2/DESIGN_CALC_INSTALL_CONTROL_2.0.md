# DESIGN — Calculator Install Reuse & User Control (V2, D13.11)

**Status**: design accepted 2026-07-23 (maintainer); implementation `todo`
**Date**: 2026-07-23
**Scope**: give the operator an explicit, inspectable place to say *"reinstall this
calculator"*, and write down the install-reuse behavior that shipped in `f106b65` without
a design doc.
**Supersedes**: the recommendation in
[`CODE_REVIEW_2026-07-23.md`](archive/reviews/CODE_REVIEW_2026-07-23.md) §1.1 to make the fingerprint
content-aware — see §1.

---

## 1. Problem statement (corrected)

`f106b65` added install reuse: each pack keeps `.jarvis_install_stamp.json` and
`installation` is skipped when a fingerprint (module name, basepath, source path + **root
dir** stat, command list) still matches.

The 2026-07-23 review framed the root-dir stat as the bug, because editing a file *inside*
the source tree leaves the directory mtime unchanged, so edits are not detected. **That
framing was wrong about the intent**: automatic source-change detection is explicitly
*not* a goal — a content hash of an arbitrary calculator tree (which may hold multi-GB
grids, build outputs, or network mounts) is exactly the kind of hidden cost this runtime
avoids elsewhere, and any auto-detection heuristic would still be a guess.

The real defect is narrower and clearer:

> **The operator has no place to control whether a calculator reinstalls.**

Today the only lever is the undocumented environment variable
`JARVIS_FORCE_CALC_INSTALL=1`, which is all-or-nothing (every module, every pack) and
leaves no trace of what happened or when. Nothing on disk tells the user which packs were
installed, from what source, or how long ago.

## 2. Goals

1. **One JSON file per calculator, in the calculator folder, outside any PackID dir**, that
   the operator can open, read, and edit to trigger a reinstall.
2. **Self-explaining**: the file records what is installed (per pack: when, from where,
   fingerprint) so "is my scan running the code I think it is?" is answerable by `cat`.
3. **One-shot, fan-out-correct**: flipping the flag once must reinstall **every** pack of
   that calculator exactly once — never "the first Worker consumed the flag and the other
   packs kept stale code".
4. **No new required YAML.** Existing cards keep working untouched (invariant 1); the
   control file is auto-created on first install.
5. **Race-free under N Workers** without file locking.

### Non-goals

- Content hashing / auto-detecting source edits (§1).
- Per-sample install control (`initialization` already runs per sample — V1 contract).
- Cross-module "reinstall everything" UI beyond the existing env var.

## 3. Design

### 3.1 Two files, two writers, no contention

```
<calculator folder>/                     ← parent of the @PackID dirs
├── jarvis_install.json                  ← ① CONTROL + SUMMARY (control process writes)
├── 001/
│   ├── .jarvis_install_stamp.json       ← ② per-pack record (installing Worker writes)
│   └── <program files…>
├── 002/
│   └── …
```

The calculator folder is always computable: it is `Calculators.Modules[].path` with the
`@PackID` component stripped (`os.path.dirname` of the decoded pack path). Non-shadow
modules have no packs and are out of scope for reuse.

**① `jarvis_install.json` — written only by the control process**, once per scan at
startup, before Workers spawn. Single writer ⇒ no locking, no races.

**② `.jarvis_install_stamp.json` — written only by the Worker** that performs an install,
inside the pack it just installed (as today, plus an `epoch` field).

### 3.2 The epoch: how one flip reaches every pack

A boolean that the first installer resets would break goal 3. Instead the control file
carries a monotone counter:

```jsonc
{
  "schema": "jarvishep2.calc_install/v2",
  "module": "EggBox",

  // ---- the operator's control ----
  "reinstall": false,        // set true → next run reinstalls every pack of this module
  "reinstall_epoch": 3,      // maintained by Jarvis; do not edit

  // ---- what is currently installed (informational) ----
  "source": "/…/src/eggbox",
  "packs": {
    "001": {"epoch": 3, "installed_at_utc": "2026-07-23T04:11:07Z", "fingerprint": "9f2c…"},
    "002": {"epoch": 3, "installed_at_utc": "2026-07-23T04:11:09Z", "fingerprint": "9f2c…"}
  },
  "_help": "Edited your calculator source? Set \"reinstall\": true and run again."
}
```

**Control process, at scan start**, per calculator module:

1. Read `jarvis_install.json` (missing ⇒ treat as `{reinstall: false, reinstall_epoch: 0}`).
2. If `reinstall` is true **or** `JARVIS_FORCE_CALC_INSTALL=1`:
   `reinstall_epoch += 1`, `reinstall = false`, write back atomically (tmp + `os.replace`).
3. Pass `reinstall_epoch` into the worker config for that module.

**Worker, before installing a pack** — reinstall when **either** holds:

- the pack stamp's `fingerprint` ≠ the current fingerprint (existing behavior: config
  changed), **or**
- the pack stamp's `epoch` < the module's `reinstall_epoch` (new: operator asked).

After a successful install the Worker stamps the pack with the current epoch and
fingerprint. Because the epoch only ever rises and each pack compares against it
independently, one flip reinstalls all N packs exactly once, in whatever order Workers
happen to reach them.

The `packs` summary in ① is refreshed by the control process at the end of the scan (best
effort, informational only — ② stays authoritative for the decision).

### 3.3 Visibility

The reuse/install decision moves from a per-sample INFO line (invisible in a 10 000-point
scan) to the **control-process run header**, one line per module:

```
EggBox: reusing install from 2026-07-19T21:04Z (epoch 3, 8 packs)   ← or
EggBox: reinstalling 8 packs (operator requested, epoch 3 → 4)
```

That single line is what makes "am I running the code I just edited?" answerable without
reading any file.

## 4. YAML surface

None required. Optional, additive (invariant 1), for cards that want the behavior pinned:

```yaml
Calculators:
  Modules:
    - name: EggBox
      reuse_install: true     # default true; false = always reinstall this module
```

## 5. Work packages

Folded into **D13.11** (see [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md)):

| Step | Accept |
|---|---|
| `jarvis_install.json` read/bump/write in the control process (atomic tmp+replace) | missing file ⇒ epoch 0, no forced install; `reinstall: true` ⇒ epoch bumps once and the flag is cleared |
| Epoch plumbed into worker config; Worker install gate = fingerprint **or** epoch | flipping `reinstall` once reinstalls **all** packs exactly once (test with 4 packs / 4 Workers) |
| Per-module run-header line | reuse and reinstall are both visible in `core.log` without `--debug` |
| Docs: YAML_REFERENCE §9.4 rewrite, `JARVIS_FORCE_CALC_INSTALL` documented, `skills/external-calculator.md` "改了程序怎么办" updated to the JSON flag | a user who edits their source finds the answer without reading source code |

## 6. Risks

1. **Stale summary**: `packs` in ① is informational and can drift if a run is killed
   mid-scan. The decision path never reads it — ② is authoritative. Documented in `_help`.
2. **Shared calculator folder across concurrent scans**: two scans of the same project
   share pack dirs today (that is pre-existing). The epoch bump is idempotent per scan
   start; the worst case is one extra reinstall, never a skipped one.
3. **Operator edits `reinstall_epoch` by hand**: lowering it could skip a needed install.
   The `_help` line says not to; the fingerprint check still catches config changes.
