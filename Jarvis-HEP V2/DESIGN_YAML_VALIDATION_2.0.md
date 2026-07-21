# DESIGN — Task YAML Validation (V2)

**Status**: **implemented** on `jarvis2` (maintainer approved recommended defaults)  
**Milestone**: **D13.9** (plan ledger) — early config contract; feeds D8 full Agent envelope when unparked  
**Note**: This design once used the label “D14” for the validation workstream; the distributed plan’s **D14** is reserved for cluster execution. Code/docs refer to **D13.9**.  
**Date**: 2026-07-21  
**Branch context**: `jarvis2` (post-D13.8 + D13.9)  
**Related**:
- V1: `jarvishep/config.py` (`ConfigValidator` + jsonschema draft-07)
- V1 schemas: `jarvishep/card/schema/*_schema.json`
- V2 as-built: [`components/config_schema.md`](components/config_schema.md),
  [`YAML_REFERENCE_2.0.md`](YAML_REFERENCE_2.0.md) §13 / Appendix A,
  [`DESIGN_AGENT_BRIDGE_2.0.md`](DESIGN_AGENT_BRIDGE_2.0.md) §4.2 / WP-D8.1–D8.4 (parked)
- Sampler Bounds reality: `jarvishep2/Sampling/dynesty_sampler.py` (`_filter_known_kwargs` warn-only)

---

## 0. Why this document exists

YAML is the **public API** of Jarvis-HEP. Wrong structure must not become wrong science
with a successful-looking exit. V1 already treated validation as a first-class gate;
V2 intentionally shipped a thin loader and deferred schema work. That gap is now
hurting users: typos and shape errors are often **coerced or dropped**, and the run
proceeds.

This design freezes **what** we validate, **when**, **how hard we fail**, and **what
we deliberately refuse to do** — before any implementation PR.

---

## 1. Problem statement

### 1.1 User-visible failure mode (the real bug)

A user can currently:

1. Mis-spell a distribution type or Bounds key,
2. Drop required prior parameters,
3. Put a dead/unknown knobs under Archiver / Cleanup / Bounds,
4. Watch Jarvis2 start Redis, spawn Workers, run for minutes/hours,
5. Get results that do not match the intent of the card — with **little or no signal**
   that the card was wrong.

That is worse than a hard crash: it erodes trust in the YAML contract.

### 1.2 As-built validation map (V2)

| Layer | What happens today | Class |
|-------|-------------------|--------|
| `load_task_yaml` | Reject top-level `Runtime`; **whitelist** `EnvReqs.V2` keys | Good hard fail |
| `normalize_*` | Illegal enums / ints → **silent default** | Silent wrong |
| `load_variables` | Empty `name` **dropped**; bad `type` deferred; params unchecked | Silent / late |
| Nested Bounds | Unknown constructor keys **warn** (if logger exists); meta typos often become defaults | Soft |
| `Distributor.set_method` | Unknown Method → clear error | Good, but late vs ideal |
| Empty Method | Falls through to weak path | Bad message / late |
| Full-task jsonschema | **None** | Gap |
| Offline dry-run | **None** (D8 `--validate` parked) | Gap |

### 1.3 What V1 actually bought us

V1 pipeline (scan modes):

```
load → dependency check → set_method(Method) → set_schema(sampler.schema)
  → ConfigValidator.validate_yaml() → exit 2 on ValidationError
```

Strengths:

- **Early** (before heavy runtime).
- **Method-selected schema** (not one mega-schema forever).
- **Variables** constrained with `if/then` per distribution type.
- **User-facing** error text + non-zero exit.

Weaknesses we must not copy blindly:

- Schemas force heavy `EnvReqs` / `Calculators` shapes that V2 does not require the same way.
- Dynesty/MultiNest Bounds schemas lag V2’s official-dynesty 3.x surface.
- `sys.exit(2)` inside the validator couples process death to library use.
- Full-document `additionalProperties` politics made Operas-only / light cards painful
  without inject-empty hacks (`ConfigLoader._normalize_optional_sections`).

---

## 2. Goals and non-goals

### 2.1 Goals

