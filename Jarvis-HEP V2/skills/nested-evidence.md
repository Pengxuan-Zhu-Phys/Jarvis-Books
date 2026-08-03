---
name: nested-evidence
title: 用嵌套采样算 evidence
intent: "我要 logZ（贝叶斯 evidence）做模型比较，顺便要后验"
triggers: [evidence, logZ, 嵌套采样, dynesty, MultiNest, nested]
level: intermediate
verified: 2026-07-21 @ jarvis2/2daf417
---

# 用嵌套采样算 evidence

## 目标

拿到 `logZ ± err`、嵌套采样的完整死点表（带重要性权重）和 runplot。

## 前提

- 跑通过 [first-scan](first-scan.md)
- 知道两个方法名的真实含义：**`Dynesty` = 动态嵌套采样**（posterior 精度更好），
  **`MultiNest` = 静态嵌套采样**（V1 老名字，引擎同样是内置 dynesty，
  **不是** Fortran MultiNest）。选 Method 即选引擎，没有 dynamic 开关。

## 复制即用

官方模板就是最好的起点（脚手架项目里自带）：

```
project_template/bin/sampling/Sampling_Dynesty_Simple.yaml
```

核心就是把 `Sampling` 换成：

```yaml
Sampling:
  Method: "Dynesty"        # 要静态就写 MultiNest
  Variables:
    - name: x
      distribution: {type: Flat, parameters: {min: 0.0, max: 5.0}}
    - name: y
      distribution: {type: Flat, parameters: {min: 0.0, max: 5.0}}
  Bounds:
    Seed: 21
    nlive: 200             # live points：精度和耗时的主旋钮
    dlogz: 0.5             # 停止阈值：要发表级精度用 0.1
    run_nested:
      print_progress: true
  LogLikelihood:
    - {name: "LogL_Z", expression: "LogGauss(z, 100, 10)"}
```

```bash
Jarvis2 run nested_card.yaml
```

## 它做了什么

内置 dynesty 引擎驱动整个采样；每批 loglikelihood 求值通过 Redis 发给 Worker
（你的 calculator/Operas 流水线原样生效）。结束时 `logZ ± err` 打进
`run_summary` 和 sampler 日志，`Results.summary()` 全文在 `logs/<scan>/sampler.log`。

## 结果在哪

- `DATABASE/sampler_summary.json` — logZ、niter、ncall
- `DATABASE/dynesty_result.csv`（MultiNest 则是 `multinest_result.csv`）—
  逐死点：`log_weight, log_Like, log_Evidence, …`；**做后验要用 `log_weight` 加权**，
  直接把死点当后验样本是错的
- `images/<scan>/…_dynesty_runplot.yaml` — runplot 场景，`Jarvis2 plot` 渲染

## 常见改动

| 想要…… | 改这里 | 详见 |
|---|---|---|
| 更准的 logZ | `nlive: 500`、`dlogz: 0.1` | YAML_REFERENCE §6.10 |
| 采样器细节（bound/sample/walks…） | `Bounds.sampler: {...}`（官方 dynesty 键名） | 同上 |
| 限制总耗时 | `run_nested: {maxcall: 100000}` | 同上 |
| 批大小对齐 Worker | 默认取 `EnvReqs.V2.batch_size`，一般不用动 | §5 |

## 常见坑

- **写了 `Bounds.dynamic: true`** → 校验直接拒绝（`JV2-BND-012`）→ 删掉，
  换 Method 名即可。
- **nlive 太小**（<50）→ logZ 误差大且不稳定 → 别低于 100。
- **每点很贵、进度慢** → 嵌套采样的 ncall 常是 10⁴–10⁵ 量级，先用
  `Jarvis2 check` 测单点耗时估算总时长，再决定 nlive/dlogz。
