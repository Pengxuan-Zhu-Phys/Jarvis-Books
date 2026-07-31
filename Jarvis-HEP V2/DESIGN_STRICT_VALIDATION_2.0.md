# DESIGN — Strict Task-Card Validation (V2, D17)

**Status**: design accepted 2026-07-31; **implemented and verified 2026-07-31** (`69085f4`) — see §7
**Date**: 2026-07-31
**Requirement (maintainer)**: *"让 coding agent 按照严格 json schema 的形式进行校验，即如果出现
了非法的 key and/or value type，提出明显的日志提醒，并退出。"*
**Builds on**: D13.9 validation gate ([`DESIGN_YAML_VALIDATION_2.0.md`](DESIGN_YAML_VALIDATION_2.0.md)),
findings in [`SCHEMA_REVIEW_2026-07-31.md`](SCHEMA_REVIEW_2026-07-31.md).

---

## 0. Read this first — strictness is phase 2, not phase 1

The goal is not in dispute: an illegal key or a wrong value type must produce an obvious
diagnostic and stop the run before anything starts. The problem is that **the schemas are
not yet complete enough to be told the truth from**, and turning strict on today would
reject most of the shipped example library.

Measured against the 65 real task cards in `Jarvis-Examples/*/bin/`:

```
通过 21   报错 44   (JV2-SCH-001 ×41, JV2-MTH-003 ×10)
```

Two thirds already fail — and the failures are **schema omissions of legitimate keys**,
not bad cards:

| what the validator rejects | × | is the card wrong? |
|---|---|---|
| `Calculators.path` | **35** | **No.** `worker_config.py:208` reads it with the comment *"Top-level Calculators.path is V1 layout metadata … **Tolerate it (do not error)**"* — the runtime deliberately accepts it while the schema deliberately rejects it. The two layers contradict each other, and the schema is the one that is wrong. |
| `Calculators.Modules[].deps_source` | 4 | No — real V1 key, absent from the schema. |
| `Operas.Modules[].selection` | 1 | No — module-level `selection` is a documented V1 feature. |
| `Sampling.{Control,Diagnostics,PPO}` | 1 | Legitimately unsupported (RLTPMCMC, un-migrated). |
| unknown `Method` (`JV2-MTH-003`) | 10 | Legitimately unsupported (un-migrated methods). |

**So the sequencing is forced**: complete the vocabulary against the real corpus first,
flip strictness second. Flipping first would not make Jarvis stricter — it would make it
wrong, loudly, 44 times.

## 1. Goals

1. In **closed** zones, an unknown key or a wrong value type is an **error**: clear
   message → non-zero exit → **nothing starts** (no Redis, no Workers, no output dir).
2. Every diagnostic states *where*, *what*, and *how to fix it* — the existing
   `_schema_error_guidance` machinery (suggestion + YAML example) already does this and
   must cover the new checks too.
3. **No false rejections.** A card that the runtime would have executed correctly must
   never be refused. This is the constraint that makes §2 necessary.
4. Strictness is **declared, not accidental**: every block is explicitly classified, and
   an unclassified block is a bug.

### Non-goals

- Validating what an external, separately-versioned component owns (§2 *delegated*).
- Making the un-migrated MCMC family strict — its vocabulary is not settled
  (maintainer, 2026-07-31); it stays *open* with a recorded reason.
- Replacing the legacy contracts layer in this WP (it may later fold into the schemas).

## 2. The vocabulary taxonomy (the core of this design)

"Strict" is meaningless without saying *who owns each name*. Every block gets exactly one
of three classifications, recorded in the schema itself via `x-jarvis-zone`:

| zone | who owns the vocabulary | unknown key | wrong type |
|---|---|---|---|
| **closed** | HEP | **error + exit** | **error + exit** |
| **delegated** | an external, separately-versioned package | **pass through** | checked only for keys HEP declares |
| **open** | nobody yet — explicitly unfinished | warn once | not checked |

**closed** — `Scan`, `Calculators` (top level + module shape), `Operas.Modules` shape,
`EnvReqs.V2`, `Sampling` core, and the migrated methods' own blocks. HEP defines these
names; a name outside the list is a typo by definition.

