# Operating Manual · Sonnet-class models · Jarvis-Workshop

> Audience: Claude Sonnet or any model of comparable capability working in
> `/Users/p.zhu/Jarvis-Workshop`. Load this file whole at session start,
> after the root `AGENTS.md`. Your job is precise execution of
> well-specified work. When this manual gives a recipe, follow the recipe;
> when no recipe fits, scope down or escalate — do not improvise
> architecture. If a rule here has stopped being true, say so rather than
> silently working around it.

## 1. Decision boundary

Before acting, classify the action into exactly one column:

| DECIDE (just do it) | PROPOSE (write plan, wait for OK) | STOP (report and ask) |
|---|---|---|
| Bug fix with a regression test, inside one module | Refactor crossing module boundaries | Anything that changes a computed physical observable |
| Adding tests, docstrings, type hints | New module or new dependency | YAML config schema add/rename/remove |
| Docs edits following §3 conventions | Changing build tooling or CI | `hep_*` Agent-Bridge tool signatures |
| Renames local to one file | Deleting more than ~30 lines | Any edit under `microMEGAs/` trees not explicitly named in the task |
| Commit messages, PR summaries | Touching `Jarvis-HEP/` (v1 — has its own gate in root `AGENTS.md`) | Destructive git ops; deleting data/logs/scan outputs |

Two absolute rules sit above the table:

1. **Numbers must not move.** If, after your change, any physics output
   (relic density, cross sections, likelihoods, sampled values) differs —
   even in the last digit — revert to STOP: report the difference, do not
   merge, do not rationalize it.
2. **Contracts are two-sided.** Config schema, `hep_*` signatures, and JSON
   I/O shapes bind code and docs (and sometimes two repos). You never change
   one side alone.

## 2. Workspace map

| Path | What it is | What you must know |
|------|-----------|--------------------|
| `Jarvis-HEP-v2/` | Python scan framework, package `jarvishep2`, tests in `tests/` | Code is machine-generated and unreviewed: never treat existing code as a style or correctness authority. The design docs are the authority |
| `Jarvis-Books/Docs/` | v2 design layer: `YAML_REFERENCE_2.0.md` (config contract), `DESIGN_AGENT_BRIDGE_2.0.md`, `V2_DISTRIBUTED_PLAN.md` | Read the relevant doc BEFORE the relevant code |
| `Jarvis-HEP/` | v1, legacy | Do not touch without the root `AGENTS.md` gate: read the five canonical docs in `Jarvis-HEP/docs/` first |
| `Jarvis-Examples/Program/microMEGAs/micromegas7/` | Modified micrOMEGAs 7 (physics code) | STOP-column territory unless the task names it. Two trees exist (`micromegas7/`, `micromegas_6.2.3/`) — confirm which one the task means |
| `Jarvis-Books/TeX/Jarvis-HEP manual/` | Finished LaTeX manual (tufte-book, English, 148 pp) | Maintenance only. Never restructure. Build: `latexmk -pdf` |
| `Jarvis-Docs/` | MkDocs site | Conventions in §3; validate every change with `mkdocs build --strict` |
| `Jarvis-Agent/` | Agent side of the Agent-Bridge | Interface = contract; see rule 2 |
| Everything else (`Jarvis-Operas/`, `Jarvis-PLOT/`, `Jarvis-Portal/`, `Workshop/`, `BP_script/`) | Uncharted | Explore read-only first; PROPOSE before editing |

## 3. Hard conventions

- **English only** everywhere: docs, comments, commits, identifiers.
- Docs site: `<aside>` blocks, never `!!!` admonitions; raw-HTML links
  between pages use the `../page/` form; finish with a clean
  `mkdocs build --strict`.
- Smallest diff that meets the quality bar. No "while I was here" changes:
  if you notice an unrelated problem, put it in your report, not in the diff.
- Comments only for constraints the code cannot express. Match the
  surrounding file's idiom.
- Every claim of "done" comes with pasted command output. "Tests pass"
  without the pasted run does not count as verification.
- From root `AGENTS.md`, applied workshop-wide: docs are updated in the
  same change cycle as the code they describe, and every PR-style summary
  starts with (1) roadmap phase, (2) affected modules, (3) whether public
  interfaces change, (4) main risks, (5) verification performed.

## 4. Known landmines (do not rediscover these)

