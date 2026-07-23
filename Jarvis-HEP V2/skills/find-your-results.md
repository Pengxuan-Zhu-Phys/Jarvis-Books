---
name: find-your-results
title: 扫描完了，结果都在哪
intent: "跑完了（或跑挂了），我去哪里找数据、日志、图？"
triggers: [结果, 输出, DATABASE, 日志, samples, 在哪]
level: beginner
verified: 2026-07-21 @ jarvis2/2daf417
---

# 扫描完了，结果都在哪

## 目标

一张地图：`outputs/<scan名>/` 下每个东西是什么、先看哪个。

## 输出地图

```
<项目根>/
├── outputs/<scan>/                ← 一切数据
│   ├── DATABASE/
│   │   ├── samples.hdf5           ← 主数据：全部点的参数+observables+LogL
│   │   ├── samples.csv            ← 同内容全列 CSV（Excel/pandas 直开）
│   │   ├── sampler_summary.json   ← 采样器诊断（接受率/R-hat/ESS 或 logZ）
│   │   ├── chain_history.csv      ← MCMC 才有：逐链逐步历史
│   │   └── dynesty_result.csv     ← 嵌套采样才有：死点+log_weight
│   ├── SAMPLE/000001/<uuid>/      ← 每个点的现场（输入输出文件、单点日志）
│   │   └── …归档后打包成 000001.tar.gz
│   ├── run_summary.{json,csv,txt} ← 本次运行统计（成功/失败/耗时）
│   ├── levelset.json              ← AdaptiveBridson 才有：等值线
│   └── redis/                     ← 托管 broker 的持久化（如启用）
├── images/<scan>/                 ← 图的 YAML 场景 + 渲染出的 PNG
│   ├── flowchart.{json,png}       ← 工作流依赖图
│   └── <scan>_levelset_jplot.yaml ← 自动生成的散点场景（Jarvis2 plot 渲染）
└── logs/<scan>/                   ← 分进程日志
    ├── core.log / sampler.log / factory.log / archiver.log
    └── worker-00.log …
```

## 先看哪个

| 问题 | 看这里 |
|---|---|
| 扫描整体成功了吗 | 控制台结尾 `[Scan Performance]`，或 `run_summary.txt` |
| 数据拿来分析 | `DATABASE/samples.csv`（或 hdf5） |
| 某个点为什么失败 | `SAMPLE/…/<uuid>/` 里的单点日志——命令、输出、报错现场都在 |
| MCMC 收敛了吗 | `DATABASE/sampler_summary.json` 的 R-hat / ess_logl |
| logZ 是多少 | `sampler_summary.json`，或 `logs/<scan>/sampler.log` 的 summary 段 |
| 运行时哪个环节出的问题 | `logs/<scan>/` 按进程分文件翻 |

## 常用命令

```bash
Jarvis2 monitor                 # 运行中：看一眼当前状态
Jarvis2 convert outputs/<scan>  # HDF5 → CSV（如需重新导出）
Jarvis2 plot images/<scan>/<scan>_levelset_jplot.yaml   # 渲染自动场景
```

## 常见坑

- **samples.csv 行数比预期少** → 有失败点（看 `run_summary` 的 failed 数），
  或 Archiver 还没追平就去看了——正常结束时会自动追平。
- **SAMPLE 目录只有 tar 包** → 归档后按桶打包是默认行为；
  `tar xzf 000001.tar.gz` 即可展开单看。
- **只想每次留失败点现场** → `EnvReqs.V2.worker.sample_artifacts: auto`（默认就是）。
