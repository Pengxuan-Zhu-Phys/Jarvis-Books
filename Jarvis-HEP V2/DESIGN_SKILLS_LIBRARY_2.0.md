# DESIGN — User Skills Library (V2, D16)

**Status**: design accepted 2026-07-21; **v1 of the library ships with this design**
(under [`skills/`](skills/)); packaging/CLI WPs `todo`
**Date**: 2026-07-21 (maintainer: "许多 YAML 设置过于复杂，是让用户止步的第一阻碍——
需要 skill 文档库")
**Scope**: a library of small, task-shaped documents — one user intent per file, one
**copy-paste-runnable minimal YAML** per intent, complexity revealed only on demand.

---

## 1. Problem

V2's YAML surface is now large and precise: 13+ sampling methods each with their own
`Bounds` vocabulary, `EnvReqs.V2` grouped settings, Calculators with clone_shadow /
tokens / Portal IO types, Operas expressions, plot scenes. The reference docs
(`YAML_REFERENCE_2.0.md`, ~50 KB) are **complete but completeness is the problem**: a
new user wanting "scan my model with MCMC" faces hundreds of keys before their first
successful run. The maintainer identifies this as **the #1 adoption barrier**.

The existing mitigations are necessary but not sufficient:

- `project_template/bin/sampling/*_Simple.yaml` (D13.10) — good minimal cards, but only
  for nested methods, and templates answer "what do I type", not "which method do I
  want" or "why did it fail";
- `Jarvis2 validate` (D13.9) — catches mistakes early but after the user already wrote
  YAML;
- `YAML_REFERENCE` — a dictionary, not a phrasebook.

## 2. Design principles

1. **One intent per skill.** A skill answers exactly one user sentence ("我想算
   evidence"), not a topic ("nested sampling").
2. **The minimal card must run as-is.** Every YAML block in a skill is verified against
   the current branch before the skill is committed; the skill records the verification
   date. A skill with a broken card is a bug (same severity as a failing test).
3. **Progressive disclosure.** Body order is fixed: 目标 → 前提 → 复制即用 → 它做了什么
   (≤5 行) → 常见改动 (每行一个知识点，指向 YAML_REFERENCE 章节) → 常见坑. The user can
   stop reading after "复制即用" and still succeed.
4. **Skills point down, references point everywhere.** Skills link into
   `YAML_REFERENCE`/design docs for depth; reference docs stay exhaustive and are not
   duplicated. A skill never explains architecture.
5. **Machine-consumable header.** Every skill carries YAML frontmatter
   (`name`/`title`/`intent`/`triggers`/`level`/`verified`) so a future `Jarvis2 skill`
   CLI and the (parked) Agent bridge can index them without parsing prose.
6. **中文正文，英文键名。** Prose in Chinese (the user base), YAML/commands verbatim
   English.

## 3. v1 contents (shipped with this design)

| Skill | Intent |
|---|---|
| [`first-scan`](skills/first-scan.md) | 第一次跑通一个最小扫描，验证安装 |
| [`choose-sampler`](skills/choose-sampler.md) | 13 个采样方法里我该用哪个 |
| [`external-calculator`](skills/external-calculator.md) | 接入我自己的外部计算程序 |
| [`mcmc-posterior`](skills/mcmc-posterior.md) | 用 MCMC/DRAM 做后验扫描 |
| [`nested-evidence`](skills/nested-evidence.md) | 用嵌套采样算 evidence (logZ) |
| [`find-your-results`](skills/find-your-results.md) | 扫描完了，结果都在哪 |
| [`plot-your-scan`](skills/plot-your-scan.md) | 快速出图 + 常见微调 |
| [`resume-and-recover`](skills/resume-and-recover.md) | 断点续跑与善后 |
| [`fix-common-errors`](skills/fix-common-errors.md) | 常见报错急救表 |

Index: [`skills/README.md`](skills/README.md) · authoring template:
[`skills/_TEMPLATE.md`](skills/_TEMPLATE.md).

## 4. Work packages

| WP | Title | Depends on | Accept |
|---|---|---|---|
| D16.1 | Skills library v1 (this design; 9 skills + index + template) | — | **done with this commit** — every YAML card verified on current branch; index lists intent sentences |
| D16.2 | Ship skills in-package + `Jarvis2 skill list / show <name>` | D16.1 | skills copied under `jarvishep2/skills/` at release; CLI prints index / one skill (plain text, no pager games); `Jarvis2 skill` appears in `--help` |
| D16.3 | Card CI: extract every ```yaml block from skills, run `Jarvis2 validate --strict` on complete task cards | D16.2 | a broken skill card fails the test suite; partial snippets markable as `no-validate` |
| D16.4 | Coverage growth pass: calculator+SLHA skill, EnvReqs tuning skill, cluster skill (after D14), analyze skill (after D15.2) | D16.3 | each new feature milestone adds/updates its skill as part of its own acceptance |

**Rollback**: docs-only in v1; D16.2 CLI is additive. **Out of scope**: web rendering,
i18n beyond zh/en mixed, agent integration (parked with D8).

## 5. Maintenance rule (binding, added to plan protocol)

From D16.4 on, **a user-facing WP is not `done` until the affected skill exists or is
updated** — same standing as YAML_REFERENCE updates. Skills are cheap; stale skills are
worse than none.