1. **Contract gate before side effects**: no Redis connect, no Worker spawn, no sample
   loop until the task card passes the active validation profile.
2. **Actionable diagnostics**: every error names a YAML path, expected shape, and
   (when useful) the bad value / “did you mean”.
3. **One truth for run and dry-run**: `Jarvis2 run` and `Jarvis2 … --validate` share
   the same pure function; only I/O and process exit differ.
4. **Method-aware Sampling contracts**: Bounds/Variables rules depend on
   `Sampling.Method`, sourced from the same registry spirit as `Distributor`.
5. **End silent “science-affecting” coercion** for knobs that change results or
   topology (priors, Method, Bounds that set stop criteria, unknown keys in
   contracted blocks).
6. **Incremental ship**: P0 stops the worst silent bugs without waiting for D8 Agent
   envelope or a complete Calculus of every YAML key.

### 2.2 Non-goals (explicit)

| Non-goal | Why |
|----------|-----|
| Clone V1 `*_schema.json` wholesale | Wrong EnvReqs surface; stale Bounds; false failures |
| Full semantic physics validation | Expressions, module install, ROOT, etc. stay later / smoke |
| Replace `YAML_REFERENCE` with schemas alone | Docs remain human source; schemas enforce a subset |
| Unpark entire D8 Agent Bridge in this milestone | Validation **kernel** first; JSON envelope can wrap later |
| Runtime dependency (Redis/Portal/conda) as part of validate | Validate must work offline on the card |
| Third-party plugin schema discovery | In-tree methods only; plugins later |
| Auto-fix user YAML | Report only; never rewrite files on disk |

---

## 3. Design principles (reflect before rules)

These are the load-bearing judgments. Implementation must not violate them without
revising this document.

### P1 — YAML is a contract, not a hint

If a key is documented as meaningful, an unknown or illegal value is a **user error**,
not a soft default — unless the key is explicitly documented as “optional with default
when omitted” (**omitted ≠ illegal**).

- Omitted optional key → apply default (OK, quiet).
- Present-but-illegal key → **error** (or warning only when listed as soft in §5).
- Unknown key in a **closed** block → **error** (P0 closed set in §6).

### P2 — Fail before side effects; never only in `--validate`

If validation is optional on the run path, people skip it and we re-create the bug.
`--validate` is a convenience mirror, not a second policy.

### P3 — Layer by failure cost, not by vanity completeness

Validate first what, if wrong, produces **silent wrong science** or **wasted cluster
time**. Defer nice-to-have inventory (module name existence on disk, expression
compile against full observable set) unless cheap and pure.

### P4 — Normalization and validation are different phases

Today `normalize_*` both **cleans** and **hides errors**. Split:

1. **Parse / merge** (`load_task_yaml`) — structural load, path stamps, EnvReqs merge.
2. **Validate** — produce `ValidationReport` (errors + warnings).
3. **Normalize for runtime** — only on a config that already passed (or that only has
   warnings under non-strict).

Coercion that changes user intent must either **fail** or emit a **warning that names
the rejected raw value** (YAML_REFERENCE A.10). Prefer fail for science knobs.

### P5 — Method contracts live with methods

Bounds rules for Dynesty must not rot in a global file nobody updates when the
sampler changes. Registration (or an adjacent contract module) owns the method’s
allowed Bounds surface; templates under `project_template/bin/sampling/` are the
human twin of that contract.

### P6 — Severity is product policy, not taste

| Severity | Effect on `run` | Effect on `--validate` |
|----------|-----------------|------------------------|
| **error** | abort (exit 2) | `ok=false` |
| **warning** | run continues; log once at gate | listed; exit 0 unless `--strict` |
| **strict-warning** | like warning; with `--strict` or env, promote to error | same |

Default run mode: **errors block; warnings do not**. Power users / CI / Agent use
`--strict` to treat unknown keys and residual coercions as failures.

### P7 — Pure core, impure edges

`validate_task_config(config) -> ValidationReport` must be:

- free of Redis, subprocess, network, cwd mutation;
- free of `sys.exit`;
- deterministic given the same config dict.

