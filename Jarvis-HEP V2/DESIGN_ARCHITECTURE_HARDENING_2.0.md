# Jarvis-HEP V2 — Architecture Hardening 2 (D25)

**Status**: Active design — structure track (behavior-preserving)
**Date**: 2026-08-20
**Code baseline**: `jarvishep2` 2.0.4 (`Jarvis` CLI, import package `jarvishep2`)
**Supersedes as the live principles review**: [`archive/reviews/DESIGN_PRINCIPLES_REVIEW_2.0.md`](archive/reviews/DESIGN_PRINCIPLES_REVIEW_2.0.md) (2026-07-04, produced **D9**; Core was 676 lines). D9 shipped. This document is the second hardening pass after D13–D24 landed on the same files.
**Ledger**: [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md) milestone **D25**.
**Audience**: maintainers and coding agents.

---

## 0. Why this exists

The V2 data plane is coherent and should be kept:

```text
Task YAML → sampler → Redis → Workers → Archiver
                                      ├─ DATABASE/samples.hdf5
                                      ├─ SAMPLE/<bucket>/<uuid>/
                                      └─ logs/<scan>/ + run_summary.*
```

D9 split `TaskFactory` into `_MonitorLoop` / `_Watchdog`, introduced the sampler
registry, and collapsed `Sample.info` dual-writes. Since then the *product* grew
(MCMC, nested, AdaptiveBridson, strict schema, resume, multimode, man, Mapper)
**on the same control-plane files**. Measured sizes on 2026-08-20:

| File | Lines | Problem |
|---|---:|---|
| `Sampling/adaptive_bridson.py` | 4165 | whole algorithm + control loop in one module |
| `man.py` | 3516 | schema walk + hardcoded EnvReqs/Calculator help + TUI |
| `core.py` | 2702 | one class, ~80 methods: YAML, Redis daemon, resume, check, convert, plots, signals |
| `client.py` | 1968 | help chrome + run adapter + entire `project` tool |
| `redis_queue.py` | 1782 | codec + task/archive/feedback + calc pools + buckets + lock + heartbeats |
| `Sampling/mcmc_sampler.py` | 1504 | transport + science + `create_*` barrel |
| `worker.py` | 1001 | process lifecycle + physics + SAMPLE layout |
| Package root `jarvishep2/*.py` | 59 modules | layers exist as filenames, not directories |

The 2026-07-04 review is not wrong; it is **stale**. Repeating D9 on today's tree
without a new plan would either no-op or fight shipped behavior.

This track does **not** add cluster, analyze, or Agent JSON. Those stay D14 / D15 / D8.
It makes those features cheaper to land by shrinking the files they would otherwise grow.

---

## 1. Binding decisions

1. **Do not rewrite the pipeline.** Redis + spawn Workers + Archiver stays the only runtime.
2. **Behavior-preserving until a WP says otherwise.** YAML, CLI exit codes, HDF5/CSV,
   checkpoint paths, `Sampling.Method` names, and golden fixtures do not change.
3. **Do not start with a package rename.** `Sampling/` / `Module/` / `SamplingVirtial`
   stay until D25.10. Code comments point at `Jarvis-Books/Jarvis-HEP V2/DESIGN_*.md`
   at the *root* of this tree — do not move those design files.
4. **One catalog for method names.** Adding a sampler must not require editing
   `manifest.json` + `distributor.py` + `contracts/methods.py` + `mapper.py` + `man.py`
   + `worker_config.py` by hand.
5. **Validation and `Jarvis man` / `Jarvis validate` must not import the sampler
   implementations.** Today `task_validation` imports `Distributor`, which registers
   every builtin factory.
6. **Copy `factory.py`, do not copy D9's file list.** The pattern is a façade plus
   private collaborators. `Jarvis2Core` still *looks* like one object to `client.py`
   and tests after D25.3.
7. **D13.12 stays D13.12.** AdaptiveBridson loop decomposition is already scoped;
   D25 must not duplicate it. D25.3/D25.5 must not expand `adaptive_bridson.py`.
8. **Invariant 12 in the live plan (`SamplingVirtial`, `make_paraller`) still holds**
   until D25.10 explicitly lifts it.

### Do not touch (red lines from the 2026-08-20 review)

- Spawn-picklable Worker / Archiver construction (`run()` builds live objects).
- `workflow.py` as a pure function.
- Portal vs HEP save/copy/delete split (`io_portal` / `file_ops` / `file_operation_service`).
- Factory does not execute Samples.
- Workers do not import `Sampling/`; samplers do not import `worker.py`.
- JSON Schema = structure, Python contracts = cross-field / numeric / path / registry.
  Do not generate `min < max` into JSON Schema.
