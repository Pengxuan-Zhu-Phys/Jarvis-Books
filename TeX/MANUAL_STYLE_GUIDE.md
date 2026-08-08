# Jarvis-Books — Manual Writing Standard (v2)

**Scope**: the LaTeX manuals under `Jarvis-Books/TeX/` (`Jarvis-HEP manual`,
`Jarvis-HEP-V2 manual`, and any future book).
**Status**: v2, adopted 2026-07-24. Supersedes v1 (2026-07-16, archived as
`attic-MANUAL_STYLE_GUIDE_v1_2026-07-16.md`).
**Language**: manuals are written in English. This guide applies to all of them.
**Audience of this file**: primarily AI writing/editing agents (Sonnet-class and up),
secondarily human editors. Every rule is written to be *mechanically checkable*
wherever possible; check commands are given inline.

v2 exists because v1 described *what good chapters look like* but not *how to produce
one reliably*. The chapters that reached high quality all went through the same three
disciplines — source-grounding, structural templates, and compile-plus-visual
verification — none of which v1 codified. v2 makes them mandatory.

---

## 0. How to use this file (agent protocol)

When asked to write or revise any chapter/appendix:

1. **Read this file first**, then the target chapter in full, then the chapters it
   cross-references (at minimum their openers and the referenced sections).
2. **Locate ground truth** (§1) for every factual claim you will add or touch.
3. **Write against the template** for the chapter's type (§3).
4. **Run the verification protocol** (§8) before declaring done — which, under the
   standing rule below, means stages V1–V2.
5. **DO NOT COMPILE.** Standing instruction from the user (2026-07-25), for *all*
   TeX work in these books, not per-task: never run `latexmk`/`pdflatex`, never
   inspect `main.log`, never rasterize the PDF. The user compiles by hand. Stages
   V3–V5 of §8 are therefore never run by an agent; do not ask for permission to
   run them, and do not report compile-derived numbers (page counts, Overfull
   tallies, "0 undefined references") — you will not have them.

Never mix a *content* change and a *mechanical migration* (rename, renumber,
environment swap) in one uninspectable pass — do the mechanical pass scripted and
verified separately, then the content pass.

---

## 1. Ground truth: source-first writing

This is the single most important rule in the file. The book's value is that it
documents what the code *actually does* — including warts — not what a plausible
framework would do.

1. **Never write a key name, default value, alias, error message, file path, log
   format, or behavior claim from memory or inference.** Open the parsing function,
   engine file, or schema that implements it, in the actual source tree
   (`Jarvis-HEP-v2/jarvishep2/` for the V2 book). Registration call sites are not
   enough — read the code that *consumes* the value.
2. **Document the wart, don't silently fix it.** If the real key is misspelled
   (`make_paraller`), the real error is a raw `KeyError`, or an invalid entry is
   *silently dropped*, the book says exactly that. These are the highest-value
   sentences in the book.
3. **Label the epistemic status of every normative sentence.** Three levels, and the
   prose must make clear which one a claim is:
   - **Contract** — stable API, safe to build pipelines on (task-card schema, CLI
     exit codes, output layout, checkpoint format). Only these may use the word
     "contract"/"stable".
   - **Behavior** — what the current code does, stated in present tense, may change
     between releases.
   - **Heuristic** — tuning advice or diagnostic rules of thumb. These live in
     `jtip` boxes or troubleshooting rows, marked as advice, and are the *only*
     place hedge words are tolerated (§5.3).
4. **When a design doc and the code disagree, the code wins**, and the discrepancy is
   reported to the user rather than papered over.
5. **Verify configuration defaults at the real consumption site.** `__init__`
   placeholder values are frequently overwritten by a later `set_config`; trust the
   effective default, and if a test asserts it (e.g. a clamp), cite that behavior.
6. Figures imported from papers or older versions may show outdated architecture.
   Either caption them with an explicit old→new terminology mapping, or don't use
   them. Never let stale vocabulary stand uncontextualized.

---

## 2. Audience and layering

Two readers, one book:

| Reader | Wants | Where served |
|--------|-------|--------------|
| **User** (physicist; may be new) | "how do I do X", recipes that run | Parts Getting Started, Running; numbered steps + copy-paste cards |
| **Power user / maintainer** | complete key reference, failure semantics, internals | per-chapter key tables, Advanced part, appendices |

Rules:

1. Chapter openers name the audience and the outcome (§4.1).
2. First use of a term of art (*Sample*, *u-space*, *pool*, *bucket*, *pack*…) is
   italicised, defined in one line, and cross-referenced to its defining chapter.
3. One analogy per concept, marked as such, always *accompanying* the precise
   statement, never replacing it. No analogies inside reference tables.
4. Layer, don't duplicate: recipe first, precise contract second, internals linked
   ("the machinery is Chapter N").
5. **Reference material is a table or list, never buried in prose.** If a maintainer
   would grep for it, it gets a row.

---

## 3. Chapter types and their templates

Every chapter belongs to exactly one type. When writing or restructuring, follow the
skeleton for its type. (Existing exemplars are from the V2 book.)

### T1 — Tutorial (e.g. ch02 Installation, ch03 Quick Start)
```
\newthought opener: audience + outcome + time budget
"Before you start" preconditions (checkable, one command each)
Numbered procedures, one action per step, expected output stated
  each step's listing: shellbox (commands) / yamlwin (cards)
The mental model the tutorial just demonstrated (one section, with figure)
"Where to go next" closing list
```
Every listing must be runnable exactly as printed, against a documented starting
state. A tutorial teaches ONE golden path; alternatives get one pointer, not
parallel treatment.

### T2 — Concept (e.g. ch01 Introduction, ch04 Core Concepts, ch11 Workflow Model)
```
\newthought opener: the N ideas this chapter defines
One section per idea: definition → consequence → where it surfaces in YAML/outputs
Margin figure for any loop or pipeline
Closing jtip that re-binds the ideas to the reader's next chapters
```
Concept chapters define each term ONCE for the whole book. Other chapters link here
instead of re-defining.

### T3 — Block reference (e.g. ch06–ch10, ch12–ch15: one YAML block each)
```
\newthought opener: what the block controls, minimal card fragment
Complete key table (Key / Default / Meaning), fixed-width per §7.4
One section per non-trivial key group: rule → example → edge cases
Failure modes section: what breaks, the EXACT message, where the log is
Closing summary element
```
Every key row carries its real default and marks required keys. Aliases are listed
with the canonical form first. "What is deliberately absent" sections (ch06 pattern)
are encouraged — they stop readers hunting for keys that don't exist.

### T4 — Method/sampler (ch17–ch24)
```
\newthought opener: who needs this method, what it produces
"How it behaves in the runtime" — grounded in the sampler's source file
YAML block: minimal card, then full key reference table
Checkpoint/resume + determinism statement (UUID minting policy!)
Reading the log: real excerpts (plaincode), field-by-field
closing \section{Summary}: prose recap + one field/value table
  (rows: proposes, feedback?, determinism, checkpoint, budget keys, chapter refs)
```
The Summary-section-plus-table closing is mandatory for method chapters — it is the
distributed replacement for a comparison appendix, and the set of summaries must
stay mutually consistent (same row vocabulary across all method chapters).

### T5 — Delta (e.g. ch23 Dynesty over ch22 MultiNest)
```
Opener states the base chapter and that ONLY differences follow
Sections mirror the base chapter's order, covering only what changed
A "decision" table: when to prefer this variant over the base
```
A delta chapter may assume its base chapter; nothing else may be assumed. If the
delta grows past ~40% of its base's length, it has stopped being a delta —
restructure.