CLI/core translate the report into logs and exit codes.

### P8 — Compatibility without infinite aliases

Accept a **short, documented** alias list (e.g. `n_live` → `nlive`, `Seed`/`seed`)
where V1/V2 already diverge. Do **not** invent fuzzy matching for every typo.
Unknown keys fail closed in contracted blocks.

### P9 — check_modules is a mode, not a free pass from all rules

`Sampling.mode: check_modules` may omit Method or ignore Method if present and unused
— but must not hard-require Nested Bounds. Rules for this mode are explicit (§7.3),
not “validation off”.

### P10 — Prefer thin declarative contracts over giant draft-07 trees

Where structure is regular (Variables distributions, enum sets), JSON Schema fragments
are fine. Where logic is mode-dependent (MultiNest forces static nested), **Python
contract functions** next to the sampler are clearer and easier to test. Hybrid is OK;
“jsonschema everywhere” is not a goal.

---

## 4. Alternatives considered

### A. Port V1 `ConfigValidator` + full method schemas

| Pros | Cons |
|------|------|
| Proven UX | Wrong required top-level blocks for V2 |
| | Stale Nested Bounds |
| | Process exit coupling |
| | Large false-positive blast radius on day one |

**Verdict**: reuse the *idea* (early, method-scoped, hard fail), not the *files*.

### B. Only improve docs / templates

| Pros | Cons |
|------|------|
| Zero code risk | Does not stop silent wrong runs |

**Verdict**: insufficient alone. Templates stay complementary.

### C. Validate only inside each sampler `initialize()`

| Pros | Cons |
|------|------|
| Natural ownership | Happens after bootstrap / some side effects |
| | Inconsistent messages |
| | Easy to forget per sampler |

**Verdict**: samplers may keep deep checks; **shared gate still required earlier**.

### D. Full jsonschema for entire task document

| Pros | Cons |
|------|------|
| One file “truth” | Operas-only / evolving EnvReqs.V2 / Calculator Portal shapes |
| | High churn; schema becomes the bottleneck for every feature |

**Verdict**: rejected as the sole approach. Closed sub-schemas per block yes;
mega-schema no for D14.

### E. Hybrid layered gate (chosen)

Early pure validator + method contracts + closed knobs + shared with `--validate`.
See §5–§8.

---

## 5. Architecture

### 5.1 Placement in the control flow

```
CLI / core
  │
  ├─ load_task_yaml(path)              # merge EnvReqs.V2, stamp paths, build internal Runtime
  │
  ├─ validate_task_config(config, *,   # NEW pure gate
  │       profile="run"|"validate",
  │       strict=False)
  │       → ValidationReport
  │
  ├─ if report.has_errors: log + exit 2
  ├─ if report.warnings: log once (gate logger)
  │
  └─ bootstrap_distributed_runtime…    # Redis, Workers, sampler loop — only if clean of errors
```

**Light modes** (`--plot`, pure `--monitor` without task, etc.) keep their existing
“load without full scan validate” behavior unless they already load a full task for
execution. Any path that **runs a scan or check-modules smoke that uses the task card**
must call the gate.

### 5.2 Module layout (proposed)

```
jarvishep2/
  task_validation.py          # ValidationIssue, ValidationReport, validate_task_config
  contracts/
    __init__.py
    common.py                 # Variables, Scan.sample_directory, shared enums
    envreqs_v2.py             # already partly in runtime_config; unknown keys policy
    archiver_cleanup.py       # closed blocks
    methods/
      __init__.py             # resolve contract by Method name
      variables.py            # distribution type → param schema
      dynesty.py
      multinest.py
      bridson.py
      randoms.py
      grid.py
      csv.py
      adaptive_level_set.py
      mcmc_base.py            # when MCMC family needs shared Bounds
  card/schema/                # OPTIONAL JSON fragments (if used)
    variables_distributions.json
    bounds_dynesty.json
    …
```

Names can be flattened (`task_validation` + `Sampling/contracts.py`) if a new package
feels heavy; **logical** separation matters more than directory aesthetics.

### 5.3 Data model

