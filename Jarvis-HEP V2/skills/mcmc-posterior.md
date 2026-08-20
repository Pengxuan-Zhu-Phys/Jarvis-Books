---
name: mcmc-posterior
title: 用 MCMC 做后验扫描
intent: "我想要参数的后验分布 / 置信区间，用 MCMC 扫"
triggers: [MCMC, DRAM, 后验, posterior, 置信区间, 链]
level: intermediate
verified: 2026-07-21 @ jarvis2/2daf417
---

# 用 MCMC 做后验扫描

> 当前 D13 实现可运行，但 MCMC 的严格 YAML schema、完整状态轨迹和部分算法修正仍按
> [`../DESIGN_MCMC_ARCHITECTURE_2.0.md`](../DESIGN_MCMC_ARCHITECTURE_2.0.md) 2.1
> 方案实施中。当前 `chain_history.csv` 是诊断导出，不要把它误当成已经完成校正的
> authoritative posterior trace。

## 目标

用 DRAM（自适应 + 延迟拒绝，最皮实的单链族方法）跑出后验样本，
拿到接受率 / R-hat / ESS 诊断和逐链历史。

## 前提

- 跑通过 [first-scan](first-scan.md)；LogL 表达式已经能算
- 明白：MCMC 的并行单位是**链**，`chains` 至少设成 Worker 数

## 复制即用

把 first-scan 卡片的 `Sampling` 换成：

```yaml
Sampling:
  Method: "DRAM"
  Variables:
    - name: x
      distribution: {type: Flat, parameters: {min: 0.0, max: 5.0}}
    - name: y
      distribution: {type: Flat, parameters: {min: 0.0, max: 5.0}}
  Bounds:
    seed: 21
    num_chains: 8        # 链数；≥ workers，否则 Worker 闲置
    num_iters: 2000      # 每链迭代数
    proposal_scale: 0.1  # 初始高斯提议尺度（单位立方体坐标）
    dr_steps: 2          # DRAM 延迟拒绝级数
  LogLikelihood:
    - {name: "LogL_Z", expression: "LogGauss(z, 100, 10)"}

EnvReqs:
  V2:
    workers: 4
```

```bash
Jarvis2 run mcmc_card.yaml
```

## 它做了什么

8 条链各自提议一个点 → 一批发给 Worker 算 LogL → 屏障处逐链接受/拒绝 →
下一代。被拒绝的链在同一代内做 DRAM 第二级提议。每个代际屏障自动存 checkpoint，
`--resume` 可续。轨迹与 Worker 数无关（换机器复现相同）。

## 结果在哪

- `DATABASE/sampler_summary.json` — 每链接受率、R-hat、`ess_logl`
- `DATABASE/chain_history.csv` — `chain_id, step, accepted, weight, logl` 逐行
- `DATABASE/samples.hdf5 / samples.csv` — 全部求值点（含 burn-in，自己按 step 截）

## 常见改动

| 想要…… | 改这里 | 详见 |
|---|---|---|
| 简单 Metropolis / 只要自适应 | `Method: MCMC` / `AM` | YAML_REFERENCE §6.11 |
| 多 Worker 吃满、目标多峰 | `Method: EnsembleMCMC`（集合式，天然并行） | YAML_REFERENCE §6.12 |
| 强多峰 | `Method: PT`（并行回火） | YAML_REFERENCE §6.12 |
| 调自适应窗口 | `Bounds.adapt.start_iter / window / scale` | YAML_REFERENCE §6.11 |

## 常见坑

- **接受率≈0 或≈1** → `proposal_scale` 太大/太小 → 调一个量级；AM/DRAM 会自适应，
  但起点太离谱仍会浪费 burn-in。
- **R-hat 明显 >1.1** → 没收敛 → 加 `num_iters`，或换 EnsembleMCMC/PT。
- **Worker 大量 idle** → `num_chains < workers` → 加链数。
