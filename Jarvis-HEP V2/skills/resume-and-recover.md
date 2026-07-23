---
name: resume-and-recover
title: 断点续跑与善后
intent: "扫描被打断了（Ctrl-C / 断电 / 挂了），怎么接着跑？残留进程怎么清？"
triggers: [续跑, resume, checkpoint, 中断, 断点, 清理进程]
level: beginner
verified: 2026-07-21 @ jarvis2/2daf417
---

# 断点续跑与善后

## 目标

被打断的扫描从检查点继续，不重算已完成的点；清干净可能残留的进程。

## 复制即用

```bash
Jarvis2 run my_card.yaml --resume     # 从最新 checkpoint 继续，跳过 30 秒确认
```

不加 `--resume` 直接重跑同一张卡时，如果发现旧 checkpoint 会给 30 秒提示让你选。

## 它做了什么

运行中每隔 30 秒（及 MCMC/自适应采样的每个代际屏障）自动存检查点；
**Ctrl-C 时也会先强制写一份再退出**。`--resume` 读取检查点恢复采样器状态和随机数
序列，把中断时在跑的点重新入队——恢复后的轨迹和没中断过一样（与 Worker 数无关）。

## 中断后的善后三件套

```bash
Jarvis2 ps          # 列出所有 Jarvis 相关进程（control/worker/archiver/redis）
Jarvis2 kill        # 交互式清掉残留（脚本里用 --yes）
Jarvis2 monitor     # 确认没有残余扫描在动
```

被强杀的 Worker 的计算槽位会由看门狗自动回收；孤儿计算子进程也会按进程组补杀——
一般不需要手动干预，`ps` 看到干净即可。

## 常见坑

- **换了 Worker 数再 resume** → 完全没问题，结果轨迹不变（这是设计保证，放心调）。
- **改了 YAML 再 resume** → 采样相关的改动会导致状态对不上；resume 只该用于
  "同一张卡接着跑"。想改配置就开新 scan（换 `Scan.name`）。
- **Redis 被清空 / 换了机器** → checkpoint 才是权威，broker 丢了也能 resume；
  但 `outputs/<scan>/` 目录必须还在原路径。
- **想彻底重跑** → 换 `Scan.name`，或删掉 `outputs/<scan>/` 后普通 `run`。