```python
@dataclass(frozen=True)
class ValidationIssue:
    level: Literal["error", "warning"]
    code: str          # e.g. "JV2-VAR-001" — stable for tests/Agent
    path: str          # dotted/JSON-pointer-ish: "Sampling.Variables[0].distribution.type"
    message: str       # human, single sentence + expected
    hint: str | None = None

@dataclass
class ValidationReport:
    issues: list[ValidationIssue]
    # convenience
    def errors(self) -> list[ValidationIssue]: ...
    def warnings(self) -> list[ValidationIssue]: ...
    @property
    def ok(self) -> bool:  # no errors (warnings allowed)
        ...
```

Exit policy:

- `run`: errors → exit **2** (config/usage family, aligned with V1).
- `--validate`: errors → exit 2, `ok=false` when JSON exists; warnings-only → exit 0
  unless `--strict`.

### 5.4 Registration of method contracts

Extend the Distributor registry spirit (no requirement to break existing API in P0):

```text
Method name → Contract
  Contract.validate_sampling(sampling_block, *, config) -> list[ValidationIssue]
  Contract.allowed_bounds_keys / closed?
```

P0 may use a simple `METHOD_CONTRACTS: dict[str, Callable]` in `contracts/methods`
without changing `Distributor.register` signature. P1 optionally adds
`contract=` to `register()` so new samplers cannot ship without a contract hook
(default: “Variables + Method known only”).

### 5.5 Relationship to D8 (parked)

| D8 piece | This design |
|----------|-------------|
| `--validate --json` envelope | **Consumes** `ValidationReport`; does not define kernel |
| Diagnostics codes `JV2-###` | Introduced here; D8 lists them in envelope |
| Strict unknown-key warnings | **P0/P1** policy here; D8 only exposes flags |
| Module inventory in validate JSON | **Out of D14 P0** (needs workflow load); D8 can add later |

**Decision**: D14 delivers the kernel + human CLI validate. D8 remains parked and
becomes a thin adapter when unparked.

### 5.6 Relationship to nested “ignore unknown kwargs”

Today Dynesty logs a warning for unknown constructor keys **if a logger is set**.
That is too late and easy to miss.

**Decision**:

- Unknown keys under `Sampling.Bounds` for Nested methods are **ValidationIssue**
  at the gate (error by default for keys not in the allow-list ∪ documented aliases).
- Runtime `_filter_known_kwargs` remains a last-line defense, not the primary UX.

---

## 6. What is closed vs open (scope of keys)

### 6.1 Closed blocks (unknown key = error)

These are user-facing and small enough to close:

| Block | Notes |
|-------|--------|
| `EnvReqs.V2` | Already closed (keep) |
| `EnvReqs.V2.worker` (mapping form) | Close after documenting allowed keys |
| `EnvReqs.V2.factory` / `redis` | Close known leaves |
| `Calculators.Archiver` | Close to `ARCHIVER_DEFAULTS` keys (+ documented aliases) |
| `Calculators.Cleanup` | Close to cleanup keys |
| `Scan.sample_directory` | Close to known sample-directory keys |
| `Sampling.Variables[]` | Only `name`, `description`, `distribution` |
| `Sampling.Variables[].distribution` | `type`, `parameters` only |
| `parameters` per type | Exact required set; `additionalProperties` false |
| `Sampling.Bounds` (per Method) | Allow-list for that method |

### 6.2 Open or deferred blocks (unknown key = ignore or warning only in strict)

| Block | Why deferred |
|-------|----------------|
| `Calculators.Modules[]` large surface | Portal/modes; high churn; wrong layer for P0 |
| `Operas.Modules[]` | Similar; validate `operator` / `call_mode` enums only in P1 |
| `LibDeps` | Mostly install-time |
| Top-level unknown keys | `additionalProperties` true historically; warn in strict only |
| Expression strings | Semantic; compile later |

**Reflection**: closing Calculators too early creates false “validation complete”
confidence while still allowing broken modules. Prefer **deep but narrow** over
**shallow and wide**.

### 6.3 Science-critical vs operational knobs

