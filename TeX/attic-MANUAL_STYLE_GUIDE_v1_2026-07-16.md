# Jarvis-Books — Technical Manual Writing Guide

**Scope**: the LaTeX user & reference manuals under `Jarvis-Books/TeX/`
(`Jarvis-HEP manual`, `Jarvis-HEP-V2 manual`, and any future book).
**Status**: adopted 2026-07-16. New chapters must follow it; existing chapters are
migrated opportunistically (V2 book: Chapter 1 is the reference implementation).
**Language**: manuals are written in English; this guide applies to all of them.

The guide rests on four principles: **know the audience**, **fixed logical frame**,
**plain and unambiguous language**, and **show, don't only tell**. Each section below
turns one principle into concrete, checkable rules.

---

## 1. Audience first

Every book serves two readers at once. Never write a paragraph without knowing which
one it is for.

| Reader | Wants | Gets |
|--------|-------|------|
| **User** (physicist running scans; may be new to the tool) | "how do I do X", minimal jargon, working recipes | Parts *Getting Started* → *Running & Outputs*; every task explained as numbered steps with a copy-paste example |
| **Power user / maintainer** | complete key reference, architecture, failure semantics, tuning | The reference tables in each chapter, the *Advanced* part, and the appendices |

Rules:

1. **Chapter openers state the audience and the outcome.** The first paragraph answers:
   who needs this chapter, and what will they be able to do after reading it.
2. **User-path chapters (Parts I, V) introduce every term before using it.** A term of
   art (Sample, u-space, pool, bucket…) appears in *italics* with a one-line plain
   definition at first use, plus a pointer to the defining chapter.
3. **Analogies are allowed on the user path, never in reference tables.** One analogy
   per concept, marked as such ("think of pools as parking permits"), and it must not
   replace the precise statement — it accompanies it.
4. **Reference material is never hidden inside prose.** If a maintainer would want to
   look it up, it is a table (keys, defaults, exit codes), not a sentence.
5. **Do not duplicate content for the two audiences.** Layer it: recipe first, precise
   contract second, internals linked ("the machinery is Chapter N").
6. **A chapter is self-contained; appendices are optional, not required reading.**
   Every table a reader needs to *use* the feature the chapter is about goes *in* the
   chapter, in full — never "see Appendix X" as the only place a key/table/code list
   actually lives. An appendix earns its place only when it serves a genuinely
   different use case than any one chapter (a print/grep-friendly master index
   spanning many chapters — the full YAML key index, the full CLI flat table), and
   even then the source chapter must already contain everything needed to use that
   feature without the trip. Before writing "Appendix~\ref{...}" in chapter prose, ask:
   if this appendix vanished, would the chapter still be complete? If not, that content
   belongs in the chapter, not the appendix. Minimize appendix pointers in body prose
   generally — one at most per chapter, only for a reader who wants the exhaustive
   cross-chapter view, never to complete an in-progress thought.

## 2. One logical frame (总—分—总)

### Book level

The fixed arc is: install → quick start → concepts → reference → operations →
advanced → appendices. Do not insert chapters that break the arc; new material lands
in the part whose question it answers:

| Part | Question it answers |
|------|--------------------|
| Getting Started | "What is this and how do I run something *today*?" |
| Task Card / Reference | "What exactly can I write, and what does it do?" |
| Backends / Samplers | "How do I express *my* workflow / choose *my* method?" |
| Running & Outputs | "How do I operate, watch, and read a scan?" |
| Advanced | "How does it work inside; how do I make it fast / reproducible?" |
| Appendices | "Where is the flat lookup table?" |

### Chapter level

Every chapter is itself 总—分—总:

1. **Opening (总)**: a `\newthought` paragraph — what this chapter covers, for whom,
   and what you can do after it. No more than two paragraphs before the first section.
2. **Body (分)**: sections ordered from most-used to least-used feature (not by
   internal architecture). One idea per section.
3. **Closing (总)**: a summary element — a `jtip` box with the take-away, a "where to
   go next" list, or a recap table. A chapter must not end on a detail.

### Section level

State the rule first, then the example, then the edge cases. Never make the reader
reverse-engineer the rule from the example.

## 3. Language rules

1. **Active voice, present tense.** "The Worker pulls a task", not "a task is pulled".
   Passive is acceptable only when the actor is truly irrelevant.
2. **Short sentences.** Target ≤ 25 words; split anything with more than one
   subordinate clause. One paragraph = one point, ≤ 6 lines in the narrow column.
3. **Procedures are numbered lists.** Any "do this then that" with ≥ 2 steps becomes
   an `enumerate`. Each step starts with an imperative verb ("Run…", "Open…",
   "Check…") and contains exactly one action. Expected result goes at the end of the
   step ("…; the console prints `PONG`").
4. **No ambiguity words.** Ban: "should work", "usually", "may want to", "etc." in
   normative text. Say what happens, when, and what the default is.