- Vendored Dynesty UUID channel. Do not switch to site-packages dynesty.

---

## 2. What is already right

Keep these; D25 refactors *around* them.

1. Closed task-card vocabulary: `schema/` + `x-jarvis-zone` + `contracts/` + `Jarvis man`.
2. `Distributor.register` + lazy factories (the kernel of a plugin API).
3. MCMC family as thin profiles over `MCMCBaseSampler` + `_uses_half_ensemble` / `_uses_pt`.
4. Nested constructor keys already flow `dynesty_sampler.NESTED_CONSTRUCTOR_USER_KEYS` →
   `contracts/nested.py` → tests. **That** is the catalog pattern to copy.
5. I/O type names: Portal is SSOT; manifest is enrichment (D13.14 still has to finish
   making validation match that claim).
6. `RunOutcome` as the CLI contract; Worker never dies on physics errors.
7. Checkpoint format firewall (`jarvis-hep.v2-distributed`, V1 refusal).

---

## 3. Findings (evidence, not slogans)

### 3.1 Method catalog is written at least four times

| Copy | Where |
|---|---|
| Manifest dispatch | `jarvishep2/schema/manifest.json` `sampling_methods` |
| Runtime registry | `distributor.register_builtin_samplers()` |
| Contract families | `contracts/methods.py` `_METHODS_NEED_VARIABLES` / `_MCMC_METHODS` / `_NESTED_METHODS` |
| Mapper warnings | `mapper.py` `STATISTICAL_METHODS` |
| Man families | `man.py` `_NESTED_SAMPLER_METHODS` / MCMC frozensets |
| Worker fallback | `worker_config.py` hardcoded feedback-method list (~lines 131–144) |

`tests/test_task_schema.py` only asserts **manifest ↔ Distributor**.
Root schema `core/sampling.json` types `Method` as a nonempty string, not an enum —
unknown methods pass JSON Schema and fail later as `JV2-MTH-003`.

### 3.2 `Jarvis2Core` is the product bus

`core.py` owns load/validate, env pretty-print, LibDeps, managed Redis, control lock,
sampler init, factory, archiver, check-modules, convert, flowchart, plots, resume,
signals, shutdown. Tests and the CLI both talk to this one type, so every flag lands here.

### 3.3 Validation import graph contradicts the docstring

`task_validation.py` is a pure report object (good) but imports `Distributor`
(`task_validation.py` ~line 21) and calls `available_methods()`.
`contracts/nested.py` imports `Sampling.dynesty_sampler`.
`contracts/mapper.py` imports runtime `mapper.py` (including a private helper).
`client.py` top-level-imports `Jarvis2Core` and `TaskFactory`, so `Jarvis validate`
and `Jarvis -v` pay the orchestrator import graph.
`core.py` imports `jarvishep2.testing.check_modules` for a **product** command.

### 3.4 `RedisQueue` is a broker SDK

One class: payload codec, task/archive/feedback lists, exclusive + shared calculator
pools (Lua), SAMPLE bucket state, control lock, heartbeats, monitor snapshots.
`sample_bucket` and `feedback_return` leak into it.

### 3.5 MCMC public names collide

- `Sampling/mcmc.py` `MCMCSampler` = YAML `Method: MCMC`
- `Sampling/mcmc_sampler.py` `MCMCSampler = MCMCBaseSampler` (compatibility alias)
- `Sampling/mcmc_base.py` three-line re-export
- `tests/test_mcmc_sampler.py` imports the **base**, not the YAML profile

`Sampling/__init__.py` only exports Bridson/CSV/Grid/`RandomS`/`SamplingVirtial`.
Production construction is `Distributor.set_method`. Dynesty subclasses
`FeedbackSampler` but no-ops `propose_generation` / `absorb_generation`; the real
path is `RedisEvaluationPool`.

### 3.6 Default pytest ignores the expensive runtime

`pyproject.toml` `--ignore`s AdaptiveBridson, ensemble, distributed acceptance,
distributed resume, MCMC, worker pool, variable distributions, worker failure.
Green on `pytest -q` does not mean those paths work. Overlaps **D13.15**.

### 3.7 Config is projected twice

User YAML is `EnvReqs.V2`. Loader injects `Runtime`. `get_runtime_block` merges both
again. `RUNTIME_DEFAULTS["mode"] = "auto"` but the loader always writes `"redis"`;
`is_redis_runtime()` only accepts `"redis"`. `mode: auto` in a raw Python config
skips Factory.