### T6 — Operations (ch25–ch30: CLI, checkpoint, outputs, monitoring, logging)
```
Opener: the operational task ("watch a running scan", "read what a run wrote")
Organised by the reader's situation, not by internal module
Real console/log excerpts (plaincode), annotated field by field
Post-mortem recipes as numbered steps
```
CLI-type content uses one `description`-list item per command/flag — never a table
(command sets grow; tables don't scale). This is a standing rule: **any enumeration
whose row count will grow over releases is a list, not a table.**

### T7 — Internals (ch31, ch32, ch35)
```
Opener says explicitly: nothing here is needed for normal use
Process/state tables; failure semantics list with per-actor bullets
Each claim links back to the user-visible consequence it explains
```

### Appendices
**Never put a feature-summary table in an appendix** (user rule, 2026-07-25) — any
table enumerating what the software *offers* (keys, flags, error codes, methods)
belongs in a chapter, however cross-cutting it is. This supersedes the earlier
carve-out for "master indexes": the YAML key index now lives in ch05, and the
validation-code table with it. An appendix may only hold material organized around
the reader's *situation* rather than the product's surface — a glossary, a
symptom→fix troubleshooting list, a gallery of worked cards. A chapter must remain
complete if its appendix vanished. At most one appendix pointer per chapter body,
never to complete an in-progress thought.

---

## 4. The frame (总—分—总)

### 4.1 Opener (总)
A `\newthought` paragraph, ≤ 2 paragraphs before the first `\section`, answering:
who is this for, what will they be able to do after, and (tutorials) how long it
takes. Check: `head -8 chapter.tex | grep newthought` must hit.

### 4.2 Body (分)
Sections ordered by reader need (most-used first), never by internal architecture.
One idea per section. State the rule first, then the example, then edge cases —
never make the reader reverse-engineer the rule from the example.

### 4.3 Closer (总)
A chapter must not end on a detail. Acceptable closers: `jtip` takeaway box,
"Where to go next" list, recap/Summary table, or (block-reference chapters) a
"deliberately absent" list. Check: the last 15 lines contain one of these.

---

## 5. Language

### 5.1 Sentence mechanics
- Active voice, present tense. Passive only when the actor is truly irrelevant.
- Target ≤ 25 words/sentence; split anything with two subordinate clauses.
- One paragraph = one point; ≤ 6 lines in the narrow tufte column.
- Procedures with ≥ 2 steps are `enumerate` lists; each step starts with an
  imperative verb and contains exactly one action; expected result ends the step.

### 5.2 Naming
- One concept, one name, book-wide, per the glossary appendix: *task card* (not
  config/YAML file), *Sample* (capitalised, the unit of work), *Worker*, *Archiver*,
  *observables*, *u-space*, *pack*, *bucket*, *pool*. New terms must be added to the
  glossary in the same change that introduces them.
- Method names always via `\sampler{}`; the canonical spelling comes from
  `Sampling.Method`'s registration, aliases mentioned once in that method's chapter.

### 5.3 Banned and restricted words
Banned in Contract/Behavior sentences (§1.3): *should work*, *usually*, *typically*,
*may want to*, *etc.*, *and so on*, *simply* (as filler), *of course*.
Restricted: *usually*/*typically* are tolerated **only** inside explicitly-marked
heuristic content — `jtip` advice, troubleshooting-table diagnosis columns — and even
there, prefer naming the condition ("when the posterior is unimodal…") over hedging.
Check: `grep -n "should work\|usually\|typically\|may want to\| etc\." chapters/*.tex`
and justify every survivor as heuristic-context or rewrite it.

### 5.3a No changelog in the manual (hard rule)

**These books document what the software does today, for a reader using it today. They
are not release notes.** Never write about a flag, key, or behaviour that no longer
exists — not to explain why it went away, not to reassure the reader it is gone, not as
a contrast to make the current design look better. A reader who has never seen
`--force` learns only that there might be a `--force`; a reader who has is not helped by
a eulogy. Describe the current behaviour and stop.

Banned constructions: *there used to be*, *no longer exists*, *was removed/dropped/
retired*, *unlike earlier releases*, *in V1 this was*, *now means*, *the flag went away*.
Also banned: framing a section around an absence ("One knob, not two") rather than
around the thing itself ("Deciding how much to watch").

The single exception is **live back-compatibility a reader can still hit**: a legacy
spelling that still works, or an old card that still loads. Document those as *current
supported behaviour* ("`Jarvis2 <task>.yaml` routes to `run`"), never as history.

Check — every hit must be either that exception or a rewrite:
```
grep -n "used to\|no longer\|not any more\|went away\|dropped\|retired\|Unlike earlier\|previously\|in V1\|now means" chapters/*.tex appendix/*.tex
```

### 5.4 Cross-reference budget
Cross-references are load-bearing, but a sentence with ≥ 3 `\ref`s is a link farm —
restructure into a list where each item carries one ref. A body paragraph should not
need more than ~3 refs total. Never reference forward to a concept mid-sentence when
a five-word inline definition would do.

### 5.5 Numbers and defaults
Every key mention states its default at first reference in the chapter, inline —
"(default 16)" — sourced per §1. Units always stated (seconds, points, MB).

---

## 6. Semantic markup (never raw `\texttt` in prose)

| Macro | Use for |
|-------|---------|
| `\yamlkey{}` | YAML keys and dotted key paths (renders in the listing key colour) |
| `\ykey{}` / `\yvalue{}` | a YAML key / value quoted in prose that must visually match the listing colours |
| `\cli{}` | shell commands and fragments mentioned mid-sentence — renders as a breakable, highlighted "terminal chip" (jShell slate text + background, via `soul`'s `\hl{}`); safe in prose, `keylist`/`description` items, and table cells |
| `\clih{}` | the **same content**, plain (no highlight) — **required instead of `\cli{}`** inside `\section{}`/`\chapter{}`/`\caption{}` (any moving argument): `soul`'s `\hl{}` hard-crashes there (`Argument of \let has an extra }`) |
| `\code{}` | generic inline code, class names, values |
| `\file{}` | file and directory paths — **never inside `\caption`/`\section`/`\chapter`** (xstring crash; use `\code{}` there) |
| `\token{}` / `\marker{}` | runtime tokens (`@SampleID`) / path markers (`&J/`) |
| `\sampler{}` | sampler method names |
| `\expr{}`, `\variable{}`/`\func{}`, `\mvar{}`/`\mfunc{}` | expressions; variable/function colouring in code and math contexts |
| `\req` / `\opt` | required/optional tags in key tables |
| `\Jtwo`, `\Jarvis`, `\Portal`, `\Operas`, `\JPlot`, `\Jversion` | product names and the single-sourced version |

Long-token line breaking: `\jtt`-based macros break **only at escaped underscores**.
For long dotted/slashed tokens that overflow, insert `\allowbreak{}` (always WITH the
empty braces) after `.` or `,` inside the argument; for `/` between two tokens,
prefer a real space around the slash. Never patch `\jtt` itself.

---

## 7. Visual elements

### 7.1 Element selection

| Content | Element |
|---------|---------|
| Runnable YAML (cards, fragments) | `yamlwin` (titled GUI-window box; optional `[filename]` argument) |
| Anything that happens in a terminal — commands, program/log output, or both | `terminal`, with `jcmd` for lines you **type** and `jout` for lines a program **printed** (see §7.2a) |
| Flow or loop | `marginfigure` stack diagram or one-line boxed pipeline |
| Decision ("which X?") | checklist `enumerate` (first match wins) or two-column decision table |
| Key/default/meaning | `booktabs` table per §7.4 |
| Directory layout | `jfiletree` + `\dirtree` with `\DTcomment` |
| Caveat/contract/advice | `jwarn` / `jnote` / `jtip` (descending severity) |

`yamlcode`/`bashcode`/`shellbox` are retired; do not introduce new uses. `plaincode`
survives for listings that are *not* terminal content (pseudocode, schema dumps).

### 7.2a The `terminal` box — one chrome, three shapes

All terminal content goes in `terminal`. Inside it, `jcmd` marks what the reader
**types** (jShell slate, Jarvis executables in orange) and `jout` marks what a program
**printed back** (the quieter jOut slate, no command highlighting). `\termsep` between
them draws the faint dashed divider:

```latex
\begin{terminal}                  \begin{terminal}              \begin{terminal}
\begin{jcmd}                      \begin{jout}                  \begin{jcmd}
Jarvis2 ps                        [Scan Performance]            Jarvis2 ps
\end{jcmd}                          wall_time_sec : 1284.3      \end{jcmd}
\end{terminal}                    \end{jout}                    \termsep
                                  \end{terminal}                \begin{jout}
  commands only                     output only                 Running ... (6):
                                                                \end{jout}
                                                                \end{terminal}
                                                                  both
```

Rules:
- **Prefer the paired form** whenever you are showing a command *and* what it printed.
  It is the shape a reader recognises from their own terminal, and it removes the
  "which output belongs to which command" ambiguity that two separate boxes create.
- Never write a shell prompt (`$ `) into `jcmd` — the box already says it is a terminal,
  and the divider already says where the reply starts.
- `[Title]` renames the title bar (default `Terminal`) for a named artifact rather than
  a session.
- Non-ASCII bytes (±, ·, •) only work where the style's `literate=` table maps them —
  extend the table in **both** `shellinner` and `termout` before quoting a new byte.
- Both inner styles must stay self-contained: the document-level `\lstset{style=jbase}`
  is ambient, so any decoration key they do not override (`frame`, `numbers`,
  `backgroundcolor`, `xleftmargin`) leaks in and paints over the box's own chrome.

### 7.2 Listings
Every listing is either complete and runnable as printed, or first line carries
`# excerpt`. YAML in listings must be *actually valid* for the current schema —
listings are part of the tested surface, conceptually.

### 7.3 Boxes
≤ 3 boxes per chapter. A `jwarn` must describe a real loss (data, time,
correctness) with the actual failure text; otherwise it is a `jnote`. Margin space
reinforces; it never carries normative content that exists nowhere else.

### 7.4 Tables — two widths, fixed columns (unchanged from v1)
Every table is **Main** (plain `table`, tufte body width) or **Full**
(`fullwidth`-wrapped). Every column is `p{<fraction>\linewidth}` from the fixed
palette; no bare `l`/`c`/`r` holding more than a few characters.

| Columns | Main split | Full split |
|---|---|---|
| 2 | 0.32 / 0.62 | 0.30 / 0.66 |
| 3 | 0.24 / 0.20 / 0.50 | 0.22 / 0.18 / 0.56 |
| 4 | 0.30 / 0.16 / 0.16 / 0.28 | 0.28 / 0.14 / 0.14 / 0.40 |
| 5–6 | switch to Full | 0.10 first + 0.18 each |

When fractions sum ≥ ~0.90, wrap the tabular in
`\begingroup\setlength{\tabcolsep}{0pt}…\endgroup` (internal gaps otherwise
overflow deterministically). Growing enumerations are lists, not tables (§T6).

### 7.5 Figures and captions
- Figures are referenced from the text and captioned with what to *see*, not what
  they are.
- **Margin-caption length budget**: a tufte figure caption longer than ~12 printed
  margin lines risks colliding with the next section's content — such collisions
  produce **no compiler warning** and are only caught visually (§8 stage V5).
  Long explanations belong in body prose; the caption keeps the terse mapping.
- Any edit that can move a float (adding paragraphs nearby) re-triggers V5 for the
  affected pages.

---

## 8. Verification protocol (mandatory, ordered)

**Agents run V1–V2 only.** Compiling is the user's job (§0 rule 5): V3–V5 below are
recorded so the *user* knows what a full check involves, and so the failure classes
they catch stay documented — an agent never executes them and never reports their
results. This makes V1's mechanical checks the last line of defence, so run them
properly rather than skimming.

**V1 — structural checks** (always, even when compiling is forbidden). Don't use
bare `grep -c` for counts you'll compare — see the V4 note on why; `grep -n` (no
`-c`) is fine, since a real zero-match search legitimately prints nothing:
```
# balanced environments touched by the change -- compare the two numbers you get:
grep -o 'begin{yamlwin}' f.tex | wc -l ; grep -o 'end{yamlwin}' f.tex | wc -l
# bare \allowbreak before a letter (gets swallowed into an undefined control seq):
grep -n '\\allowbreak[a-zA-Z]' chapters/*.tex
# \file{} in moving arguments:
grep -n 'caption.*\\file{\|section.*\\file{\|chapter.*\\file{' chapters/*.tex
```

**V2 — content self-review**: re-read the diff against §1 (every claim sourced?),
§3 (template satisfied?), §5.3 (banned words?), §7 (right element per content?).

**V3 — compile** *(user only; never run by an agent)*: `latexmk -pdf -interaction=nonstopmode -halt-on-error main.tex`.
Required: exit 0.

**V4 — log checks** *(user only)*: do these with Python, not bare `grep -c` — this sandbox's
`grep` wrapper has been observed to print *nothing at all* (not even `0`) on some
non-trivial match counts, which silently gets misread as "zero" if you trust empty
output. Always use a direct string count instead:
```python
import re
content = open('main.log', encoding='utf-8', errors='replace').read()
print('fatal errors:', content.count('\n!'))                    # must be 0
refs = re.findall(r'.*(Reference|Citation).*undefined.*', content)
print('broken refs/citations:', len(refs))                       # must be 0
```
Fatal errors and broken refs/citations must be exactly 0 — that genuinely is this
book's baseline. **Overfull/Underfull are NOT zero at baseline** (this book runs
~230+ Overfull / ~2500+ Underfull just from ordinary tufte narrow-column
typesetting — do not write or trust a report claiming "0 Overfull/Underfull", it
has never been true and signals the check was broken, not the book). The
actionable check is whether your change added a *new* Overfull >15pt (the
long-established severity floor — smaller ones are cosmetic noise):
```python
cur  = set(round(float(v),2) for v in re.findall(r'Overfull \\hbox \(([\d.]+)pt too wide\)', content) if float(v) > 15.0)
base = set(round(float(v),2) for v in re.findall(r'Overfull \\hbox \(([\d.]+)pt too wide\)', open('baseline.log', encoding='utf-8', errors='replace').read()) if float(v) > 15.0)
print('NEW overfull >15pt:', cur - base)   # must be empty
```
This needs a `baseline.log` from the last known-good compile (keep one around);
without it, at minimum eyeball every `Overfull >15pt` hit's surrounding file/lines
and judge whether it's plausibly pre-existing (old chapter, unrelated content) or
newly caused by your edit.

**V5 — visual check (rasterize)** *(user only)*: render every page whose content the change
touched or could have shifted (PyMuPDF: `fitz.open(...)[p].get_pixmap(dpi=150)`),
and *look at them* for the failure classes that emit no warning:
- listing boxes that split across pages losing colour/decoration;
- margin captions colliding with body/heading/table content;
- floats landing pages away from their reference;
- table rows visually cramped or rules colliding.

An agent's change is "done" after V1–V2, and its report must say plainly that it
was not compiled. Never let a hand-off imply a compile happened.

---

## 9. Self-review rubric

Before handing back, grade the change; anything at Blocker/Major gets fixed first.

**Blocker** — factual claim not verified against source; listing that cannot run as
printed; broken/undefined `\ref`; new compile warning; V5 visual defect.
**Major** — template section missing for the chapter type (no failure-modes section
in a block reference; no Summary table in a method chapter); banned word in a
contract sentence; key row without default; table off-palette; box budget exceeded;
appendix pointer that completes an in-progress thought.
**Minor** — sentence > 25 words; ref-farm paragraph; missing italics-on-first-use;
analogy without its precise statement.

---

## 10. Change discipline (renames, renumbers, migrations)

- **Renames** must be searched in BOTH forms: LaTeX-escaped (`adaptive\_level\_set`,
  prose/tables) and raw (`adaptive_level_set`, inside verbatim listings). One regex
  never catches both.
- **Chapter renumbering**: `mv` in an order that never collides with a
  yet-to-be-created filename; update `main.tex` includes in the same pass; keep old
  `\label` names (labels are invisible plumbing) unless a label's *meaning* changed;
  re-derive every external `\ref` to a moved/split label from its surrounding prose,
  never blind-replace.
- **Environment/style migrations** are scripted (Python string replace over files),
  verified by before/after counts, then compiled — never hand-edited per file.
- The version banner and `\JversionNum` are single-sourced in the preamble; never
  hard-code a version number in prose.

---

*Reference implementations:* ch01 (concept opener + figure captioning), ch02 (T1),
ch04 (T2), ch12 (T3 incl. wart documentation), ch22 (T4 at full depth), ch23 (T5),
ch28/ch31 (T6/T7), ch06's "What is deliberately absent" closer.
