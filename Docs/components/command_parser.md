# Component — CommandParser (`jarvishep2/command_parser.py`)

**Role**: two-phase resolution of calculator commands and paths. Resolve everything static
once after YAML load; leave only strong per-Sample tokens for the Worker to resolve at run
time. Also owns `registered_executables`.
**Status**: design — plan WP-D3.1.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §8;
discussion `Jarvis-HEP_Discussion_Summary_2026-06-21.md` §5.
**Reuses V1**: the token-resolution helpers in `jarvishep/base.py` and
`CalculatorModule._resolve_sample_runtime_tokens` — consolidated and split into two phases.

---

## 1. Responsibilities

1. **Phase 1 (after YAML load, once)**: resolve `registered_executables`, `LibDeps` paths,
   `&J/` project-root markers, anchors, and all static tokens. Output a "Phase-1 resolved"
   config with **no** static tokens left.
2. **Phase 2 (in the Worker, per Sample)**: resolve only `@SampleID`, `@Sdir`, `@PackID`.
3. Replace repeated `ln -sf` boilerplate via `registered_executables`
   (`resolution: direct_path | symlink`).
4. Guarantee that after Phase 1, a command string contains **only** per-Sample tokens (or
   none) — a property the tests assert.

---

## 2. Structure

```python
class CommandParser:
    def __init__(self, project_root: str, libdeps: dict, registered: list[dict]): ...
    registered: dict[str, ResolvedExecutable]     # name → path
    project_root: str

@dataclass
class ResolvedExecutable:
    name: str
    path: str
    resolution: str        # direct_path | symlink

STATIC_TOKENS  = ("&J/", "${LibDeps:...}", "${Scan:...}", registered names)
SAMPLE_TOKENS  = ("@SampleID", "@Sdir", "@PackID")
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `from_config` | `(cls, config) -> CommandParser` | Build from `LibDeps.registered_executables` + project root. |
| `resolve_static` | `(text: str) -> str` | **Phase 1**: expand `&J/`, `${LibDeps:…}`, `${Scan:…}`, registered executable names. |
| `resolve_static_config` | `(config: dict) -> dict` | Apply `resolve_static` across a whole module config (commands, paths, io). |
| `resolve_sample` | `(text, *, sample_info) -> str` | **Phase 2**: expand `@SampleID/@Sdir/@PackID` (Worker-side). |
| `has_static_tokens` | `(text) -> bool` | Test helper: True if any Phase-1 token remains (must be False after Phase 1). |
| `register` | `(spec) -> ResolvedExecutable` | Resolve one `registered_executables` entry: `direct_path` (default; zero cleanup) or create a `symlink`. |

---

## 4. Two-phase flow

```
YAML load:                 raw config
  └─ CommandParser.resolve_static_config(config)   → Phase-1 config (static tokens gone)
       ├─ registered_executables → direct paths / symlinks
       ├─ &J/ → project root, ${LibDeps:X} → resolved dep path
Worker, per Sample:
  └─ CommandParser.resolve_sample(cmd, sample_info) → final command (@SampleID/@Sdir/@PackID)
```

`registered_executables` example:

```yaml
LibDeps:
  registered_executables:
    - name: eggboxlk
      source: "${LibDeps:EggBoxSafe}/eggbox"
      resolution: "direct_path"     # default; symlink optional
```

After Phase 1, a command like `"eggboxlk @Sdir/in.json"` becomes
`"/abs/EggBoxSafe/eggbox @Sdir/in.json"`; the Worker later resolves `@Sdir`.

---

## 5. Concurrency / lifecycle / failure semantics

- Phase 1 runs **once** in the control process; the resolved config is what gets shipped to
  Workers (picklable strings, no closures).
- Phase 2 is **pure string substitution** per Sample — cheap, thread/process-safe, no I/O
  except `direct_path` lookups (cached).
- `symlink` resolution creates links at install time and is the only mode with cleanup
  considerations; `direct_path` is the recommended default (zero cleanup).
- Unresolved token after the wrong phase → raise with a clear message (no silent passthrough).

---

## 6. Interfaces

- **config loader / core** → `resolve_static_config` after YAML validation.
- **Worker / CalculatorModule** → `resolve_sample` at execution (Phase 2).
- **LibDeps / library.py** → `register` for executables.

---

## 7. Tests (`tests/test_command_parser.py`)

Unit:
1. **Phase separation** — after `resolve_static_config`, `has_static_tokens(...)` is False for
   every command; only `@…` Sample tokens remain.
2. **Registered executable** — `direct_path` expands to the absolute path; `symlink` creates a
   link and expands to it.
3. **Phase 2** — `resolve_sample` expands `@SampleID/@Sdir/@PackID` against a sample_info; a
   leftover static token raises.
4. **Parity** — a calculator command resolved via CommandParser is identical to the V1
   `_resolve_sample_runtime_tokens` output for the same inputs (golden).
5. **Picklability** — the Phase-1 resolved config pickles under spawn.

Verification logic: test 1 is the invariant the design promises (no static tokens cross into
the Worker), test 4 keeps command resolution byte-equal to V1.

---

## 8. Open questions

- Default `direct_path` vs `symlink` per tool class (default direct_path).
- Caching location for resolved registered executables (in-memory per process is enough).