---

## 4. Work packages

Statuses live in [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md). One WP = one PR.
Do not start a later WP until its `Depends on` rows are `done`.

### D25.0 — Docs library hygiene (this document's sibling change)

- **Goal**: this Books tree is navigable. Dated reviews leave the root. README is a
  current map, not a 2026-07-21 changelog.
- **Files**: `Jarvis-Books/Jarvis-HEP V2/README.md`, `archive/`, this file, live plan.
- **Accept**: root no longer lists dated `*_REVIEW_*.md`; `archive/reviews/README.md`
  indexes them; live plan next-pick includes D25.
- **Out of scope**: rewriting every `components/*.md` member table; TeX manuals.

### D25.1 — `SamplerSpec` is the method catalog (P0)

- **Goal**: one Python catalog drives Distributor, manifest keys, contract families,
  `STATISTICAL_METHODS`, and man sampler families. Adding a method is "register a spec
  + add `schema/sampling/methods/<name>.json`", not a six-file hunt.
- **Design refs**: this §3.1; nested-keys pattern in `contracts/nested.py`;
  [`DESIGN_DISTRIBUTOR_REGISTRY_2.0.md`](DESIGN_DISTRIBUTOR_REGISTRY_2.0.md).
- **Files**: NEW `jarvishep2/sampler_catalog.py` (names + flags only, **no** sampler
  class imports); `distributor.py`; `contracts/methods.py`; `mapper.py`;
  `worker_config.py` (delete hardcoded fallback list — `publish_feedback = method not in STATELESS_METHODS`);
  `schema/core/sampling.json` (Method enum *or* generated from catalog in tests);
  `tests/test_task_schema.py` / NEW `tests/test_sampler_catalog.py`.
- **Steps**:
  1. `SamplerSpec(name, stateless, resume, statistical, nested, schema_id, factory=None)`.
     Factories stay lazy in `distributor.py` so the catalog module stays import-safe
     for `task_validation`.
  2. Test-lock: `set(manifest.sampling_methods) == set(catalog.names) == set(Distributor.available_methods())`.
     `STATISTICAL_METHODS == catalog.statistical`. `_MCMC_METHODS` / `_NESTED_METHODS`
     become catalog views.
  3. Root schema: unknown `Sampling.Method` is a schema error (`JV2-SCH-001` + did-you-mean)
     *or* keep `JV2-MTH-003` but make the schema enum match the catalog. Pick one code
     and document it in `man_codes.py`.
- **Accept**: adding a fake test method via `Distributor.register` without a spec fails
  the catalog test; deleting the `worker_config` fallback list does not change feedback
  on AdaptiveBridson / MCMC / Dynesty; `Jarvis validate` on an unknown Method still
  exits 2 with zero side effects.
- **Rollback**: revert the commit. Do not leave half-updated frozensets.
- **Out of scope**: closing MCMC Bounds (still `x-jarvis-status: unstable`); D13.14 Portal I/O.

### D25.2 — Validation / CLI do not import the runtime graph (P1)

- **Goal**: `Jarvis validate`, `Jarvis man`, and `import jarvishep2` (version/helpers)
  do not load Worker / Redis / sampler implementations.
- **Depends on**: D25.1 (catalog without factories).
- **Files**: `task_validation.py`; `contracts/nested.py`; `client.py`;
  `jarvishep2/__init__.py`; `core.py` (`testing.check_modules` → product module);
  `man.py` (Distributor import).
- **Steps**:
  1. `task_validation` and `man` read method names from `sampler_catalog`, not
     `Distributor.available_methods()`.
  2. Move `NESTED_CONSTRUCTOR_USER_KEYS` next to the catalog/contracts (keep a
     re-export in `dynesty_sampler` so current imports do not break).
  3. `client.py`: lazy-import `Jarvis2Core` / `TaskFactory` inside `dispatch_run` /
     `dispatch_monitor`.
  4. `__init__.py`: export version + validation/sample types; Worker/Core become
     lazy or documented as `jarvishep2.core`.
  5. Move `jarvishep2/testing/check_modules.py` to a product path (e.g.
     `jarvishep2/check_modules.py`). Keep `testing/` for spawn-safe *test* operators.
- **Accept**: a new test that imports `task_validation` (and, separately, `client` +
  `dispatch_validate`) fails if `jarvishep2.worker` or `Sampling.mcmc_sampler` is
  already in `sys.modules`. `Jarvis validate` on a good card still exits 0.
