# Component — env_setup (`jarvishep2/env_setup.py`)

**Role**: activate `source`-based environments (e.g. Rivet's `rivet_env.sh`) for calculator
subprocesses by capturing the sourced environment **once per Worker** and merging it into the
subprocess env.
**Status**: **As-built** @ `jarvis2` `d0de31a`. `env_setup.py` 135 lines.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §8.
**Reuses V1**: none by import.

---

## 1. Class defined — `EnvCapture`

Capture `source`-based environments once per Worker process. Type alias
`EnvRunner = Callable[..., subprocess.CompletedProcess[str]]`.

**Class attributes:** `_cache: dict[str, dict[str,str]]` (script-key → captured env),
`_runner: EnvRunner | None` (injectable for tests).

| Method | Behavior |
|--------|----------|
| `set_runner(runner)` (`@classmethod`) | inject a subprocess runner (tests). |
| `clear_cache()` (`@classmethod`) | drop the cache. |
| `capture_from_source(script, base_env=None)` (`@classmethod`) | run `bash -c 'source <script> && env'`, parse `KEY=VALUE` lines; cache by `(abspath, hash(base_env))`; raise on missing script / non-zero exit / empty output. |
| `merged_env(scripts, base_env=None)` (`@classmethod`) | start from base/`os.environ`, apply each captured env in order, return merged. |
| `_cache_key(script_path, base_env)` (`@staticmethod`) | abspath + sha256(base_env)[:16]. |
| `_run_capture(command, *, env)` (`@classmethod`) | invoke `_runner` or `subprocess.run` (capture, text, no check). |
| `_parse_env(text)` (`@staticmethod`) | parse `env` output into a dict (skip blank/`#`). |

---

## 2. Module-level function

| Function | Behavior |
|----------|----------|
| `resolve_env_setup_sources(env_setup, *, command_parser=None) -> list[str]` | extract `source` script paths from `env_setup` entries, optionally Phase-1-resolving each via `command_parser.resolve_static`. |

---

## 3. Capture flow

```
Worker._init_runtime: for each calculator with env_setup:
   sources = resolve_env_setup_sources(module.env_setup, command_parser=parser)
   module.bind_env(EnvCapture.merged_env(sources))      # sourced ONCE per Worker (cached)
...
calc subprocess job runs with the cached env on every Sample.
```

The expensive `source` happens at Worker startup; per-Sample execution reuses the cached dict.

---

## 4. Concurrency / failure semantics

- Cache key includes the script abspath + a hash of the base env (different base envs don't
  collide); cache is per-process (per Worker).
- Missing/erroring/empty environment → clear `RuntimeError`/`FileNotFoundError` (no silent empty
  env).
- Only scripts named in the validated YAML are sourced.

---

## 5. Interfaces / collaborators

- **Worker._init_runtime** ([worker.md](worker.md)) → `merged_env` per calculator with `env_setup`.
- **CalculatorModule.bind_env** ([calculator.md](calculator.md)) stores the dict; the scheduler
  passes it as `SubprocessJob(env=…)` ([subprocess_scheduler.md](subprocess_scheduler.md)).
- **CommandParser** ([command_parser.md](command_parser.md)) Phase-1-resolves source paths.

---

## 6. Tests

`tests/test_env_setup.py` (11): capture, source-once caching, subprocess visibility, failure
errors, merge order.
