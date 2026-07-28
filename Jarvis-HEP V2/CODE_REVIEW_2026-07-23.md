# Code Review — V2 runtime @ `2daf417` (2026-07-23)

**Baseline**: `jarvishep2` branch `jarvis2`, commits through `2daf417`; working tree clean.
**Focus**: the two commits that had never been reviewed — `f106b65` ("improve adaptive
sampling and runtime reliability", 4 099 lines added to `adaptive_bridson.py` alone) and
`2daf417` (`Jarvis2 convert`) — plus a doc↔code alignment pass over the whole package.
**Method**: read the diffs and the live code; verified two findings empirically rather than
by inspection (§1.1, §1.2). No test-suite run (per standing instruction — agents test
before committing).

**Verdict**: the reliability work in `f106b65` is real and valuable — the
`socket_timeout=None` fix in particular removes a whole class of spurious barrier crashes
that redis-py 5+/8 defaults would otherwise cause. But the same commit shipped a
**silent-wrong-results bug** (§1.1) whose blast radius is a physicist's own edits being
ignored, and it did so while the reference documentation still describes the *opposite*
behavior. Four findings below, plus a structural concern about a 959-line method.

---

## 1. Findings

### 1.1 Editing your calculator source no longer takes effect (severity: **high** — silent wrong results)

`f106b65` added install reuse: a pack directory keeps a
`.jarvis_install_stamp.json`, and `installation` commands are skipped when the stored
fingerprint matches. The fingerprint's only notion of "did the program change" is
[`_source_stat_payload`](../../Jarvis-HEP-v2/jarvishep2/Module/runtime_preparer.py) —
a `stat()` of the **source root directory**.

A directory's `mtime` does not change when the *content* of a file inside it changes.
Verified on this machine against the shipped function:

```
edit src/mycalc/prog.py  →  fingerprint payload changed?  False
before: {... 'mtime_ns': 1785166045732129172, 'size': 96 ...}
after : {... 'mtime_ns': 1785166045732129172, 'size': 96 ...}   ← identical
```

So the sequence a physicist actually performs — *edit the model code, re-run the scan* —
skips the install, leaves the previous binary/script in `EggBox/001…N`, and produces a
full set of results computed by **the old code**, with no warning. The run looks
completely normal.

Three aggravating factors:

1. The escape hatch (`JARVIS_FORCE_CALC_INSTALL=1`) is documented **nowhere** — not in
   YAML_REFERENCE, README, INSTALL, any design doc, or the skills library (§2).
2. `YAML_REFERENCE_2.0.md` §9.4 still tells the user the opposite is true (§2.1) — so the
   documented mental model actively prevents suspecting this.
3. The reuse decision is logged at INFO into the per-sample log — invisible in a
   10 000-point scan.

> **⚠ Correction (2026-07-23, maintainer)** — the mechanism above is accurate, but the
> conclusion this review originally drew from it was wrong. It recommended making the
> fingerprint content-aware; **automatic source-change detection is explicitly not a
> goal** (hashing an arbitrary calculator tree — multi-GB grids, build outputs, network
> mounts — is exactly the hidden cost this runtime avoids elsewhere, and any heuristic is
> still a guess). The root-dir stat is fine as designed.
>
> The real defect is narrower: **the operator has no place to control whether a
> calculator reinstalls.** The only lever is the undocumented, all-or-nothing
> `JARVIS_FORCE_CALC_INSTALL=1`, and nothing on disk says what is installed or when.
>
> **Accepted fix**: a JSON control file in the *calculator folder* (outside any PackID
> dir) that records install state and carries a `reinstall` flag the operator flips.
> Full design — including the epoch mechanism that makes one flip fan out to **all** packs
> exactly once, and the two-file/two-writer split that keeps it race-free without locking
> — in [`DESIGN_CALC_INSTALL_CONTROL_2.0.md`](DESIGN_CALC_INSTALL_CONTROL_2.0.md).
> The three aggravating factors above stand unchanged and are fixed by that design
> (documented flag, corrected §9.4, control-process run-header visibility).

### 1.2 Unmatched feedback records are silently discarded in the sampler base (severity: medium)

[`FeedbackSampler.wait_for_generation`](../../Jarvis-HEP-v2/jarvishep2/Sampling/feedback_sampler.py):

```python
record = redis.pull_feedback(timeout=wait)   # destructive BLPOP
...
if uuid not in self._pending_uuids:
    continue                                  # ← consumed and thrown away, no log
```

This is the exact pattern flagged in the 2026-07-17 review (§2.2 there) and fixed under
**D13.7b** — but the fix landed only in `RedisEvaluationPool` (nested-sampling path),
which now logs a proper warning before discarding. The **base class used by every other
feedback sampler** — AdaptiveBridson, MCMC/AM/DRAM, Ensemble/DEMCMC, PT — never got it.

The failure mode is the same and the diagnosis is worse: if a uuid ever fails to match
(resume boundary, re-queue interaction, coercion bug), the barrier waits until
`TimeoutError` while the evidence that would explain it has already been popped off the
queue and dropped. **Fix: port the D13.7b warning into the base** (~5 lines) and remove it
from the pool subclass or keep both — either way one behavior, not two.

### 1.3 Calculator slot can leak permanently on a Redis error during release (severity: medium)

[`Worker._force_release_pack`](../../Jarvis-HEP-v2/jarvishep2/worker.py) pops the pack from
`self._held_calc_packs` **before** attempting the release:

```python
with self._hb_lock():
    held_pack = self._held_calc_packs.pop(step_name, None)   # ① forget it locally
...
if not self._redis.force_release_calc(step_name, str(held_pack)):   # ② may fail
    try:
        self._redis.release_calc(step_name, str(held_pack))          # ③ may also fail
    except Exception:
        pass                                                          # ④ silently
```

`force_release_calc` returns `False` for two very different reasons — a benign
double-release *and* any Redis exception (`except Exception: return False`) — so ② and ③
both fail together on a connection blip. The pack is then **busy in Redis, absent from
`_held_calc_packs`, and unlogged**. The watchdog's safety net (`sweep_held_calc_slots`)
recovers slots by reading `held_calc_packs` from the Worker heartbeat; the next heartbeat
(fired immediately after, on the following line) overwrites the hash *without* the pack.
The slot is gone for the rest of the run. With `make_paraller: 1` that means the next
`acquire_calc` blocks until timeout and the scan stalls.

There is a second, narrower hole in the same area: `force_release_calc` does `HDEL` and
then `RPUSH` + counter updates in a **separate** pipeline. A connection loss between them
leaves the slot in neither `busy` nor `free`, with `busy` count un-decremented — the same
leak by a different route. (The atomic-`HDEL` guard added after the 2026-07-14 review
correctly solved the *double-push* race; this is the complementary *torn-release* case.)

**Recommended fix**: (a) keep the pack in `_held_calc_packs` until a release is confirmed,
so the watchdog can still see and sweep it; (b) log the failure — this path is currently
the only fully silent one in the Worker; (c) make release atomic with a small Lua script
(`HDEL` + `RPUSH` + `HINCRBY`×2 in one `EVAL`), which also lets the return value
distinguish "already free" from "error".

### 1.4 `Jarvis2 convert` can leave a silently truncated CSV that is never repaired (severity: medium)

[`convert_hdf5_to_csv`](../../Jarvis-HEP-v2/jarvishep2/database.py) writes straight to the
destination:

```python
with open(target, "w", encoding="utf-8", newline="") as handle:
    csv_writer.writeheader()
    for row in records: csv_writer.writerow(...)
```

Interrupt it (Ctrl-C — a **first-class supported path** in this runtime, with its own
checkpoint handling) and the result is a partial CSV that looks like a finished one. Then
the skip-if-exists policy makes it permanent: every later `Jarvis2 convert` reports
`skipped_exists` and the user analyses a truncated dataset until they think to pass
`--force`.

The codebase already has the right pattern one module away —
`write_install_stamp` writes `path + ".tmp"` then `os.replace(...)`. **Fix: same
tmp-then-rename here** (and in the `samples.csv` exporters in `plot_scene.py`, which write
directly for the same reason). `os.replace` is atomic on both supported platforms, so an
interrupted convert leaves the previous good CSV intact.

Minor, same function: `read_records()` materialises the whole HDF5 into a list of dicts
before writing. Fine at today's scan sizes; worth streaming if 10⁶-row scans become normal.

### 1.5 `timeout` on the adaptive loop is per-generation, not a run budget (severity: low — clarity)

`core.run_adaptive_scan(timeout=3600.0)` passes the same value into **every**
`_wait_generation` call. With `max_generations` defaulting to 16 (25 in `__init__`, 16 in
`set_config`) plus fill passes, a stuck scan can wait ~16 h before failing, and
`_wait_for_archive_caught_up` gets the full 3600 s again afterwards. Per-generation is
arguably the *right* semantic for the slow regime — a legitimate 20-hour scan must not be
killed by a run-level clock — but the name says otherwise and nothing documents it.
**Fix: rename to `generation_timeout` (keep `timeout` as an accepted alias) and state the
semantic in the docstring + YAML_REFERENCE.**

---

## 2. Doc ↔ code drift (the alignment pass)

### 2.1 YAML_REFERENCE actively describes the pre-`f106b65` behavior

§9.4 still carries this note (added during the stable-PackID review, when it was true):

> *PackIDs are stable slots (001…N) whose directories persist across samples, Workers, and
> runs. **Each NEW Worker re-runs installation on an already-populated pack directory**, so
> installation commands MUST be idempotent…*

After `f106b65` the install is *skipped* on a fingerprint match. The paragraph is now
wrong in the one direction that matters: it tells users their edits will be picked up.
**Corrected in this commit** (see §4) — but the underlying behavior still needs §1.1.

### 2.2 Undocumented surface

| Shipped in code | Documented? |
|---|---|
| Install-reuse feature as a whole (stamp file, fingerprint, reuse policy) | **no design doc at all** — a user-visible semantic change shipped undesigned |
| `JARVIS_FORCE_CALC_INSTALL=1` (the only recovery from §1.1) | **nowhere** |
| `Jarvis2 convert` / legacy `--convert` | **absent from `components/cli.md`** (only the repo README mentions it) |

`components/cli.md` otherwise matches the code exactly — all 11 subcommands present except
`convert`, so the file is one entry stale rather than systematically rotten.

### 2.3 Uncommitted design doc

`DESIGN_ADAPTIVE_BRIDSON_LIVE_BAND.md` is untracked in `Jarvis-Books` while
`adaptive_bridson.py` already implements it and `run_adaptive`'s docstring cites it by
name. Same bookkeeping failure as D13.6 in the last review, mirrored: there the code was
uncommitted and the ledger said done; here the *design* is uncommitted while the code
ships. Left untouched (another session's work in progress) — flagged for that session to
commit.

---

## 3. Structural: `run_adaptive` is a 959-line method

`adaptive_bridson.py` is now 4 068 lines, and `run_adaptive` alone is **959 lines with
nine `return total_submitted` exit points** inside one `while True`. Every band
classification, fill pass, root correction, gap bridge, and generation-advance decision is
inlined into one scope. Nothing here is *wrong* — the module's tests are extensive
(1 987 lines) and the acceptance suite passes — but this is the highest-risk file in the
package by a wide margin: the next person to change a termination condition has to hold
all nine exits in their head simultaneously.

This is worth a dedicated refactor WP *before* the next feature lands in that file, not an
emergency. The natural seams are already visible in the code's own comments: generation-0
bootstrap, the fill-pass sub-loop, root correction, gap bridging, and the
advance/converge decision each want to be a method returning an explicit
`LoopDecision` (continue / advance / stop-with-reason). That also makes the exit reasons
testable individually, which nine inline `return`s are not.

---

## 4. Action items

Registered as **D13.11** (bug fixes) and **D13.12** (refactor) in
[`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md):

| # | Item | Severity |
|---|---|---|
| 1 | Operator-facing install control file (`jarvis_install.json` + epoch) per [`DESIGN_CALC_INSTALL_CONTROL_2.0.md`](DESIGN_CALC_INSTALL_CONTROL_2.0.md) + run-header visibility + document the flag | **high** |
| 2 | Port the D13.7b unmatched-feedback warning into `FeedbackSampler.wait_for_generation` | medium |
| 3 | Slot-release: keep in `_held_calc_packs` until confirmed, log failures, make release atomic (Lua) | medium |
| 4 | Atomic tmp+rename in `convert_hdf5_to_csv` and the `samples.csv` exporters | medium |
| 5 | `generation_timeout` naming + documented semantic | low |
| 6 | Design doc + `cli.md` entry for install reuse and `convert` | doc |
| 7 | Decompose `run_adaptive` behind explicit loop decisions (D13.12) | structural |

**Fixed directly in this commit** (documentation only, no code): the false idempotency
note in YAML_REFERENCE §9.4 — it was actively misleading and is the reason §1.1 would go
unsuspected.