**delegated** — vocabularies HEP does not own and must not pin:
- **`Calculators.Modules[].execution.{input,output}[].type`** → the authority is the
  **Jarvis-Portal source**. Accept whatever `available_io_formats(direction)` reports;
  a bundled per-format schema only *enriches* the check when present. (See
  `SCHEMA_REVIEW_2026-07-31.md` §2: pinning this list in HEP's manifest breaks the
  documented "upgrade Portal without a HEP release" principle and reddens HEP's suite on
  any Portal upgrade.)
- **`Sampling.Bounds.{run_nested,sampler}`** → forwarded verbatim to the vendored dynesty
  API. HEP must not enumerate another library's kwargs surface.
- **`Operas.Modules[].kwargs`** → forwarded to the operator callable; the signature
  filter (D11.4) is the real check, at call time.
- **`EnvReqs` siblings of `V2`** (`Python`, `CERN_ROOT`, `Check_default_dependencies`, …)
  → V1 environment vocabulary; `task_config.py` already tolerates them by design.

**open** — recorded unfinished work, each with a reason and a WP link in the schema:
`Sampling` blocks of the un-migrated methods (MCMC family, RLTPMCMC's
`Control`/`Diagnostics`/`PPO`). These warn once ("`Method: DRAM` is not migrated;
its options are not validated — D13.x") instead of pretending to be checked.

**Rule**: a block with no `x-jarvis-zone` fails the schema self-test (§5). That is what
keeps "open" from silently becoming the default the way it did for `AdaptiveBridson`.

## 3. Value types: two traps that must be handled before flipping

### 3.1 `1e-5` is a **string** in YAML

```
a: 1e-5    → '1e-5'  str      ← physicists write this constantly
b: 1.0e-5  → 1e-05   float    ← only this form is a float
c: 1E-5    → '1E-5'  str
e: 1e5     → '1e5'   str
```

YAML 1.1 requires a decimal point and a signed exponent for scientific notation. A naive
`{"type": "number"}` therefore rejects `threshold: 1e-5` — a card that runs fine today,
because the runtime coerces with `float()`.

**Decision**: numeric fields use a `numeric` type that accepts a number **or** a string
matching `^[+-]?(\d+\.?\d*|\.\d+)([eE][+-]?\d+)?$`, defined once in `core/common.json`.
A string form emits a **warning** suggesting the canonical `1.0e-5`, never an error.
Strictness must catch typos and wrong *kinds* of value — not YAML's own notation quirk.

### 3.2 `on`/`off`/`yes`/`no` are booleans

YAML 1.1 parses `on`, `off`, `yes`, `no` as booleans. A field expecting a string (e.g. a
mode name) receives `True`. Declare such fields as `string` and let the error message name
the quoting fix (`mode: "on"`), with an example.

## 4. Failure behaviour (what the maintainer asked for)

On any **error**-level issue, from both `Jarvis2 validate` and `Jarvis2 run` /`check`:

```
Task card validation FAILED — 3 error(s), 1 warning(s). Nothing was started.

  [error] JV2-SCH-010  $.Sampling.AdaptiveBridson.initial_radiuz
          unknown key 'initial_radiuz' in a closed block
          suggestion: Remove or rename it. Allowed: initial_radius, refinement_factor,
                      threshold, max_generations, max_points, target, target_value
          did you mean: initial_radius
          example:
            AdaptiveBridson:
              initial_radius: 0.08

  [error] JV2-SCH-011  $.Sampling.Bounds.chains
          expected integer, got string '四'
          ...

Fix the card and re-run.  Full key reference: docs/task-card-schema.md
```

Binding requirements:

- **Validation runs before any side effect** — before Redis connect, Worker spawn, output
  directory creation, or `logs/<scan>/`. Today's gate already runs early; keep it there.
- **Exit code 2** (matches the existing `Jarvis2 validate` convention; `run` must adopt
  the same code rather than inventing one).
- **All errors reported at once**, sorted by path — never fail on the first one. A
  physicist should fix a card in one pass, not N runs.
- **"did you mean"**: unknown keys in closed blocks run a `difflib.get_close_matches`
  against the allowed set. This is the single highest-value line in the whole message,
  since the dominant real failure is a typo.
- **Warnings never exit.** Dead-but-tolerated keys (`Calculators.path`) warn and say what
  they are ignored *for*.

## 5. Work packages

| WP | Title | Accept |
|---|---|---|
| **D17.1** | Complete the closed vocabulary against the real corpus | Every key appearing in the 65 `Jarvis-Examples` cards + 23 template/parity cards is either declared in a closed schema, classified delegated/open, or **deliberately** an error with a written justification. `Calculators.path`, `Modules[].deps_source`, `Operas.Modules[].selection` specifically resolved (declare, or demote to warning — the runtime tolerates them). |
| **D17.2** | `x-jarvis-zone` classification + schema self-test | Every object block carries a zone; a missing zone fails the self-test. Delegated I/O consults `available_io_formats()`; Portal↔manifest test relaxed to a **subset** assertion. |
| **D17.3** | `numeric` type + YAML-quirk handling | `threshold: 1e-5` passes with a canonical-form warning; `chains: "四"` errors. Corpus stays green. |
| **D17.4** | Strict error semantics + "did you mean" + one-pass reporting + exit 2 from `run`/`check` | A card with 3 mistakes reports all 3 once; nothing is created on disk; exit 2. |
| **D17.5** | **Flip the switch** + corpus regression test in CI | `test_all_shipped_cards_validate_strictly` over `Jarvis-Examples` + templates + parity cards is green **before** the flip lands; skills + YAML_REFERENCE updated (D16 rule). |

**Ordering is binding**: D17.5 must not land before D17.1–D17.4 are green. The corpus
test is the gate, not a formality.

**Rollback**: `JARVIS_VALIDATE_PERMISSIVE=1` downgrades errors to warnings for one
release, printing a prominent banner naming the keys that would have failed. Documented
as an emergency escape with a removal date, not a supported mode. (If the maintainer
prefers no escape at all, drop it — the corpus gate in D17.5 is what actually protects
users.)

## 6. Risks

1. **A "strict" flip that rejects valid cards destroys trust faster than lax validation
   ever did.** The corpus gate exists for exactly this; do not weaken it to unblock the
   flip.
2. **Delegated zones drifting into closed** — a future agent "tidying up" by adding
   `additionalProperties: false` to a delegated block would re-break the Portal principle.
   The `x-jarvis-zone` marker plus its self-test is the guard; keep the rationale comment
   in the schema file next to it.
3. **Warning fatigue** — if `Calculators.path` warns on all 35 cards forever, users learn
   to ignore warnings. Prefer *declaring* tolerated V1 keys in the schema (silent, valid)
   over warning about them, and reserve warnings for things that genuinely deserve a fix.


---

## 7. Implementation verification (2026-07-31, `69085f4`)

Every acceptance criterion was re-measured against the running code, using the same
scripts that produced §0's numbers.

**The gate: the corpus went green.**

```
65 real Jarvis-Examples cards:   通过 55  报错 10      (before: 通过 21  报错 44)
remaining 10 = JV2-MTH-003 only  (DNN, RLTPMCMC, … — methods V2 genuinely does not
                                  implement; correct rejections, not schema gaps)
```

`Calculators.path` (×35), `deps_source`, `Operas.Modules[].selection` are now declared —
the schema no longer contradicts `worker_config.py`'s "Tolerate it (do not error)".

| requirement | result |
|---|---|
| §2 zones declared | `x-jarvis-zone` in 25 schema files |
| §2 delegated: dynesty pass-through | `Bounds.{run_nested,sampler}` with arbitrary keys → accepted |
| §2 delegated: Portal authority | simulated a Portal upgrade adding `HepMC` → **accepted**; a format Portal does *not* have → `JV2-SCH-002` listing Portal's real formats. The manifest is no longer an allowlist. |
| §1/§2 AdaptiveBridson (the one uncovered migrated method) | `initial_radiuz` → error **+ "Did you mean 'initial_radius'?"** |
| §3 numeric union | `1e-5` (YAML string) ✅, `1.0e-5` ✅, `'0.08'` ✅, `'abc'` ✗, `max_points: 'many'` ✗ |
| §4 exit code | `run` **2**, `validate` **2** |
| §4 zero side effects | after a failed `run`: no `outputs/`, no `logs/` — nothing created |
| §4 one-pass | two typos in one block reported together |
| §5 regression test | `tests/test_task_schema_corpus.py` (19 schema tests green) |

### 7.1 Three follow-ups (none blocking — filed as D17.6)

1. **Multiple typos lose both the key name and the "did you mean".**
   `task_schema.py:117` matches only the singular form:
   ```python
   re.search(r"\('([^']+)' was unexpected\)", error.message)
   ```
   jsonschema emits `('a', 'b' were unexpected)` for 2+ keys, so the regex misses and the
   suggestion degrades to the literal placeholder:
   ```
   1 typo : Remove or rename 'initial_radiuz'. Did you mean 'initial_radius'? …
   2 typos: Remove or rename 'the unexpected key'. Allowed keys: …      ← help lost
   ```
   The user gets *less* help exactly when they made *more* mistakes. Match both forms and
   run `get_close_matches` per key.

2. **`anyOf` numeric errors are not actionable** — a side effect of the §3 fix. Making
   numeric fields a number-or-numeric-string union means jsonschema reports the generic
   parent error:
   ```
   message : 'abc' is not valid under any of the given schemas
   suggestion: Use one complete accepted form for this object; inspect the listed schema alternatives.
   example : (none)
   ```
   §4 requires an editable instruction. Special-case the numeric union in
   `_schema_error_guidance`: *"expected a number (e.g. `0.05` or `1.0e-5`), got the string
   `'abc'`"*.

3. **The gate test is not hermetic.** `test_task_schema_corpus.py:14` globs
   `Path(__file__).parents[2] / "Jarvis-Examples/*/bin/*.yaml"` and then asserts
   `len(cards) >= 65` with no skip guard — so on any checkout where Jarvis-Examples is not
   a sibling directory (a clean CI runner), the gate fails for the wrong reason and
   reads as a schema regression. Either vendor a corpus snapshot under `tests/fixtures/`
   or `skipUnless` the directory exists **and** run it in a job that checks out both
   repos. A gate that cannot run everywhere is not yet a gate.

Minor: the `Allowed keys:` list inlines all 27 AdaptiveBridson keys into one line. With
"did you mean" already naming the fix, consider truncating the dump.

---

## 8. ASCII-only task cards (maintainer decision, 2026-07-31)

**Decision**: *"YAML 中明确不支持中文，若有中文直接退出。"* Non-ASCII text in a task card
is an error, reported like any other strict violation and exiting before anything starts.

### 8.1 Corpus impact: none

Measured across everything shipped — 65 `Jarvis-Examples` cards, 7 `project_template`
cards, 6 `parity_project` cards:

```
78 份卡片，0 份含非 ASCII  →  0 份会被新规则拒绝
```

Unlike D17's strictness (which needed the vocabulary completed first), **this rule can
ship immediately**: nothing in the corpus breaks, so there is no migration phase and the
gate test stays green on day one.

### 8.2 Scope — and what stays legal

| location | rule | why |
|---|---|---|
| **keys** | ASCII only → error | already an error today, but as a confusing *"additional properties"*; this gives it an honest message |
| **string values** | ASCII only → error | `Scan.name` **currently passes with Chinese and becomes a directory name** — the most damaging case, since it propagates into paths, tar members, and HDF5 attributes |
| **comments** | **unrestricted** ✅ | YAML comments are discarded by the parser and never reach validation (verified). Users keep documenting cards in Chinese — the skills library tells them to |

The hint on every violation should say exactly that: *put the Chinese in a `#` comment,
which is fully supported.* That turns a refusal into a one-line fix.

### 8.3 ASCII, not "CJK"

Ban **all non-ASCII**, not Chinese specifically:

1. **Homoglyphs.** A Cyrillic `а` (U+0430) inside `nаme:` is visually identical to ASCII
   `a`. Under a CJK-only rule it slips through and produces a baffling "unknown key
   `nаme`" against an allowed list that appears to contain it. One ASCII rule kills the
   entire class.
2. **No "is this Chinese enough" ambiguity** — one predictable rule, one message.
3. Paths, tar members, and HDF5 attribute names are safest as ASCII regardless of script.

**No field is exempt** — maintainer, 2026-07-31: *"scan.name 也不允许用中文，英文是唯一
YAML 语言."* That closes the one open question in this section: free-text fields
(`description`, …) follow the same rule as everything else. English is the task card's
only language; anything a user wants to write in Chinese goes in a `#` comment, which
stays fully supported.

`Scan.name` is the sharpest case and is explicitly covered: it passes validation today
and then becomes a directory name, propagating into paths, tar members, and HDF5
attributes.

### 8.4 Where it runs, and what it prints

- **Before schema validation**, on the parsed document, so a Chinese key produces
  `JV2-ENC-001` instead of a confusing `JV2-SCH-001 additional properties` error.
- New code **`JV2-ENC-001`**, one issue per offending location, with the YAML path and
  the character positions.
- Same exit contract as the rest of D17: **exit 2, nothing started, nothing on disk.**

### 8.5 This supersedes the CJK table fix — if diagnostics escape non-ASCII

Note the trap: banning Chinese does **not** by itself fix D17.6's misaligned summary
table, because the error *reporting* the Chinese must display it, and that string is
exactly what overflows the box-drawing width.

Resolve it by **escaping non-ASCII in diagnostics** — render the offending text as
`\uXXXX` (or as a caret position under an ASCII-only excerpt) so every line the table
prints is ASCII by construction. Then the table's character count equals its display
width, and D17.6 item (1) (`unicodedata.east_asian_width` plumbing) can be **dropped
entirely** rather than implemented. One rule replaces two mechanisms.

### 8.6 Implementation verification (`6e308a4`)

Re-measured against the running code.

| requirement | result |
|---|---|
| Chinese **key** | `JV2-ENC-001`, path + character positions |
| Chinese **`Scan.name`** (passed before!) | now rejected |
| Chinese **`description`** (no exemption) | rejected |
| **Cyrillic homoglyph** `nаme` (U+0430) | rejected at position 2 — and rendered `nаme`, which makes the invisible character *visible*. Unplanned benefit of the escaping rule |
| **Comments** | a card with Chinese comments throughout **validates successfully** |
| **Corpus** | 77 cards, **0** rejected by the ASCII rule — zero migration cost, as predicted |
| exit code / side effects | 2, nothing created |
| §8.5 escaping ⇒ table alignment | solved *better* than specced: the table itself is now ASCII (`+---+`) **and** contents are escaped, so it stays aligned by construction. D17.6's `east_asian_width` work was correctly dropped |

D17.6 items also verified: plural typos now name **each** key with its own hint
(`'initial_radiuz' (did you mean 'initial_radius'?); 'max_pointz' (did you mean
'max_points'?)`); numeric errors read *"expected a number (e.g. 0.05 or 1.0e-5), got the
string 'abc'"*; the corpus test is now `skipUnless(EXAMPLES_ROOT.is_dir())`;
`_format_issue_summary_table([])` no longer raises.

### 8.7 Two follow-ups (filed as D17.8)

**(1) Derived keys produce phantom errors — the user cannot find them in their file.**
Validation runs on the **post-`load_task_yaml`** config, which carries keys the runtime
injected (`task_config.py:474–477`: `task_root`, `project_root`, `task_result_dir`,
`scan_name`). One Chinese `Scan.name` therefore reports **three** errors on the real CLI
path:

```
| 1 | JV2-ENC-001 | $.Scan.name       | … in string value '暗…'          ← the real mistake
| 2 | JV2-ENC-001 | $.scan_name       | … in string value '暗…'          ← injected copy
| 3 | JV2-ENC-001 | $.task_result_dir | … at position(s) 134, 135, …         ← derived path
```

The user wrote none of `scan_name` / `task_result_dir`; grepping their YAML for them
returns nothing, and "position 134" refers to an absolute path they never typed. Report
encoding violations only for **user-authored** keys — either validate the raw parsed
document before injection, or mark injected keys and skip them (and dedupe values derived
from an already-reported one).

**(2) Encoding errors suppress schema errors, so fixing takes two passes.** A card with
both Chinese *and* two typo'd keys reports only the `JV2-ENC-001`s; the typos appear only
after the Chinese is fixed and the card re-run. §4 requires *all errors at once — a
physicist should fix a card in one pass, not N runs*. The short-circuit is defensible for
non-ASCII **keys** (schema would re-report them as "additional properties"), but not for
unrelated blocks: suppress only the schema errors that name an offending key, and let the
rest through.

Neither is a correctness bug — the feature works and the corpus is green. Both are
against the design's own §4 usability contract.