- **Out of scope**: splitting `man.py` pages; Typer/Click help rewrite.

### D25.3 — Split `Jarvis2Core` the way `TaskFactory` was split (P0)

- **Goal**: `Jarvis2Core` remains the façade `client.dispatch_run` and existing tests
  construct. Internals become collaborators Core *holds*.
- **Depends on**: D25.2 recommended (so validate tests do not construct Core).
- **Files**: `core.py`; NEW collaborators under `jarvishep2/` (names are local, not
  a new public package): e.g. `_RuntimeSupervisor`, `_ScanDriver`, `_ResumeService`
  — private, like `_MonitorLoop`. Do **not** invent a parallel public orchestrator.
- **Suggested cut** (methods already exist; move bodies, keep signatures on Core as
  thin delegates if tests monkeypatch them):

  | Collaborator | Takes from Core |
  |---|---|
  | RuntimeSupervisor | `init_redis`, managed Redis, control lock/lease, `init_factory`, `init_archiver`, signals, `shutdown` |
  | ScanDriver | `run_distributed_scan`, `run_adaptive_scan`, `run_check_modules`, `wait_for_results`, archive catch-up |
  | ResumeService | checkpoint path, `prepare_resume`, DATABASE seed, fingerprints |
  | Core façade | `load_task_yaml`, `validate_loaded_config`, `run()`, logging options |

  Convert / flowchart / plot stay on Core as *post-run product* methods for this WP
  (splitting them is D25.9). Do not fold them into RuntimeSupervisor.
- **Accept**: existing `tests/test_core_run_distributed.py`, `test_graceful_stop.py`,
  `test_run_outcome.py`, `test_cli.py` green without assertion changes. Public
  `Jarvis2Core.run()` / `shutdown()` signatures unchanged.
- **Out of scope**: D13.12 AdaptiveBridson; changing shutdown order
  (Archiver → lock → Factory → Redis last).

### D25.4 — Split `RedisQueue` by keyspace (P1)

- **Goal**: one connection object, several mixins or helper modules:
  TaskBroker (task/archive/feedback), CalcPool, SampleBuckets, ControlLock + HeartbeatStore.
- **Files**: `redis_queue.py`; callers should keep importing `RedisQueue` (façade).
- **Accept**: `tests/test_redis_queue.py`, calculator pool tests, sample-bucket tests
  green. No Redis key rename.
- **Out of scope**: changing Lua; broker auth (D14.3).

### D25.5 — `SampleExecutor` extracted from `Worker` (P1)

- **Goal**: `Worker` = Redis loop + heartbeat + FileOperation lifetime + signals.
  `SampleExecutor.process(sample)` = mapper, layers, calculators, operas, likelihood,
  nuisance. SAMPLE layout is a small helper, not inline in `process_task`.
- **Files**: `worker.py`; NEW `jarvishep2/sample_executor.py` (or similar);
  tests that currently spawn Workers for physics can, over time, call the executor
  in-process — not required in the first PR if that expands scope.
- **Accept**: `tests/test_worker_mvp.py`, `test_worker_calculator.py`,
  `test_layer_concurrency.py` green. Spawn pickling still only stores Redis settings.
- **Out of scope**: changing layer concurrency; FileOperation process vs inline.

### D25.6 — MCMC / Sampling public surface (P2)

- **Goal**: one meaning per name.
- **Depends on**: D25.1.
- **Steps**:
  1. Delete `mcmc_sampler.MCMCSampler = MCMCBaseSampler`. Update
     `tests/test_mcmc_sampler.py` to import `MCMCBaseSampler` or `mcmc.MCMCSampler`
     explicitly.
  2. Move `create_*` out of `mcmc_sampler.py`; Distributor already lazy-imports
     method modules.
  3. `Sampling/__init__.py`: export bases + pointer to Distributor; do not pretend
     the catalog is four coverage samplers.
  4. Dynesty/MultiNest: inherit `CheckpointedSampler` + pool, not the
     generation-barrier `FeedbackSampler` ABC (no-op propose/absorb is the smell).
- **Accept**: `Distributor.set_method("MCMC")` still returns `mcmc.MCMCSampler`;
  nested/MCMC tests that are *not* in the default ignore list stay green. Do not
  enable ignored tests here (D25.7).
- **Out of scope**: merging AM into AMMCMC YAML; renaming `SamplingVirtial`;
  engine copy-paste in `Source/MCMC/` (optional follow-up, same WP only if tiny).

### D25.7 — Test gate: markers, not silent `--ignore` (P1, overlaps D13.15)

