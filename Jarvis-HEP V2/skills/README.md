# Jarvis-HEP V2 · Skills 文档库

**一句话一个技能：找到你想做的事，复制那个最小 YAML，跑起来。**

这不是参考手册（那是 [`../YAML_REFERENCE_2.0.md`](../YAML_REFERENCE_2.0.md)）。
每个 skill 只回答一个用户意图，给一张**复制即可运行**的最小卡片，
高级选项按需展开。看完"复制即用"一节就可以停——剩下的是你需要时才读的。

## 我想……

| 我想…… | Skill | 难度 |
|---|---|---|
| 第一次跑通一个扫描，确认安装没问题 | [first-scan](first-scan.md) | ⭐ |
| 知道该选哪个采样算法 | [choose-sampler](choose-sampler.md) | ⭐ |
| 接入我自己的计算程序（SPheno、micrOMEGAs、自写脚本…） | [external-calculator](external-calculator.md) | ⭐⭐ |
| 首次构建/复用多个共享第三方库 | [shared-libraries](shared-libraries.md) | ⭐⭐ |
| 用 MCMC 做后验扫描 | [mcmc-posterior](mcmc-posterior.md) | ⭐⭐ |
| 算贝叶斯 evidence（logZ，模型比较） | [nested-evidence](nested-evidence.md) | ⭐⭐ |
| 扫描完了，找到我的结果 | [find-your-results](find-your-results.md) | ⭐ |
| 快速出图 | [plot-your-scan](plot-your-scan.md) | ⭐ |
| 断点续跑 / 中断后善后 | [resume-and-recover](resume-and-recover.md) | ⭐ |
| 看懂报错并修好它 | [fix-common-errors](fix-common-errors.md) | ⭐ |

## 约定

- 每个 skill 的 YAML 卡片都在**提交前于当前分支实测验证**，frontmatter 里的
  `verified` 是最后验证日期。卡片跑不通 = bug，和测试挂了同级。
- 想深入某个键的完整语义 → 跟着 skill 内的链接进 `YAML_REFERENCE_2.0.md`。
- 新增 skill：复制 [`_TEMPLATE.md`](_TEMPLATE.md)，遵守"一个意图、一张能跑的卡、
  先复制后解释"三原则。维护规则见
  [`../DESIGN_SKILLS_LIBRARY_2.0.md`](../DESIGN_SKILLS_LIBRARY_2.0.md) §5。
