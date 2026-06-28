# Component — Paths & Runtime Tokens (`jarvishep2/base.py`)

**Role**: the path and token foundation. Resolves the `&J/` project-root marker and the runtime
tokens `@SampleID` / `@Sdir` / `@PackID`, and owns the per-scan output directory layout.
**Status**: design — auxiliary; D0.1 onward (every component inherits from `Base`).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §3, §8; CLAUDE.md
"Key Conventions"; [command_parser.md](command_parser.md) (command-level resolution),
[sample.md](sample.md) (`@Sdir` triggers materialize).
**Reuses V1**: `Base` (`jarvishep/base.py`) path management + token resolution.

---

## 1. Responsibilities

1. Resolve `&J/` → the standalone project root (identified by `.jarvis-project.json` /
   `jarvis.project.yaml`).
2. Own the output layout: `outputs/<scan>/DATABASE/`, `outputs/<scan>/SAMPLE/<bucket>/<uuid>/`,
   `outputs/<scan>/staging/`, `checkpoints/<scan>/<sampler>/state.pkl`.
3. Resolve the **per-Sample tokens** `@SampleID` (uuid), `@Sdir` (save dir), `@PackID`
   (calculator instance / slot) — the Phase-2 surface (Worker-side).
4. Be the shared base class with these helpers, inherited by Sampler/Workflow/Modules/etc.

Token split (with [CommandParser](command_parser.md)):
- **Static** (`&J/`, `${LibDeps:…}`, `${Scan:…}`, registered executables) → Phase 1, control
  process.
- **Per-Sample** (`@SampleID`, `@Sdir`, `@PackID`) → Phase 2, Worker, at execution.

---

## 2. Structure

```python
class Base:
    project_root: str
    scan_name: str
    task_root: str
    def resolve_project_root(self, start: str) -> str: ...       # find &J/ anchor
    def expand_J(self, text: str) -> str: ...                    # &J/ → abs path
    def output_dir(self, kind: str) -> str: ...                  # DATABASE|SAMPLE|staging
    def sample_dir(self, bucket: str, uuid: str) -> str: ...
    def resolve_runtime_tokens(self, text: str, *, sample_info: dict) -> str: ...
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `resolve_project_root` | `(start) -> str` | Walk up to the `.jarvis-project.json`/`jarvis.project.yaml` anchor; cache. |
| `expand_J` | `(text) -> str` | Replace `&J/` with the absolute project root. |
| `output_dir` | `(kind) -> str` | `outputs/<scan>/{DATABASE|SAMPLE|staging}` (DATABASE always under `outputs/<scan>/DATABASE/`, frozen). |
| `sample_dir` | `(bucket, uuid) -> str` | `outputs/<scan>/SAMPLE/<bucket>/<uuid>/` — the `@Sdir` target. |
| `checkpoint_path` | `() -> str` | `checkpoints/<scan>/<sampler>/state.pkl` (frozen UX). |
| `resolve_runtime_tokens` | `(text, *, sample_info) -> str` | Replace `@SampleID/@Sdir/@PackID`; resolving `@Sdir` triggers `Sample.materialize()` (lazy). |
| `bucket_for` | `(uuid) -> str` | Bucket assignment — **control-resolved**; the Worker receives a pre-resolved `save_dir` string (never allocates buckets, invariant from §4). |

---

## 4. Path & token flow

```
control process:  resolve_project_root → expand_J on static config (Phase 1)
                  bucket_for(uuid) → save_dir string  (control allocates, never the Worker)
Worker, per Sample:  resolve_runtime_tokens(cmd, sample_info)
                        @SampleID → uuid
                        @Sdir     → pre-resolved save_dir (materialize on first touch)
                        @PackID   → acquired slot id / pack_id
```

Bucket allocation staying control-local is a hard rule (carried from the throughput-core
lessons): Workers receive **pre-resolved** `save_dir` strings, so no shared allocator crosses a
process boundary.

---

## 5. Concurrency / lifecycle / failure semantics

- `&J/` and output-layout resolution are **deterministic, cached** — cheap and process-safe.
- `@Sdir` resolution is the **lazy-materialization trigger** (M1): the directory is created only
  when a token actually needs it; opera-only success leaves `SAMPLE/` empty.
- `@PackID` binds to the slot acquired from the Redis free-pool ([calculator.md](calculator.md))
  — unique per run for traceability (`pack_id`).
- Output-contract paths (`DATABASE/` layout, `SAMPLE/<bucket>/<uuid>/`, checkpoint location) are
  **frozen** (invariant #3/#4).

---

## 6. Interfaces

- **Every component** inherits `Base` (path helpers).
- **Sample** → `sample_dir`, `resolve_runtime_tokens` (materialize on `@Sdir`).
- **CommandParser** → Phase-1 `expand_J`; Phase-2 delegates `@…` to `resolve_runtime_tokens`.
- **Archiver** → `output_dir("staging"|"SAMPLE"|"DATABASE")`.
- **Checkpoint** → `checkpoint_path`.

---

## 7. Tests (`tests/test_paths_tokens.py`)

Unit:
1. **&J/ resolution** — finds the project anchor from a nested cwd; `expand_J` yields the right
   absolute path; cached.
2. **Output layout** — `output_dir`/`sample_dir`/`checkpoint_path` match the frozen contract
   byte-for-byte vs. a V1 golden tree.
3. **Token resolution** — `@SampleID/@Sdir/@PackID` resolve against a sample_info; an unknown
   `@token` raises.
4. **Lazy `@Sdir`** — resolving `@Sdir` triggers `materialize()`; not touching it leaves
   `SAMPLE/` empty (M1 regression).
5. **Bucket is control-local** — `bucket_for` runs in the control process; the Worker only ever
   receives a resolved `save_dir` string (assert the Worker has no allocator).
6. **Parity** — `resolve_runtime_tokens` output equals V1 for the same inputs (golden).

Verification logic: test 2 keeps the frozen output layout; test 5 enforces the "no shared
allocator across processes" rule.

---

## 8. Open questions

- Whether bucket layout changes for very high Worker counts (keep V1 layout; revisit only if a
  single SAMPLE dir becomes an FS hotspot).