| Class | Examples | Policy |
|-------|----------|--------|
| Science / topology | Method, Variables, Bounds stop criteria, selection expr present-if-used | **error** on bad/unknown |
| Operational scale | workers, batch_size, archiver batch | Present-but-illegal → **error**; omitted → default |
| Cosmetic | print_progress | Soft defaults OK |

---

## 7. Validation layers (content)

### L0 — Load (existing)

- File exists; top-level mapping.
- No top-level `Runtime`.
- `EnvReqs.V2` whitelist.
- Defaults merge.

Failures stay `ValueError` / `FileNotFoundError` (or folded into report in P1 for
uniform CLI — optional).

### L1 — Task skeleton (P0)

Always (except pure non-task CLI):

1. `Sampling` is a mapping (if scan path).
2. `Sampling.Method` handling:
   - **run path**: required unless `mode in {check_modules, check-modules}`.
   - Value must be in `Distributor.available_methods()` when required.
   - Empty string / whitespace → error (not “default sampler”).
3. `Sampling.Variables` (when method needs variables — all current methods except CSV
   fixed-set style):
   - Non-empty list after filtering? **No silent filter**: invalid entries are errors,
     not drops.
   - Each item: `name` non-empty string; `distribution.type` in allowed set;
     `parameters` complete for type (see L1b).
4. CSV method: require `Sampling` data path keys per existing sampler contract.
5. Bridson: `length` policy unified (required vs default) — pick one in Key Decisions;
   validate accordingly.

#### L1b — Distribution parameter tables (align V1 + V2 templates)

| type | required parameters | forbidden extras |
|------|---------------------|------------------|
| `Flat` | `min`, `max` | yes closed |
| `Log` | `min`, `max` | yes |
| `Logit` | `location`, `scale` | yes |
| `Normal` | `mean`, `stddev` | yes |
| `Log-Normal` | `mean`, `stddev` | yes |

Optional Bridson extension: `length` on Flat/Log parameters — method contract may
require it when Method is Bridson.

Numeric sanity (P0 minimal): `min < max` where both present; `stddev > 0` if cheap.

**Case sensitivity**: types are **exact** (`Flat` not `flat`). Message lists allowed
values. (Reflection: lowercasing silently would re-create silent coercion.)

### L2 — Method Bounds contracts (P0 for Nested; P1 for others)

#### Dynesty (static + dynamic)

Allow-list includes (names as in templates + code aliases already supported):

- Meta: `nlive`/`n_live`, `rseed`/`seed`/`Seed`, `dynamic`/`Dynamic`,
  `dlogz`/`dlogz_init`, `run_nested`, `sampler`/`constructor` (+ case variants already in code — **document the canonical set and accept aliases explicitly**).
- Nested `run_nested` keys: intersect with dynesty allow-list already in
  `dynesty_sampler.py`.
- Nested `sampler`/`constructor` keys: same.

Rules:

- `nlive` if present: integer ≥ 2.
- `dynamic` if present: bool.
- MultiNest contract: `dynamic` must be false or absent; if true → **error**
  (copy-paste Dynesty card).

Unknown Bounds key → **error** (P0 Nested).

#### MultiNest

Same static Nested allow-list; force static semantics in contract (mirror sampler).

#### Bridson / Random / Grid / CSV / AdaptiveLevelSet

P0: L1 only (+ method-specific required keys already enforced late today, pulled
forward). P1: close Bounds/Point number surfaces and unify naming messages.

### L3 — Operational blocks (P0)

| Check | Severity |
|-------|----------|
| `EnvReqs.V2` unknown keys | error (existing) |
| `Archiver` / `Cleanup` unknown keys | **error** (new) |
| Illegal enum (`Archiver.mode`, cleanup strategy, `sample_artifacts`) | **error** (stop silent default) |
| Non-numeric `workers` / `batch_size` when present | **error** (stop silent default) |
| Dead keys (`execution.input[].save`, internal-only if user-visible) | **warning** P0; strict→error P1 |

### L4 — Strict extras (P1 / `--strict`)

