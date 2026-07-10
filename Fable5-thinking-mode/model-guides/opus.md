# Operating Manual · Opus-class models · Jarvis-Workshop

> Audience: Claude Opus or any model of comparable capability working in
> `/Users/p.zhu/Jarvis-Workshop`. Load this file whole at session start,
> after the root `AGENTS.md`. If a rule here has stopped being true,
> delete it — a stale rule is worse than no rule.

## 1. Your mandate

You are trusted with engineering judgment: refactors, test design, tooling,
documentation structure, dependency choices, and proposing architecture.
Decide, act, and record the reasoning where the next reader will look
(design docs, not chat). Two gates apply regardless of how confident you are:

1. **The physics gate.** Any change that moves a computed physical
   observable — relic density, direct-detection rates, cross sections,
   likelihoods, sampled distributions — must be explicit: state the expected
   numerical shift on a named reference point, show before/after values, and
   get confirmation before merging. A refactor that "shouldn't change
   numbers" must demonstrate that it didn't. Never let numbers move silently.
2. **The contract gate.** Public interfaces are two-sided contracts: the
   YAML config schema, the Agent-Bridge `hep_*` tool signatures, the JSON
   I/O shapes. Change both sides and the governing design doc in the same
   change cycle, or change neither.

Everything not behind these gates is yours to decide.

## 2. Workspace map

| Path | What it is | State / caution |
|------|-----------|-----------------|
| `Jarvis-HEP-v2/` | Python scan framework, package `jarvishep2`, tests in `tests/` | Largely machine-generated code, never human-reviewed (see §5). Test suite green (207 as of 2026-07; trust the live run, not this number) |
| `Jarvis-Books/Docs/` | Design layer for v2: `YAML_REFERENCE_2.0.md` (config contract), `DESIGN_AGENT_BRIDGE_2.0.md` (V2 D8 ↔ Agent M4.5 `hep_*` tools), `V2_DISTRIBUTED_PLAN.md`, code-review notes | This is the authority on intent; the code is not |
| `Jarvis-HEP/` | v1 framework, legacy | Governed by the change gate in root `AGENTS.md`: read its five canonical docs in `Jarvis-HEP/docs/` before touching it |
| `Jarvis-Examples/Program/microMEGAs/micromegas7/` | Modified micrOMEGAs 7 | Carries local physics modifications (§4). `micromegas_6.2.3/` beside it is an older tree that also carries mods — confirm which tree a task targets before editing |
| `Jarvis-Books/TeX/Jarvis-HEP manual/` | Finished 148-page tufte-book LaTeX manual, English | Complete. Maintenance edits only; do not restructure. Build: `latexmk -pdf` |
| `Jarvis-Docs/` | MkDocs documentation site | Strict conventions (§3). Validate with `mkdocs build --strict` |
| `Jarvis-Agent/` | Agent side of the Agent-Bridge (M4.5, `hep_*` tools) | Interface changes hit the contract gate |
| `Jarvis-Operas/`, `Jarvis-PLOT/`, `Jarvis-Portal/`, `Workshop/`, `BP_script/` | Not yet covered by this manual | Explore before assuming anything |

Reading order for a task: root `AGENTS.md` → this file → the design docs of
the target repo (v2: YAML reference + Agent-Bridge design; v1: the five
canonical docs; micrOMEGAs: §4 below, then the source you touch).

## 3. Standing rules

- Root `AGENTS.md` is binding. Two of its rules generalize beyond v1 and you
  should apply them workshop-wide:
  - **Documentation-sync**: when an implementation phase completes or an
    understanding turns out wrong, write the correction into the canonical
    docs in the same change cycle — never leave decisions only in chat.
  - **PR contract**: open every PR-style summary with (1) roadmap phase,
    (2) affected modules, (3) whether public interfaces change, (4) main
    risks, (5) verification performed.
- **English only** in all docs, comments, commit messages, and identifiers.
- Docs site conventions: `<aside>` blocks, never `!!!` admonitions;
  raw-HTML links between pages use the `../page/` form; every docs change
  ends with a clean `mkdocs build --strict`.
- Comments state constraints the code cannot express; they do not narrate
  the change or justify it to a reviewer.
- Match the idiom of the file you are editing, even where you would have
  written it differently.

## 4. Known landmines

Each of these has already cost real debugging time. The rule is the distillate.

1. **macOS `complex.h` vs C++ linkage** (micrOMEGAs): C99 `_Complex` types
   must never cross an `extern "C"` boundary into C++ code. Rule: new
   micrOMEGAs project code is C++-only; complex arithmetic uses `<complex>`;
   headers shared with the C core keep complex types out of their signatures.
2. **JSON I/O is the default** (micrOMEGAs): new project programs are built
   on `main.cpp` + `include/micromegas_json.hpp` (nlohmann). Do not regress
   to the stock C `main.c` flow when creating or extending projects.
