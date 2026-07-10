---
name: extract-approach
description: >
  After solving a non-trivial problem, distill HOW it was solved into a
  permanent learnings note before moving on. Trigger after: a debugging
  session that found a root cause, an architecture or physics-modeling
  decision, a tricky build/toolchain fix, a performance hunt, or any solve
  that took more than ~20 minutes of reasoning. Do NOT trigger for routine
  edits, typo fixes, boilerplate, or changes fully covered by existing
  conventions.
---

# extract-approach

You just solved a non-trivial problem. Before moving on, write a learnings
note so the approach outlives this session. A solution without its learnings
note is unfinished work.

## Steps

1. **Name the solve.** One line: what was broken or undecided, and what is
   true now. If you cannot state this in one line, the problem is not solved
   yet — go back.
2. **Write the note** to `docs/learnings/YYYY-MM-DD-<kebab-slug>.md` using
   the template below. Create the directory if it does not exist.
3. **Link it.** If a `docs/learnings/INDEX.md` exists, append one line:
   `- [title](YYYY-MM-DD-<slug>.md) — one-line hook`. Create INDEX.md on
   first use.
4. **Promote if general.** If the "general rule" line is a convention every
   future session should follow (not just a fact about one bug), also add it
   to the repo's CLAUDE.md under its conventions section — one line, linking
   back to the note.
5. Keep the whole note **under one page**. If it is longer, you are
   narrating; cut until only the transferable parts remain.

## Note template

```markdown
---
date: YYYY-MM-DD
problem: <one line>
area: <repo/module, e.g. jarvis-hep-v2/sampler or micromegas/build>
tags: [<2-4 short tags>]
model: <model that produced this solve, e.g. claude-fable-5>
---

### Symptom
The exact observable failure: error strings verbatim, wrong numbers with
their expected values, the command that reproduces it.

### Root cause / key insight
The single sentence that, had you known it at the start, would have made
this a 5-minute fix. Then the minimal supporting explanation.

### The approach that worked
Numbered steps, in order, with exact commands and file paths. A model with
no access to this conversation must be able to replay them.

### Dead ends
What was tried and failed, and WHY it failed — this saves every future
model from repeating the loop. Omit only if there truly were none.

### The general rule
MANDATORY. One sentence a weaker model can apply without judgment.
Bad:  "be careful with linkage on macOS"
Good: "new micrOMEGAs projects are C++-only; complex numbers use <complex>
       and never cross an extern \"C\" boundary as C99 complex"

### Verification
How we knew it was fixed: the command run and the output that proves it.
```

## Rules

- Write for a model that has **no access to this conversation** and no
  memory of today.
- Prefer exact commands, file paths, and verbatim error strings over prose.
- One note per problem. Two problems solved = two notes, even if solved in
  the same session.
- The **general rule** line is the distillate — it is mandatory, and it must
  be checkable, not an adjective.
- Never skip the note because the fix "turned out to be simple". If finding
  it was expensive, the note is valuable precisely because the fix looks
  simple in hindsight.
