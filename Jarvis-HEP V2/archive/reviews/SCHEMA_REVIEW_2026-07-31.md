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

**Verdict on the schema subsystem: the plumbing is sound; two real gaps.** The loader,
registry, manifest dispatch, and diagnostic rendering are well built — 38 files load,
`$id`s are unique and resolvable, no manifest/disk drift.

> **⚠ Scoped after maintainer feedback (2026-07-31).** This review first reported "0 of 17
> method schemas constrain `Bounds`" and recommended doing the MCMC family first. Two
> corrections: **(a) the MCMC family is not migrated yet**, so its key vocabulary is not
> settled and writing strict schemas for it now would pin an interface that is still
> moving — the requirement is only that the **already-available** methods are right;
> **(b) I/O formats are not a peer of sampling methods** — their authority is the
> **Jarvis-Portal source**, not HEP's registry, which changes what §2's finding actually
> is. Both sections below are rewritten accordingly; the raw measurements are unchanged.

---

## 1. Among the migrated methods, only `AdaptiveBridson` is unguarded

Every method file has the same shape — a `$ref` to the shared `variables.json` plus
`{"Method": {"const": "<its own name>"}}`. That const is **tautological**: the dispatcher
selects `dram.json` *because* `Method == "DRAM"`, then asserts `Method == "DRAM"`. It can
never fail. And `core/sampling.json` types `Bounds` as a bare `{"type": "object"}`, so no
method schema constrains it.

But the consequence is **not** uniform, because a second layer (the legacy contracts) also
runs. Measured per method, with a correct card and a typo'd one:

| method (migrated / in production) | typo'd key caught? | caught by |
|---|---|---|
| Bridson | yes | schema (`additionalProperties` on the Sampling core) + `JV2-MTH-010` |
| Random / CSV | yes | their method schemas carry real `required`/`anyOf` content |
| Grid | yes | schema |
| Dynesty / MultiNest | yes | legacy contracts (`JV2-BND-001`) |
| **AdaptiveBridson** | **no — 放行** | **nothing** |

```
AdaptiveBridson: {initial_radius: 0.08}                  → 放行   (correct)
AdaptiveBridson: {initial_radiuz: 0.08, 瞎写: 1}          → 放行   ← identical outcome
```

`Sampling.AdaptiveBridson` is typed as a bare object, so its whole sub-block is a
free-for-all: a misspelled `initial_radius` / `refinement_factor` / `threshold` /
`max_points` is accepted and the sampler silently uses its default. This is the method
driving the production iDM scans, and its keys *are* already specified in
YAML_REFERENCE §6.9 — so closing it is transcription, not design.

**Recommendation**: give `adaptive_bridson.json` a real `AdaptiveBridson` sub-schema
(`properties` + types + `additionalProperties: false`) from §6.9. Leave the MCMC-family
schemas as-is until that family is actually migrated — pinning an unsettled interface is
worse than leaving it open. Consider moving the nested-sampler `Bounds` rules from the
contracts layer into `dynesty.json`/`multinest.json` later, so one layer owns per-method
shape; not urgent, since they are covered today.

**Open question for the maintainer** (scoping, not a bug): the un-migrated MCMC family is
nonetheless *registered in the Distributor and passes validation today*, so a user can
write `Method: DRAM`, get a clean `Jarvis2 validate`, and launch it. If that family is not
ready for users, it may be worth marking it experimental at the registry level (a warning
diagnostic, or omission from the "Available:" list in `JV2-MTH-003`) rather than letting
validation imply readiness.

Also: `test_manifest_gives_each_sampling_method_its_own_schema` asserts only that the 17
*URIs* are distinct — it passes trivially while the contents are equivalent. It reads like
a guard for per-method content but does not enforce any.

## 2. The manifest pins the I/O format list, which breaks the Portal-upgrade principle

I/O formats do not come from HEP's registry — their authority is the **Jarvis-Portal
source**, and Portal is a separately-versioned package. The README states the resulting
principle plainly: *plugin packages — upgrade them to extend HEP without a HEP release.*

The validator does not honour that. `_validate_selected_io` **never consults Portal**; it
reads `manifest["io"][direction]` and treats a name that is absent from that hardcoded map
as a hard error:

```python
schema_uri = formats.get(kind)          # manifest is the sole authority
if schema_uri is None:
    issues.append(issue("error", "JV2-SCH-002",
        f"unsupported {direction} format {raw_kind!r}; register a local schema in schema/manifest.json"))
```

Simulated a Portal upgrade that adds `HepMC` (HEP source untouched, exactly the scenario
the principle promises to support):

```
Portal now supports input: [CSV, DAT, JSON, SLHA, Text, TSV, Wolfram, HepMC]
card using type: HepMC  →  JV2-SCH-002  "unsupported input format 'HepMC';
                                         register a local schema in schema/manifest.json"
```

So a format Portal can genuinely read is rejected until someone edits HEP and ships a
release — the coupling the plugin design exists to avoid. And
`test_manifest_matches_builtin_portal_formats` asserts **set equality** between the
manifest and `available_io_formats()`, which means the same Portal upgrade also turns
HEP's test suite red without a single HEP change. That test currently enforces the
coupling rather than the principle.

**Recommendation**: invert the authority. Accept the format if **Portal** supports it
(`available_io_formats(direction)`); use the manifest only to *enrich* validation for the
formats HEP happens to ship a schema for. Then:

- Portal-supported **and** schema present → full schema validation (today's behavior);
- Portal-supported, **no** schema → accept; at most an info/debug note that detailed
  checking is unavailable — never an error;
- **not** Portal-supported → the existing hard error, now with the honest message
  ("Portal does not provide an adapter for …", listing `available_io_formats()`), which is
  also the message a user can act on.

Change the test to a **subset** assertion (every manifest format must be Portal-supported,
not vice versa), so adding a Portal format never breaks HEP, while a stale HEP schema for
a format Portal dropped still fails loudly.

Note the related asymmetry is *not* a defect once the categories are separated: an
unknown **sampling method** is caught by `JV2-MTH-003` against the Distributor registry
(verified: `Method: dram` → clear error listing all valid names, exit 2). The one case
that still fails open is a method that **is** registered but has no manifest entry — the
normal act of adding a sampler. Simulated by registering one, a card with garbage `Bounds`
produced **zero** diagnostics. There is no Distributor↔manifest consistency test to catch
that omission (Portal↔manifest has one). Worth adding, and cheap.

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
| 1 | I/O validation must follow **Portal**, not the pinned manifest: accept any Portal-supported format (schema only enriches), and relax the equality test to a subset assertion | **high** — rejects valid cards + breaks HEP tests on any Portal upgrade |
| 2 | `adaptive_bridson.json`: real sub-block schema from YAML_REFERENCE §6.9 (the only migrated method with no coverage; drives the production iDM scans) | **high** — silent default fallback today |
| 3 | Distributor↔manifest consistency test + a diagnostic when a registered method has no schema (and manifest↔directory while at it) | medium |
| 4 | Strengthen `test_manifest_gives_each_sampling_method_its_own_schema` to require method-specific content | low |
| 5 | Root schema `$id`/filename consistency | cosmetic |

**Separately filed as D13.15** — the plan header has recorded *"608 passed with 8
pre-existing failures"* since 2026-07-29. That was not re-verified in this review (the
standing instruction is not to re-run the suite for review work), but a persistent set of
known-failing tests should not stay a footnote: it makes Protocol §5 (*a WP is not done
with failing tests*) unenforceable, because the next agent cannot separate its own
breakage from the background. Fix them or `xfail` them with a stated reason, so that
green means green.