- Unknown keys under top-level or open modules.
- Residual coercions if any remain.
- `Operas.Modules[].call_mode` not in `{call, acall}` → error.
- Optional: expression compile smoke with declared symbols only.

### L5 — Out of scope for D14

- Module binaries exist on disk.
- Full Portal I/O type graph.
- Remote Redis reachability (runtime concern).
- Physics correctness of likelihood expressions.

---

## 8. CLI and logging UX

### 8.1 Run path

On errors:

```
Jarvis-HEP.Config  Config validation failed (2 errors, 1 warning):
  [error] JV2-VAR-002  Sampling.Variables[0].distribution.type
          'flat' is not supported; expected: Flat, Log, Logit, Normal, Log-Normal
  [error] JV2-BND-010  Sampling.Bounds.nlive
          expected integer ≥ 2, got 'auto'
  [warning] JV2-DEAD-001  Calculators.Modules[0].execution.input[0].save
          key is ignored in V2 (dead key)
```

Then exit 2 **before** Redis banner spam if possible (order: load → validate → logo
optional; prefer validate before expensive setup).

### 8.2 `Jarvis2 <task.yaml> --validate [--strict] [--json]`

P0 human output is enough. `--json` may wait for D8 envelope **or** ship a minimal
`{"ok", "issues":[...]}` without full agent inventory — prefer small stable JSON early
if Agent needs it; otherwise human-only until D8.

**Must not**: start Redis, spawn workers, write `run_state.json`, create SAMPLE trees
(read-only on the config dict).

### 8.3 Codes

Reserve prefixes:

| Prefix | Domain |
|--------|--------|
| `JV2-LOAD-*` | load/merge |
| `JV2-MTH-*` | Method |
| `JV2-VAR-*` | Variables / distributions |
| `JV2-BND-*` | Bounds |
| `JV2-ENV-*` | EnvReqs.V2 |
| `JV2-ARC-*` | Archiver/Cleanup |
| `JV2-DEAD-*` | dead keys |
| `JV2-OPR-*` | Operas |

Stable codes are a feature for tests and Agent; do not renumber casually.

---

## 9. Interaction with existing code (migration)

### 9.1 `load_variables`

Today: skip bad entries; default type `Flat`.

**After**:

- Gate already rejected bad entries → `load_variables` may assume well-formed list
  **or** re-check and raise (defense in depth).
- Do not default unknown type to Flat.
- Do not skip nameless entries silently.

### 9.2 `runtime_config.normalize_*`

**Change policy** for values that are **present**:

- Illegal enum / non-coercible number → do not substitute default; either raise or
  rely on pre-gate (prefer gate-only raise path so normalize stays pure for valid
  configs).

**Keep** defaults for **omitted** keys.

Implementation pattern:

```text
validate (raw merged config)
  → on success → normalize (trusted)
```

Avoid double-normalization surprises: validate should see the same structure the user
wrote (post-merge EnvReqs), not a half-normalized ghost.

### 9.3 Nested sampler kwargs filter

Keep as belt-and-suspenders; gate owns UX.

### 9.4 Tests that relied on silent coercion

Expect some tests to need cards fixed or to assert new errors. That is desired
signal, not breakage of science.

---

## 10. Backward compatibility and blast radius

### 10.1 Cards that were “working” via typos

They will start failing. That is **correct**. Provide:

- Clear message + allowed keys.
- Template YAMLs already under `project_template/bin/sampling/` as golden goods.
- Optional short migration note in YAML_REFERENCE §13.

### 10.2 Alias list (explicit, versioned)

Documented aliases accepted without warning (P0):

| Alias | Canonical |
|-------|-----------|
| `n_live` | `nlive` |
| `Seed` / `seed` (Sampling or Bounds) | as today |
| `Dynamic` | `dynamic` |
| `workers` vs scalar `worker` | already handled |

Aliases that mask typos (`live`, `NLive`, `nLive`) are **not** accepted.

### 10.3 Feature flag / escape hatch

**Default: validation on.**

Optional escape (debate — default **off** for production honesty):

- `JARVIS2_SKIP_CONFIG_VALIDATION=1` for emergency only; log loud warning.
- Not advertised in quickstart.