- **Goal**: `pytest` with default addopts means "the fast gate is green". Slow /
  Redis-heavy tests are `pytest.mark.slow` (or similar) and run in an explicit job.
  Standing failures are `xfail`/`skip` **with a ticket**, or they are fixed.
- **Depends on**: can start in parallel with D25.1.
- **Files**: `pyproject.toml`; the eight ignored test modules; D13.15 notes.
- **Accept**: `pytest -q` (fast) has **zero** ignored-via-addopts test files;
  `pytest -q -m slow` documents how to run the rest; D13.15 can close when every
  previously ignored failure is either green, xfail-with-reason, or skipped-with-reason.
- **Out of scope**: rewriting AdaptiveBridson tests (D13.12).

### D25.8 — One runtime read model (P2)

- **Goal**: loader emits a single normalized runtime mapping. `get_runtime_block`
  is the only reader. `mode: auto` is either implemented or removed from
  `RUNTIME_DEFAULTS` (today it is a footgun).
- **Files**: `task_config.py`, `runtime_config.py`, `core.py` check-mode overrides.
- **Accept**: tests that skip the loader and pass a raw dict still work *or* are
  updated to call `normalize_runtime_block`. No YAML key changes.
- **Out of scope**: renaming `EnvReqs.V2` (user vocabulary stays).

### D25.9 — Slice `client.py` (P3)

- **Goal**: `cli/run.py` (validate/run/check/convert), `cli/project.py`,
  `cli/help.py`. `main()` stays the argv router. `dispatch_run` stays a thin
  adapter onto `Jarvis2Core.run`.
- **Depends on**: D25.2, D25.3.
- **Accept**: `tests/test_cli.py` green; `Jarvis project --help` does not import
  `worker.py`.
- **Out of scope**: replacing argparse with Typer as the parser (help already
  uses Click/Rich).

### D25.10 — Package layout (P3, last)

- **Goal**: directories match layers. Suggested (compatibility re-exports required):

  ```text
  jarvishep2/cli/        client, man
  jarvishep2/cardload/   task_config, task_schema, task_validation, contracts, schema
  jarvishep2/runtime/    core façade, factory, worker, worker_config
  jarvishep2/queue/      redis_*
  jarvishep2/io/         archiver, database, file_ops, sample_bucket
  ```

  `Sampling/` → `sampling/` and `Module/` → `calculators/` only in this WP, with
  `Sampling` / `Module` re-export packages. **Lift invariant 12** (`SamplingVirtial`
  rename) only here, with an alias.
- **Depends on**: D25.3–D25.6 actually landed; otherwise this is churn.
- **Accept**: `from jarvishep2.Sampling.bridson import Bridson` still works;
  entry point `Jarvis = jarvishep2.client:main` still works (or is updated and
  documented in INSTALL.md + this plan).
- **Out of scope**: moving Books `DESIGN_*.md` (code comments use those paths).

---

## 5. Sequencing against other milestones

```text
D25.0 docs          (now)
D25.1 catalog  ──►  D25.2 import graph  ──►  D25.3 Core split
D25.7 test markers   (parallel with D25.1; closes D13.15)
D13.12 AdaptiveBridson loop   (independent; do before growing that file)
D25.4 RedisQueue / D25.5 SampleExecutor   after D25.3
D25.6 MCMC names / D25.8 config           after D25.1
D14 cluster / D15 analyze                 may proceed after D25.3
                                          (do not grow Core before the split)
D8 Agent                                  stays parked
D25.9 CLI slice / D25.10 layout           last
```

**Next pick if you are on the structure track:** **D25.8**.
**Next pick if you are on the feature track:** D14.1 is allowed. Do **not** add
methods to `adaptive_bridson.py` until D13.12. New Core behavior belongs on
`_RuntimeSupervisor` / `_ScanDriver` / `_ResumeService`, not the façade.

---

## 6. Verification (every D25 WP)

```bash
python3 -m pip install -e '.[distributed,dev]'
python3 -m pytest -q
# after D25.7:
# python3 -m pytest -q -m slow
```

Parity against captured V1 goldens is unchanged. A WP that changes YAML diagnostics
must keep `Jarvis validate` exit 2 / zero side effects (D17).

---

## 7. Explicit non-goals

- Generating the entire JSON Schema from Python contracts.
- Making samplers independent of Redis (extract an `EvaluationSink` only if a WP
  needs in-process tests; do not fake a second runtime).
- Fortran MultiNest.
- Un-parking D8.
- Mass-renaming `Jarvis2` strings inside archived reviews.
