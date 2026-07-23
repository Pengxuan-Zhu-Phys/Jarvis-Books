---
name: plot-your-scan
title: 快速出图
intent: "扫完了，我想马上看到一张参数平面图，再微调它"
triggers: [画图, plot, 出图, JarvisPLOT, 散点, colorbar]
level: beginner
verified: 2026-07-21 @ jarvis2/2daf417
---

# 快速出图

## 目标

一条命令渲染扫描散点图；知道最常用的三处手动微调。

## 前提

- 装了绘图扩展：`pip install "jarvishep2[plot]"`（或已装 JarvisPLOT）
- 扫描已完成，`images/<scan>/` 下有自动生成的 `*_jplot.yaml`

## 复制即用

```bash
Jarvis2 plot images/<scan>/<scan>_levelset_jplot.yaml
```

PNG 出在同目录。场景 YAML 是**普通文本**，改完重跑同一条命令即可。

## 三个最常用的微调（当前需要手动，见下方"已知限制"）

**① 对数轴**——如果变量是 Log 分布扫的，务必加：

```yaml
  frame:
    ax:
      xscale: log        # 加这两行
      yscale: log
```

**② 颜色范围钳制**——有失败点或 LogL 动态范围大时，colorbar 会被拉平，
在 `frame.axc` 下加：

```yaml
    axc:
      color: {cmap: rainbow, scale: linear, vmin: -3, vmax: 0.0}
```

**③ 轴标签换成物理记号**：

```yaml
      labels: {x: "$m_{\\chi}$ [GeV]", y: "$\\sin\\theta$"}
```

## 它做了什么

扫描结束时 V2 自动把 `DATABASE/samples.csv` + （如有）levelset 写成一张
stock jplot 场景；`Jarvis2 plot` 把场景交给 JarvisPLOT 渲染，绘图算法全在 PLOT 侧。
嵌套采样运行还会额外生成 runplot 场景。

## 已知限制（改进已立项：D15.5–D15.7）

当前自动场景**只有散点**，而且不会自动做上面三件事——Log 分布变量不会自动用
log 轴、颜色不做稳健钳制、嵌套采样的死点散点未按 `log_weight` 加权（那不是后验！）。
物理级自动场景（profile likelihood 等值线、后验加权密度、best-fit 星标、
`Variables[].latex` 标签）见
[`../DESIGN_PHYSICS_PLOT_SCENES_2.0.md`](../DESIGN_PHYSICS_PLOT_SCENES_2.0.md)。
在那之前，请按本 skill 手动微调——**你的手改文件不会被覆盖**（重新生成会另存新文件）。

## 常见坑

- **图黑压压一坨点** → 没开 log 轴（微调①）。
- **colorbar 只有一种颜色** → 有 `-inf`/超深失败点 → 微调②钳制 vmin/vmax。
- **`Jarvis2 plot` 报缺包** → 没装 `[plot]` 扩展 → 见前提。