Reflection: an easy skip re-creates the bug for CI. Prefer no skip, or skip only in
unit tests via API `validate=False` on internal helpers — not user CLI.

---

## 11. Testing strategy

| Suite | Content |
|-------|---------|
| Unit | Each code path for Variables tables; Nested Bounds unknown key; Archiver enum; empty Method |
| Golden | All `project_template/bin/sampling/*.yaml` fragments merged into minimal task shells → must pass |
| Negative | Mutated goldens (typo Method, `flat`, bad nlive) → expect specific `JV2-*` codes |
| Integration | `core` bootstrap aborts before Redis mock when validation fails (spy) |
| Non-regression | Existing parity/check-modules projects still pass |

No network. No real dynesty runs required for P0 validation tests.

---

## 12. Documentation updates (with implementation, not before)

When code lands:

1. `YAML_REFERENCE_2.0.md` §13 — replace “no schema layer” with gate description.
2. `components/config_schema.md` — document `task_validation` phase.
3. `V2_DISTRIBUTED_PLAN.md` — D14 work packages (if accepted).
4. Manual TeX (later) — “Validation successful” V2 wording.

This design file is the plan; avoid drive-by doc rewrites until implementation shape
is stable.

---

## 13. Risks and reflections (honest)

### R1 — Over-validation freezes iteration

If Bounds allow-lists lag dynesty features, advanced users pad against us.

**Mitigation**: allow-list is derived from the same frozensets in `dynesty_sampler.py`
(single source of truth), not a second hand-maintained JSON forever. Advanced
pass-through keys stay allowed if in that set; truly unknown still fail.

### R2 — Under-validation leaves Calculators free

Accepted for P0. Document clearly: “validated surface” vs “not yet closed”.

### R3 — Strict mode ignored

Make CI examples use `--strict`. Agent (D8) defaults strict when unparked.

### R4 — Double ownership of rules (schema + Python)

Prefer Python contracts calling shared frozensets for Nested; optional JSON only for
Variables if it helps external tools. Do not maintain two divergent lists.

### R5 — Performance

Negligible vs Redis. Report construction is fine.

### R6 — validate order vs logging init

If validate runs before logger sinks exist, print to stderr with the same format.
Do not drop errors.

### R7 — Cultural: “warnings nobody reads”

Science errors must be **errors**, not warnings. Warnings are for dead keys and
future deprecations only.

---

## 14. Key Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | Hybrid layered gate; not full V1 schema import | Correct V2 surface; lower false positives |
| D2 | Pure `validate_task_config` + report; no `sys.exit` inside | Testable; Agent-ready |
| D3 | Gate on every scan/smoke path; `--validate` is a mirror | Prevents optional safety |
| D4 | Closed blocks for EnvReqs.V2, Archiver/Cleanup, Variables, Nested Bounds | Highest silent-wrong ROI |
| D5 | Present-but-illegal operational knobs → **error**, not silent default | Closes A.10 for those knobs |
| D6 | Unknown Nested Bounds keys → **error** at gate | Prefer fail over warn-later |
| D7 | Distribution type names exact-case | Avoid silent Flat coercion |
| D8 | D14 kernel independent of D8 unpark | Ship user safety without Agent scope |
| D9 | Method contracts adjacent to methods; allow-lists shared with runtime Nested code | Single source of truth |
| D10 | Bridson `length`: **required** when Method is Bridson (unify; drop lenient default as source of truth) | Removes dual-path inconsistency (A.11) |
| D11 | No user-facing skip flag by default | Honesty over convenience |
| D12 | check_modules: Method optional; if present and unknown → still error (fail closed) unless we explicitly ignore Method in that mode — **Decision: ignore Method when mode is check_modules** (document; avoids copy-paste traps) | Fixes A.12 coupling without disabling other rules |

---

## 15. Open questions (need maintainer if contested)

These have a **recommended default** already in Key Decisions; escalate only if you
disagree.

1. **check_modules + Method** — Recommended: **ignore Method** when
   `mode: check_modules`. Alternative: require a valid Method always.