5. **Consistent names.** One concept, one name, book-wide (see the glossary appendix).
   Never alternate between synonyms (task card / YAML file / config — pick *task
   card*).
6. **Semantic markup always.** `\yamlkey{}` for keys, `\cli{}` for commands,
   `\token{}`/`\marker{}` for tokens, `\file{}` for paths, `\sampler{}` for methods.
   Never raw `\texttt{}` in prose.
7. **Numbers and defaults are stated inline**, not "see the code". Every key mention
   carries its default on first reference in a chapter.

## 4. Visual elements

Text explains *why*; visuals carry *structure*. Choose deliberately:

| Content | Element |
|---------|---------|
| A flow or pipeline | margin diagram (`marginfigure`) or a one-line boxed pipeline |
| A decision ("which X do I use?") | a two-column decision table |
| Key/default/meaning reference | `booktabs` table, one row per key |
| Directory layouts | `jfiletree` + `\dirtree` with `\DTcomment` annotations |
| Runnable configuration | `yamlcode` listing, complete enough to copy-paste |
| Shell interaction | `bashcode` listing, with expected output as comments |
| Caveat / contract / advice | `jwarn` / `jnote` / `jtip` boxes (that order of severity) |

Rules:

1. **Every chapter has at least one non-prose element** per ~2 pages (table, listing,
   tree, or figure). Walls of text are a review failure.
2. **Listings must run.** Every YAML/shell listing is either complete and runnable or
   explicitly marked as a fragment (`# excerpt`).
3. **Figures are referenced from the text** (`Figure~\ref{…}`) and captioned with what
   to *see* in them, not just what they are.
4. **Boxes are rare and load-bearing.** ≤ 3 boxes per chapter; a `jwarn` must describe
   a real loss (data, time, correctness), or it is a `jnote`.
5. **Margin space is for reinforcement** (sidenotes, small figures), never for
   normative content that appears nowhere else.

### 4.1 Table widths — exactly two formats, always fixed columns

Every table in the book is one of exactly two widths. There is no third option, and
no table may use a bare `l`/`c`/`r` column for anything that can hold more than a
handful of characters — auto-width columns are how every table-overflow bug in this
book's history happened (a long `\yamlkey{}`/`\code{}` token has no break point at
commas or dots, only at underscores, so it forces the row past the margin instead of
wrapping).

| Format | Container | Total width | Use for |
|---|---|---|---|
| **Main** (default) | plain `table` | `\linewidth` (the tufte body measure) | the vast majority of tables: 2–4 columns, fits the narrow column |
| **Full** | `\begin{fullwidth}…\end{fullwidth}` around the `table` | the full text block (body + margin) | tables that are dense even after wrapping in Main — long key paths, ≥5 columns, or many long code tokens per row |

**Every column is `p{<fraction>\linewidth}`, chosen from this fixed palette** —
never a one-off percentage invented per table:

| Columns | Main-format split (of `\linewidth`) | Full-format split (of the wider `\linewidth` inside `fullwidth`) |
|---|---|---|
| 2 | `0.32` / `0.62` | `0.30` / `0.66` |
| 3 | `0.24` / `0.20` / `0.50` | `0.22` / `0.18` / `0.56` |
| 4 | `0.30` / `0.16` / `0.16` / `0.28` | `0.28` / `0.14` / `0.14` / `0.40` |
| 5–6 | avoid --- switch to Full | `0.18` per named column, first column `0.10` |

Pick the split that matches the column's *role*, not its content length: a
label/key column is the first (narrower) fraction, a short value/status column is
the middle fraction, and a prose/description column always gets the widest one.
When a table's natural shape doesn't fit 2–4 columns cleanly (a wide comparison
matrix), it goes to Full and keeps roughly even per-column fractions.

Before adding or editing any table: pick Main or Full, pick the split for its
column count from the table above, write every column as `p{…\linewidth}`, and
verify with an isolated `pdflatex` compile that no `Overfull \hbox` warning
appears for it (Section 4, rule 1 already requires the table to exist; this adds
that it must not overflow).

## 5. Chapter checklist (apply before committing)

- [ ] Opener names the audience and the outcome; ≤ 2 paragraphs before section 1.
- [ ] Every procedure is a numbered list of single-action imperative steps.
- [ ] Every new term is italicised + defined at first use, or cross-referenced.
- [ ] Every key/command mention uses semantic markup and states its default.
- [ ] At least one table/listing/diagram per ~2 pages; all listings runnable.
- [ ] Every table is Main or Full width (§4.1), every column is a fixed `p{}` from
      the standard split, no bare `l`/`c`/`r` holding more than a few characters.
- [ ] Ends with a summary element (tip box / next-steps list / recap table).
- [ ] No "should/usually/etc." in normative sentences; active voice throughout.
- [ ] Cross-references compile (`\ref` targets exist); terms match the glossary.

---

*Reference implementation:* `TeX/Jarvis-HEP-V2 manual/chapters/01-introduction.tex`
follows this guide; use it as the template when migrating other chapters.