1. micrOMEGAs on macOS: C99 `_Complex` must never cross an `extern "C"`
   boundary into C++. New project code is C++-only; use `<complex>`.
2. micrOMEGAs new projects default to JSON I/O (`main.cpp` +
   `include/micromegas_json.hpp`, nlohmann). Never fall back to the stock
   C `main.c` flow.
3. `nextOdd` is **0-indexed**.
4. `DeltaDD` (`sources/directDet.c`, default `0` = elastic) implements the
   inelastic **kinematics only**. The cross-section side is deferred **on
   purpose**. Never implement it as a drive-by; that is a STOP.
5. The R(s) correction (`sources/improveCS.c`) applies to relic density
   only, gated on `T>0`. Any edit near it must keep the gate intact.
6. Jarvis-HEP-v2: a green test suite proves the happy path only. Passing
   tests after your change is necessary, never sufficient evidence that
   surrounding code you relied on is correct.
7. Any config option change without a matching `YAML_REFERENCE_2.0.md`
   update is an incomplete change, even if all tests pass.

## 5. Recipes

Follow the numbered steps in order. Paste real output at every ✔ point.

### R1 · Fix a bug in Jarvis-HEP-v2

1. Reproduce it: write the regression test FIRST and watch it fail. ✔ paste
   the failing run. If you cannot reproduce, STOP and report what you tried.
2. Read the module and the section of the design doc that covers it.
3. Make the smallest fix inside the module.
4. Full suite: `cd Jarvis-HEP-v2 && python -m pytest`. ✔ paste the tail
   (summary line + any warnings).
5. Ask yourself: did any numeric output change? If yes → STOP rule 1.
6. Update any doc the fix invalidates (same change cycle).
7. Report using the PR-contract header (§3), root cause in one sentence.

### R2 · Add or change a config option (PROPOSE first)

1. Draft the `YAML_REFERENCE_2.0.md` entry FIRST — name, type, default,
   validation, one runnable example. This is the proposal; wait for OK.
2. Implement schema + validation. Invalid input must produce an error that
   names the key and the accepted form.
3. Add tests: one accepting the valid form, one rejecting an invalid form.
4. Grep `Workshop/YAMLs/` for existing files the change breaks; list them.
5. Full suite ✔ paste; doc entry landed in the same commit.

### R3 · Edit the docs site

1. Check conventions before writing: `<aside>` not `!!!`; internal
   raw-HTML links as `../page/`; English.
2. Make the edit.
3. `cd Jarvis-Docs && mkdocs build --strict` ✔ paste the result. A single
   warning = not done.

### R4 · Anything in the micrOMEGAs trees (only when the task names it)

1. Confirm the target tree (`micromegas7/` unless told otherwise) and say
   so in your report.
2. Read §4 landmines 1–5 again. C++-only for new code.
3. Build the touched project directory; re-run its reference point.
4. Diff the JSON output against the previous reference output. ✔ paste the
   diff (empty diff expected unless the task says otherwise; non-empty →
   STOP rule 1).

### R5 · Edit the LaTeX manual

1. Maintenance only: fix content, never layout architecture.
2. `cd "Jarvis-Books/TeX/Jarvis-HEP manual" && latexmk -pdf` ✔ paste the
   final status lines; zero new warnings introduced.

## 6. Anti-patterns (each has already happened; do not repeat)

- Inventing a config key from memory instead of grepping the schema and
  `YAML_REFERENCE_2.0.md`.
- Treating existing v2 code as a pattern to imitate (it is unreviewed
  generator output — imitate the design docs instead).
- "While I was here" refactors bundled into a fix.
- Claiming verification without pasted output.
- Editing physics constants, units, or formulas to "clean up" code.
- Leaving a doc stale because "the code is the truth" — here the docs are
  the truth (root `AGENTS.md` doc-sync rule).

## 7. When uncertain

1. Search the design docs in `Jarvis-Books/Docs/`, then the code, then git
   history — in that order.
2. Still uncertain after all three: STOP and ask, stating what you checked
   and the exact decision you need. Never fill a gap with a guess in
   DECIDE-column clothing.

---
*Written 2026-07-08 by Claude (Fable 5) from a full-workspace review.
Keep facts in §2/§4 in sync with `opus.md` and `local-9b.md`.*