3. **`nextOdd` is 0-indexed** (micrOMEGAs). Off-by-one here silently
   corrupts particle bookkeeping.
4. **`DeltaDD` is half-implemented on purpose** (`sources/directDet.c`):
   the global (default `0` = elastic) feeds the inelastic kinematics
   (`v_min`) only. The cross-section side is deliberately deferred. Do not
   "helpfully" complete it; a task that needs it starts with a design note,
   not code.
5. **The R(s) hadronic correction is relic-only** (`sources/improveCS.c`
   override): gated on `T>0`, switched from config, `sigma_mumu`
   auto-tabulated. It must never leak into direct-detection or
   indirect-detection paths. Preserve the gate in any refactor.
6. **Green tests prove the happy path, not the design** (Jarvis-HEP-v2).
   The suite passing is necessary, never sufficient — see §5.
7. **YAML schema drift**: any config option added, renamed, or removed must
   land in `YAML_REFERENCE_2.0.md` in the same change cycle, with a runnable
   example. The reference doc is the contract; code follows it.
8. **The Agent-Bridge is a two-sided interface**: `hep_*` tool signatures
   bind `Jarvis-HEP-v2` (D8) and `Jarvis-Agent` (M4.5). Changing a
   signature means changing both repos and `DESIGN_AGENT_BRIDGE_2.0.md`
   together.

## 5. Working with the machine-generated code in Jarvis-HEP-v2

The v2 codebase was largely written by a code generator and has not been
human-reviewed. This inverts a default you normally hold:

- **Chesterton's fence does not apply.** An odd construct is more likely a
  generator artifact than a load-bearing subtlety. But "likely" is not
  "certainly": before simplifying, write the failure-path test that would
  catch the regression you fear, watch it pass, then simplify.
- **Do not trust comments or docstrings** to describe actual behavior; they
  were produced by the same process as the code. Verify claims against
  behavior (run it) or against the design docs (which are authoritative).
- **Duplication is the most common artifact.** Unify only when tests cover
  every call site you are merging.
- **Prioritize review effort** by physics-output impact × complexity:
  sampler internals and likelihood plumbing before CLI glue.
- When you find and fix a real defect, leave a regression test and a one-line
  note in the module docstring; the next model must not rediscover it.

## 6. Quality bars

A deliverable is done when every line of its bar checks off — not before.

**New module (v2)**
- [ ] Unit tests including at least one failure-path test
- [ ] Entry in `YAML_REFERENCE_2.0.md` if it adds config surface
- [ ] Design-doc touch if it adds architecture
- [ ] Full suite green, output pasted into the summary
- [ ] PR-contract header (§3)

**Bug fix**
- [ ] Regression test that fails before the fix and passes after
- [ ] Root cause stated in one sentence in the summary
- [ ] Diff contains nothing unrelated to the fix

**Config-schema change**
- [ ] `YAML_REFERENCE_2.0.md` updated with a runnable example
- [ ] Validation rejects the old/invalid form with an error message that
      names the offending key and the fix
- [ ] Migration note if any existing YAML in `Workshop/YAMLs/` breaks

**Physics change (micrOMEGAs or v2 physics modules)**
- [ ] Reference point named; before/after observable values in the summary
- [ ] Comparison against the unmodified path (switch off = old numbers)
- [ ] Config switch documented
- [ ] Physics gate (§1) explicitly cleared

**Docs change**
- [ ] Conventions of §3 held
- [ ] `mkdocs build --strict` clean, output pasted

## 7. Verification recipes

```bash
# Jarvis-HEP-v2 — full suite
cd /Users/p.zhu/Jarvis-Workshop/Jarvis-HEP-v2 && python -m pytest

# Docs site — strict build
cd /Users/p.zhu/Jarvis-Workshop/Jarvis-Docs && mkdocs build --strict

# LaTeX manual
cd "/Users/p.zhu/Jarvis-Workshop/Jarvis-Books/TeX/Jarvis-HEP manual" && latexmk -pdf

# micrOMEGAs fork — rebuild the touched project, re-run its reference point,
# and diff the JSON output against the last committed reference output.
```

Run the recipe that covers what you touched, every time, and paste real
output — never summarize it as "tests pass".

## 8. Escalate only when

- The physics gate or contract gate triggers (§1).
- The action is destructive or hard to reverse: history rewrites, force
  pushes, deleting data/logs/scan results, overwriting files you did not
  create.
- The task implies a milestone-scope change (e.g. bringing forward the
  deferred DeltaDD cross-section work).
- Credentials, tokens, or anything leaving the machine.

Everything else: proceed, verify, and report what you did with evidence.

---
*Written 2026-07-08 by Claude (Fable 5) from a full-workspace review.
Keep facts in §2/§4 in sync with `sonnet.md` and `local-9b.md`.*
