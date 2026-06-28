# Component — env_setup (`jarvishep2/env_setup.py`)

**Role**: activate `source`-based environments (e.g. Rivet's `rivet_env.sh`) for calculators
running in isolated subprocesses, by capturing the sourced environment once and injecting it
into `SubprocessJob(env=…)`.
**Status**: design — plan WP-D3.2.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §8;
discussion `Jarvis-HEP_Discussion_Summary_2026-06-21.md` §6.
**Reuses V1**: `SubprocessJob` already accepts `env` (`jarvishep/async_subprocess.py`); this
component only produces the env dict.

---

## 1. Responsibilities

1. Given a list of `source`-able scripts, **capture** the resulting environment variables.
2. **Cache** per (script, base-env) so each script is sourced **once per Worker**, not per
   Sample.
3. Merge with `os.environ` and hand the merged dict to the calculator's subprocess jobs.

---

## 2. Structure

```python
class EnvCapture:
    _cache: dict[str, dict[str, str]] = {}        # script-key → captured env

    @classmethod
    def capture_from_source(cls, script: str, base_env: dict | None = None) -> dict[str, str]: ...
    @classmethod
    def merged_env(cls, scripts: list[str], base_env: dict | None = None) -> dict[str, str]: ...
    @classmethod
    def clear_cache(cls) -> None: ...
```

YAML surface:

```yaml
Calculators:
  Modules:
    - name: RivetAnalysis
      env_setup:
        - source: "&J/External/Rivet/rivet_env.sh"
      execution:
        commands: ["rivet --analysis=… input.hepmc"]
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `capture_from_source` | `(script, base_env=None) -> dict` | Run `bash -c 'source <script> && env'`, parse `KEY=VALUE` lines into a dict. Cache by `(abspath(script), hash(base_env))`. |
| `merged_env` | `(scripts, base_env=None) -> dict` | Start from `base_env or os.environ`; apply each script's captured env in order; return merged dict. |
| `clear_cache` | `() -> None` | Drop the cache (tests / reload). |
| `_parse_env` | `(text) -> dict` | Parse `env` output (handle multi-line/edge values defensively). |

---

## 4. Capture flow

```
Worker._init_calculators:
   for calc with env_setup:
       env = EnvCapture.merged_env([s["source"] for s in calc.env_setup])
       calc.bind_env(env)                      # cached; runs the source ONCE per Worker
...
calc.run_command(...) → SubprocessJob(env=calc.env)   # every Sample reuses the cached env
```

The expensive `source` happens at Worker startup; per-Sample execution just reuses the cached
dict.

---

## 5. Concurrency / lifecycle / failure semantics

- **Cache key** includes the script abspath + a hash of the base env, so different base envs
  don't collide.
- Sourcing runs in a controlled `bash -c`; a non-zero exit or unparseable output raises with a
  clear message (no silent empty env).
- The cache is **per process** (per Worker) — safe, since each Worker has its own; no
  cross-process sharing needed.
- Security note: only scripts named in the validated YAML are sourced; no arbitrary
  user-string execution beyond that.

---

## 6. Interfaces

- **Worker._init_calculators** → `merged_env` once per calculator with `env_setup`.
- **CalculatorModule.bind_env** → stores the dict; `run_command` passes it to `SubprocessJob`.
- **CommandParser** → unrelated (env vs. tokens), but both run before execution.

---

## 7. Tests (`tests/test_env_setup.py`)

Unit:
1. **Capture** — a temp script `export FOO=bar` → `capture_from_source` returns `FOO=bar`
   merged over `os.environ`.
2. **Cache once** — `merged_env` called N times sources the script once (assert via a spy that
   counts subprocess invocations / a sentinel file appended on each source).
3. **Subprocess sees it** — a calculator with `env_setup` runs a command that echoes `$FOO`
   and the captured value appears in output.
4. **Failure** — a missing/erroring script raises a clear error, not a silent empty env.
5. **Merge order** — later scripts override earlier ones; base env preserved otherwise.

Verification logic: test 2 (source-once) is the performance contract — without it, env capture
would re-run per Sample and dominate light scans; test 3 is the functional gate.

---

## 8. Open questions

- Whether `env_setup` is a new field or folded into `initialization` (design §13.3 — proposed:
  new field, clearer intent).
- Caching across Workers via a shared file (probably unnecessary; per-Worker is cheap enough).