2. **`--json` in D14** — Recommended: human-only P0; minimal JSON optional if easy.
3. **Emergency skip env** — Recommended: **no**. Alternative: `JARVIS2_SKIP_…` with
   loud warning.
4. **JSON Schema files on disk** — Recommended: Python-first; add fragments later if
   external tooling needs them.
5. **Promote all Nested unknown keys to error immediately** — Recommended: **yes** for
   Nested Bounds. Alternative: warning for one release.

---

## 16. Work packages / PR Plan

### PR1 — Kernel + L1 Variables/Method (D14.1)

- **Title**: `D14.1: task_validation kernel + Variables/Method gate`
- **Files**: `jarvishep2/task_validation.py`, `contracts/…/variables.py`,
  `core.py` hook, `client.py` optional flag stub, tests
- **Deps**: none
- **Desc**: Report model; Method registry check; Variables closed shape; run-path
  abort; golden + negative tests. No Archiver/Nested yet if need thin PR — but
  Method/Variables alone already high value.

### PR2 — Operational closed knobs (D14.2)

- **Title**: `D14.2: Archiver/Cleanup/EnvReqs illegal values hard-fail`
- **Files**: `runtime_config.py` policy shift, `task_validation` L3, tests
- **Deps**: PR1
- **Desc**: Stop silent enum/int coercion for present keys; unknown Archiver/Cleanup
  keys error.

### PR3 — Nested Bounds contracts (D14.3)

- **Title**: `D14.3: Dynesty/MultiNest Bounds allow-list at gate`
- **Files**: contracts dynesty/multinest; share frozensets with `dynesty_sampler.py`;
  template-aligned tests
- **Deps**: PR1
- **Desc**: Unknown Bounds keys error; nlive type; MultiNest rejects dynamic true.

### PR4 — CLI `--validate` + Bridson/others pull-forward (D14.4)

- **Title**: `D14.4: Jarvis2 --validate + remaining method required keys`
- **Files**: `client.py`, docs §13, Bridson length policy, CSV path presence
- **Deps**: PR1–3
- **Desc**: Dry-run UX; unify Bridson length; document surface.

### PR5 — Strict mode + dead keys + YAML_REFERENCE (D14.5)

- **Title**: `D14.5: --strict, dead-key warnings, docs`
- **Files**: docs, call_mode validation, optional minimal JSON
- **Deps**: PR4
- **Desc**: A.2/A.3/A.10 remainder; prepare D8 adapter surface.

### Optional later

- D8.1 envelope wrapping the same report.
- Calculator/Operas deep contracts.
- TeX manual validation chapter.

---

## 17. Acceptance criteria (D14 done)

1. Mutating a golden Nested template (`type: flat`, `Bounds.live: 50`,
   `Archiver.mode: turbo`) fails **before** Redis with coded paths.
2. Unchanged golden templates and parity projects still run.
3. `validate_task_config` unit-tested without network.
4. `YAML_REFERENCE` §13 no longer claims “no schema validation layer”.
5. No `sys.exit` inside the pure validator.
6. check_modules tasks with leftover junk Method names do not fail solely due to
   Method (per D12); other L1 rules still apply where relevant.

---

## 18. Implementation readiness checklist (when landing starts)

- [ ] Maintainer agrees Key Decisions D1–D12 (or records overrides).
- [ ] Open questions §15 closed or accepted as recommended.
- [ ] Identify single source of truth for Nested allow-lists (export from
      `dynesty_sampler` or move to `contracts` imported by sampler).
- [ ] List all in-tree Method names for contract stubs.
- [ ] PR1 branch + tests only — no doc flood until PR4/5.

---

## 19. Summary

V2 should **re-learn V1’s discipline** (early, method-aware, hard fail, clear text)
and **reject V1’s baggage** (monolithic outdated schemas, exit-inside-library,
forced EnvReqs shape).

The product rule is simple:

> **If the card is wrong in a way that would change the run, refuse to run.**

D14 builds that gate as a pure, layered contract so users and future Agent tooling
share one source of truth — without waiting for the full D8 bridge.
