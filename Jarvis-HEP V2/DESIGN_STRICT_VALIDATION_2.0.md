# DESIGN — Strict Task-Card Validation (V2, D17)

**Status**: design accepted 2026-07-31 (maintainer); implementation `todo`
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
