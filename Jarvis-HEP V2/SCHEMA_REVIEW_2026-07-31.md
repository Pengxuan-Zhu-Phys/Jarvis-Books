# Review — Task-Card JSON Schema Subsystem (2026-07-31)

**Baseline**: `jarvishep2` branch `jarvis2` @ `43b1122`; working tree clean.
**Focus**: the newly introduced `jarvishep2/schema/` subsystem (38 schema files + manifest
+ `task_schema.py` loader/dispatcher), added across `a05d78f` → `43b1122`, plus a
confirmation pass on D13.11.
**Method**: read the loader and every schema file; verified each finding by executing
`validate_task_config` against constructed cards rather than by inspection.

**Verdict on D13.11 (previous review's fixes): landed and correct.** The `jarvis_install.json`
control file, the `reinstall_epoch` fan-out (`can_reuse_install` requires *both* a
fingerprint match *and* `stamp_epoch >= install_epoch`, so one flip invalidates every pack
independently), the unmatched-feedback warning in `feedback_sampler.py`, Lua-atomic slot
release, and `os.replace` in the CSV writer are all in, with
`test_reinstall_epoch_fans_out_to_every_pack_once` guarding the part that was easy to get
wrong. Nothing to add there.

**Verdict on the schema subsystem: the plumbing is sound; the schemas are empty.** The
loader, registry, manifest dispatch, and diagnostic rendering are well built — 38 files
load, `$id`s are unique and resolvable, no manifest/disk drift, Portal alignment is
test-guarded. But the per-method schemas that the whole structure exists to dispatch to
**contain no per-method constraints**, so for 14 of 17 methods the layer validates nothing
a user could get wrong. Two structural holes below.

---

## 1. The per-method split currently carries no per-method content

`manifest.json` maps 17 methods to 17 schema files, `task_schema.py` dispatches by
`Sampling.Method`, and `test_manifest_gives_each_sampling_method_its_own_schema` asserts
each method has its own schema. That all works. But here is `dram.json` in full:

```json
{"$schema":"…","$id":"…/methods/dram.json",
 "allOf":[{"$ref":"…/methods/variables.json"},
          {"properties":{"Method":{"const":"DRAM"}}}]}
```

Every other method file is the same shape. Two consequences:

**The `Method: const` assertion is tautological.** The dispatcher selects `dram.json`
*because* `Sampling.Method == "DRAM"`, then validates that `Method == "DRAM"`. It cannot
fail. (Verified: the const in every file matches its manifest key, so no card ever trips
it.)

**`Bounds` — the block that actually differs per method — is unconstrained everywhere.**
`core/sampling.json` types it as bare `{"type": "object"}`, and **0 of 17** method schemas
say anything about it. Measured:

| method schema | constrains `Bounds`? | content beyond const + shared `required: [Variables]` |
|---|---|---|
| Bridson / Random / CSV | no | `required` / `anyOf` (real, useful) |
| **the other 14** (DRAM, MCMC, AM, AMMCMC, Dynesty, MultiNest, PT, PTMCMC, PTEnsemble, DEMCMC, Ensemble*, Grid, AdaptiveBridson) | no | **nothing** |

What that costs a user, executed against the current tree:

```
DRAM  Bounds:{chains:4,  steps:100}    → 放行  (correct card)
DRAM  Bounds:{chainz:4,  stepz:100}    → 放行  ← every key misspelled
DRAM  Bounds:{chains:"四", steps:100}   → 放行  ← string where an int is required
```

The scan then runs with `chains` silently falling back to its default. The user asked for
4 chains, gets 1, and the only evidence is a number in a log line they have no reason to
read. This is the same class of failure as the install-reuse bug in the last review:
**the run looks completely normal and the results are quietly not what was requested.**

Coverage is also uneven *between* families, which makes it unpredictable: the legacy
contracts layer does check nested-sampler `Bounds` (`Dynesty Bounds:{nlive:500, walkz:9}`
→ `JV2-BND-001`), so a Dynesty typo is caught while the identical mistake in DRAM is not.
A user cannot tell which of their mistakes the validator will find.

**Recommendation**: put the per-method `Bounds` vocabulary into the per-method schemas —
that is the one job this split exists to do. Each needs `Bounds.properties` with types and
`additionalProperties: false` (the keys are already documented in YAML_REFERENCE §6.11–6.13,
so this is transcription, not design). Do the MCMC family first: it is the family with no
contract-layer coverage at all. Note the shipped schemas do use `x-jarvis-example`, so the
diagnostic machinery for good messages is already there and unused for these keys.

Also: `test_manifest_gives_each_sampling_method_its_own_schema` asserts only that the 17
*URIs* are distinct — it passes trivially while all 17 *contents* are equivalent. It reads
like a guard for this design intent but does not enforce it. Worth strengthening to
"each method schema constrains at least one method-specific key" once §1 lands.

## 2. Unknown sampling method fails **open**; unknown I/O format fails **closed**

`_validate_selected_io` emits `JV2-SCH-002` for an unregistered format:

```python
schema_uri = formats.get(kind)
if schema_uri is None:
    issues.append(issue("error", "JV2-SCH-002", …))   # fail closed ✓
```

The sampling path, given the same situation, does nothing:

```python
schema_uri = manifest["sampling_methods"].get(method)
if schema_uri is not None:
    issues.extend(_issues_for(...))                   # else: silently no validation
```

For a *typo'd* method this is harmless — the legacy `JV2-MTH-003` check catches it against
the Distributor registry (verified: `Method: dram` → clear error listing all 17 valid
names, exit code 2). The hole opens for a method that **is** registered but has no manifest
entry — i.e. the normal act of adding a sampler. Simulated by registering one:

```
Distributor.register("MyNewSampler", …)
card Sampling.Bounds = {chains: "这里本该是整数", 完全瞎写的键: {...}}
→ errors: 无   warnings: 无      ← zero diagnostics, silently unvalidated
```

And nothing fails in CI when that happens: `test_manifest_matches_builtin_portal_formats`
guards the **Portal ↔ manifest** boundary, but there is **no equivalent test for
Distributor ↔ manifest**. I checked all four boundaries by script — manifest↔disk,
`$id`↔manifest, Distributor↔manifest, Portal↔manifest — and all four are clean *today*;
three of them are held only by whoever last edited by hand.

**Recommendation**: (a) mirror the IO behavior — emit a diagnostic when a registered
method has no schema, so the gap announces itself; (b) add the missing
Distributor↔manifest test alongside the Portal one; (c) consider asserting
manifest↔directory too (cheap, and it is the one that turns into a startup crash rather
than a silent skip).

## 3. Minor: root schema `$id` does not follow the file-path convention

37 of 38 files have `$id` ending in their path relative to `schema/`. The exception is
`task-card-v2.schema.json`, whose `$id` is `…/schema/v2/task-card.json`. It resolves
correctly (the manifest's `root` points at the `$id`, not the filename), so nothing is
broken — but it is the kind of inconsistency that costs someone an hour when a future
loader change assumes path↔`$id` symmetry. Either rename the file or the `$id`.

---

## 4. What is genuinely good here

Worth recording so it is not refactored away by accident:

- **Local-only registry.** Schemas are loaded from disk into a `referencing.Registry`
  keyed by `$id`; the `https://jarvis-hep.org/...` URIs are identifiers, never fetched.
  No network dependency in the validation path.
- **`check_schema` on every file at load.** A malformed schema fails loudly at startup
  rather than mis-validating cards.
- **Diagnostics are actionable.** `_schema_error_guidance` turns each jsonschema
  validator into an editable instruction plus a YAML example, and
  `test_each_reported_error_has_a_correction_suggestion` enforces that every error carries
  one. This is the part users will feel, and it is done right.
- **Whitespace normalization is dispatch-consistent** (`Method`/`type` are `.strip()`ed
  before lookup *and* before validation), with a test pinning it to runtime behavior.

---

## 5. Action items

Registered as **D13.14** in [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md):

| # | Item | Severity |
|---|---|---|
| 1 | Give each method schema its real `Bounds` vocabulary (`properties` + types + `additionalProperties: false`); MCMC family first | **high** — silent misconfiguration today |
| 2 | Fail-closed for registered-method-without-schema + Distributor↔manifest test (and manifest↔directory) | medium |
| 3 | Strengthen `test_manifest_gives_each_sampling_method_its_own_schema` to require method-specific content | low |
| 4 | Root schema `$id`/filename consistency | cosmetic |

**Separately filed as D13.15** — the plan header has recorded *"608 passed with 8
pre-existing failures"* since 2026-07-29. That was not re-verified in this review (the
standing instruction is not to re-run the suite for review work), but a persistent set of
known-failing tests should not stay a footnote: it makes Protocol §5 (*a WP is not done
with failing tests*) unenforceable, because the next agent cannot separate its own
breakage from the background. Fix them or `xfail` them with a stated reason, so that
green means green.
