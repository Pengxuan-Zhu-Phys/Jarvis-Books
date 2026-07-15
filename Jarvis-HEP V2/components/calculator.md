# Component — CalculatorModule (`jarvishep2/Module/calculator.py`)

**Role**: run one external calculator for one Sample. **Held long-term by a Worker** (one instance
per type), templates pre-loaded, executed through the Worker's `AsyncSubprocessScheduler`, and
concurrency-capped by the Redis free-pool.
**Status**: **As-built** @ `jarvis2` **`399633b`**+ (HEP FileOperation save after Portal R/W).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §4, §8;
[`../DESIGN_PORTAL_IO_2.0.md`](../DESIGN_PORTAL_IO_2.0.md).
**Reuses V1**: none by import.

> **As-built drift:** the design said `CalculatorModule(Module)` extends a V1 base. **Shipped as a
> standalone class** (no `Module` ABC — see [module_base.md](module_base.md)). `execute` takes a
> `runtime_prepared` flag; install is `ensure_shadow_installed` (not `install()`); calculator
> **format** I/O is Portal ([`io_portal`](io_system.md)); **SAMPLE `save:`** is HEP
> `FileOperationService` after each write/read; symlink isolation via [`LibraryManager`](library.md).

---

## 1. Class defined — `CalculatorModule`

External calculator runner with template preload and a synchronous `execute`.

**Attributes** (from `__init__`):

| Attribute | Type | Meaning |
|-----------|------|---------|
| `name`, `config` | str, dict | module name + raw config |
| `clone_shadow` | bool | per-pack physical install vs symlink |
| `installation`, `initialization` | list | shadow install/init command stages |
| `execution`, `commands_template`, `input_specs`, `output_specs` | dict/list | execution block + command/I/O templates |
| `timeout` | float\|None | per-run wall clock |
| `basepath`, `source`, `symlink_name` | str | runtime path, source tree, symlink name |
| `env_setup` | list | env-capture sources |
| `_subprocess_env` | dict\|None | cached environment (`bind_env`) |
| `sample_info`, `PackID` | dict, str\|None | current run context + pack id |
| `_templates_loaded`, `_template_parse_count`, `_command_counter` | bool/int | preload + command indexing |
| `_scheduler`, `_command_parser`, `_expression_context` | attached handles | scheduler + Phase-2 resolver + expressions |
| `_file_ops` | `FileOperationService`\|None | SAMPLE save/copy (Worker attaches) |
| `_installed_shadows` | set | pack_ids already installed |
| `_library` | `LibraryManager` | symlink helper |

**Member functions:**

| Method | Behavior |
|--------|----------|
| `_normalize_timeout` (`@staticmethod`) | `>0` float or None. |
| `custom_format` (`@staticmethod`) | V1-compatible log format hook. |
| `preload_templates()` | cache templates once per Worker (idempotent). |
| `attach_scheduler` / `attach_command_parser` / `attach_expression_context` / `attach_file_ops` | bind per-Worker handles (scheduler, parser, expressions, FileOperation). |
| `bind_env(env)` | store cached env-setup vars. |
| `env_setup_sources()` | list of `source` scripts from `env_setup`. |
| `acquire_pack_id(pack_id)` | tag the current run (`PackID`). |
| `decode_shadow_path(path)` / `decode_shadow_commands(command)` | resolve `@PackID` in paths/commands (clone_shadow). |
| `shadow_runtime_path()` | abspath of the decoded basepath. |
| `_expand_install_tokens(text)` | replace `${source}` / `${path}`. |
| `_stage_command(command, *, stage)` | decode shadow + expand install tokens → `{cmd,cwd}`. |
| `_run_stage_commands_sync(commands, *, stage)` | run install/init stages serially. |
| `ensure_shadow_installed()` | per-pack physical install (`copytree` from source or `installation` commands, then `initialization`); cached in `_installed_shadows`. |
| `ensure_symlink_runtime(sample_info)` | non-shadow: materialize sample dir, symlink the tool in via `LibraryManager`. |
| `prepare_runtime(sample_info)` | set `sample_info`; install (shadow) or symlink (safe). |
| `_logger()` | per-Sample logger from `sample_info`. |
| `_resolve_runtime_tokens(text, *, stage, field)` | delegate to `CommandParser.resolve_sample` if attached, else inline `@PackID/@SampleID/@Sdir` resolution (materializes on `@Sdir`). |
| `_next_command_index()` | monotonic command counter. |
| `_terminate_process(process, grace)` (`@staticmethod async`) | SIGTERM the process group, then SIGKILL. |
| `_run_command_sync(command, *, stage, command_index, timeout_sec)` | resolve tokens, ensure cwd, submit a `SubprocessJob` to the scheduler; raise on non-ok. |
| `run_command(...)` (`async`) | scheduler path → `_run_command_sync`; else `asyncio.create_subprocess_shell` with timeout + terminate. |
| `load_input` / `read_output` | Portal format R/W then HEP `apply_hep_io_save` (YAML `save:` → SAMPLE / `.temp`). |
| `_execute_commands_sync` / `execute_commands` (`async`) | run the command template with a deadline. |
| `_execute_sync` / `_execute_async` | input → commands → output, returning merged observables. |
| `execute(sample_info, *, runtime_prepared=False) -> dict` | **sync entry**: set context, optionally `prepare_runtime`, pick scheduler-sync vs `asyncio.run`, clear context in `finally`. |
| `from_config_list(modules)` (`@classmethod`) | build `{name: CalculatorModule}` + `preload_templates`. |

Module function `mint_pack_id() -> str` — fresh uuid pack id (local/testing path).

---

## 2. Execution within the Worker

```
Worker._run_calculator_step(step, sample):
  pack_id = redis.acquire_calc(step.name)         # blocks on calc:free:<name>
  calc.acquire_pack_id(pack_id); calc.prepare_runtime(sample.info)
  updated = calc.execute(sample.info, runtime_prepared=True)
  finally: redis.release_calc(step.name, pack_id) # always
```

Same-layer calculators run on the Worker's `ThreadPoolExecutor`; each acquires its own slot, so
global concurrency stays capped at the configured pool size across all Workers.

---

## 3. Concurrency / isolation / failure semantics

- One instance per type per Worker, reused; each *run* takes a fresh slot + `pack_id`.
- `clone_shadow` → per-pack physical dir; safe tools → symlink ([library.md](library.md)).
- `timeout` enforced via the scheduler / async path; failure raises and marks the Sample Failed.
- `env_setup` applied once per Worker via `bind_env` ([env_setup.md](env_setup.md)).

---

## 4. Interfaces / collaborators

- **Worker** ([worker.md](worker.md)) constructs + drives it.
- **RedisQueue** ([redis_queue.md](redis_queue.md)) free-pool acquire/release.
- **AsyncSubprocessScheduler** ([subprocess_scheduler.md](subprocess_scheduler.md)) runs commands.
- **CommandParser** ([command_parser.md](command_parser.md)) Phase-2 token resolution.
- **io_json** ([io_system.md](io_system.md)) JSON I/O; **LibraryManager** ([library.md](library.md))
  symlink; **env_setup** ([env_setup.md](env_setup.md)) env capture.

---

## 5. Tests

`tests/test_worker_calculator.py` (8), `test_clone_shadow.py` (5), `test_env_setup.py` (11),
`test_library.py` (5): execute parity, template preload-once, slot discipline, clone_shadow
isolation, env_setup capture, pack_id traceability.
