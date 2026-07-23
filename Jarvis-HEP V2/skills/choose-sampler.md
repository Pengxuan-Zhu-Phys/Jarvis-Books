---
name: choose-sampler
title: 该选哪个采样算法
intent: "V2 有十几个 Sampling.Method，我该用哪个？"
triggers: [采样方法, 选算法, Method, MCMC还是嵌套, 决策]
level: beginner
verified: 2026-07-21 @ jarvis2/2daf417
---

# 该选哪个采样算法

## 目标

30 秒定位适合你问题的 `Sampling.Method`，然后跳到对应 skill 或模板。

## 先回答一个问题：你要的是什么？

| 你要的是…… | 用这类 | Method 名 |
|---|---|---|
| **把参数空间均匀看一遍**（画热图、摸地形、初次探索） | 固定点集 | `Random`（最简单）/ `Bridson`（更均匀，推荐）/ `Grid`（规则网格） |
| **重算一批已有的点**（别人给的点表、上次扫描的子集） | 点表回放 | `CSV` |
| **找等值线 / 排除边界**（如 Ωh² = 0.12 的等值面，2–5 维） | 自适应加密 | `AdaptiveBridson` |
| **后验分布**（参数的置信区间、相关性） | MCMC | `MCMC` / `AM` / `DRAM`（单链族）；`EnsembleMCMC` / `DEMCMC`（集合式，多 Worker 更划算）；`PT` 系（多峰问题） |
| **贝叶斯 evidence logZ**（模型比较） | 嵌套采样 | `Dynesty`（动态，posterior+evidence 都好）/ `MultiNest`（静态，V1 同名兼容） |

## 快速建议（如果你不想细想）

1. **第一次用 / 探索新模型** → `Bridson`（见 [first-scan](first-scan.md)）
2. **要置信区间** → `DRAM`，链数 ≥ Worker 数（见 [mcmc-posterior](mcmc-posterior.md)）
3. **要 evidence** → `Dynesty`（见 [nested-evidence](nested-evidence.md)）
4. **要某条实验界线的形状** → `AdaptiveBridson`
5. 多峰 / 简并结构明显 → 集合式（`EnsembleMCMC`）或 `PT`

## 几个事实，帮你避免误选

- **`MultiNest` 不是 Fortran MultiNest**：V1/V2 里它一直是静态嵌套采样引擎
  （dynesty NestedSampler），只是沿用了老名字。要动态就用 `Dynesty`，两者不用切开关。
- **MCMC 的并行单位是"链"**：`chains < workers` 时多余的 Worker 会闲着。
  集合式方法（EnsembleMCMC/DEMCMC）天然并行度更高。
- **固定点集类没有反馈**：`Random`/`Bridson`/`Grid`/`CSV` 不会根据 LogL 调整撒点，
  纯地毯式；要"越算越聪明"就得选后三类。
- 每个方法的完整键位表：`YAML_REFERENCE_2.0.md` §6；嵌套采样官方模板：
  `project_template/bin/sampling/Sampling_{Dynesty,MultiNest}_{Simple,Full}.yaml`。
