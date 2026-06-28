# Component — Project tools (`jarvishep2/project_scaffold.py`, `project_packager.py`, `official_project_library.py`)

**Role**: the `Jarvis2 project …` subcommands — scaffold a new standalone project, pack one for
sharing/reproduction, and browse/fetch the official project library.
**Status**: design — auxiliary; reused largely as-is from V1, retargeted to `Jarvis2`.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §11 (reused);
CLAUDE.md CLI reference; [cli.md](cli.md).
**Reuses V1**: `project_scaffold.py`, `project_packager.py`, `official_project_library.py`,
`card/template/`, `card/official_project_library.json`. Behavior is unchanged; the V2 work is
ensuring scaffolds emit V2-shaped YAML defaults.

---

## 1. Responsibilities

1. **`create`** — scaffold a standalone project (`bin/`, `data/`, `deps/`,
   `.jarvis-project.json`) from `card/template/`.
2. **`pack`** — package a project (`--share` / `--repro` / `--full`, optional `--man` manifest).
3. **`browse` / `fetch` / `info`** — official project library access.
4. Ensure the scaffold's default task YAML is **V2-aware** (optionally emits a `Runtime.mode:
   redis` example + `Archiver` block, all commented/optional so it still runs on V1 too).

---

## 2. Structure

```python
class ProjectScaffold:
    def create(self, name: str, *, template="default") -> str: ...
class ProjectPackager:
    def pack(self, path: str, *, mode="share", manifest=False) -> str: ...   # share|repro|full
    def pack_from_manifest(self, manifest_path: str) -> str: ...
class OfficialLibrary:
    def browse(self) -> list[dict]: ...
    def fetch(self, name: str) -> str: ...
    def info(self, name: str) -> dict: ...
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `ProjectScaffold.create` | `(name, template) -> path` | Copy `card/template/` → new project; write `.jarvis-project.json`; emit a starter task YAML (V2-aware, optional keys commented). |
| `ProjectPackager.pack` | `(path, mode, manifest) -> archive` | Bundle by mode: `share` (configs+bin), `repro` (+ data/deps refs), `full` (everything); optional `--man` manifest. |
| `ProjectPackager.pack_from_manifest` | `(manifest) -> archive` | Pack per an explicit `pack-manifest.yaml`. |
| `OfficialLibrary.browse/fetch/info` | `(...)` | List / download / describe official projects (`card/official_project_library.json`). |

---

## 4. Lifecycle / failure semantics

- All are **control-process, one-shot** CLI actions — no Workers, no Redis.
- `create` refuses to overwrite an existing non-empty target (clear message).
- `fetch` validates the downloaded project against the schema before declaring success
  (reuses [config_schema.md](config_schema.md)).
- `pack` is deterministic given the same inputs (stable archive for reproducibility).

---

## 5. Interfaces

- **cli.py** → `Jarvis2 project {create,pack,browse,fetch,info}`.
- **config_schema** → validate fetched/scaffolded YAML.
- **paths_tokens** → `.jarvis-project.json` anchor + `&J/` root for new projects.

---

## 6. Tests (`tests/test_project_tools.py`)

Unit:
1. **Scaffold** — `create` produces a project that **validates** and runs `--check-modules` on
   `Jarvis2` end to end.
2. **V2-aware default** — the scaffolded YAML's optional `Runtime`/`Archiver` examples validate
   and are inert by default (still V1-runnable).
3. **Pack modes** — `share`/`repro`/`full` archives contain the expected file sets; deterministic.
4. **Library** — `browse`/`info` parse the library card; `fetch` downloads + validates a project.
5. **No overwrite** — `create` into a non-empty dir errors clearly.

Verification logic: test 1 ensures a fresh project actually runs on V2; test 2 keeps scaffolds
backward-compatible (optional keys only).

---

## 7. Open questions

- Whether `create` should default to a Redis example or stay V1-shaped with V2 as commented
  options (default: V1-shaped + commented V2 — runs anywhere out of the box).
- A `Jarvis2 project migrate` helper to add a `Runtime.mode: redis` block to an existing project
  (nice-to-have; defer).
