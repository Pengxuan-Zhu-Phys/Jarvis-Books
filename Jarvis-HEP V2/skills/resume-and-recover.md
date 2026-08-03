---
name: resume-and-recover
title: 断点续跑与善后
intent: "扫描被打断了（Ctrl-C / 断电 / 挂了），怎么接着跑？残留进程怎么清？"
triggers: [续跑, resume, checkpoint, 中断, 断点, 清理进程]
level: beginner
verified: 2026-08-02 @ D21 end-to-end acceptance
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

运行中默认每隔 30 秒（**仅**由 `EnvReqs.V2.checkpoint_heartbeat_sec` 调整；不要写在
`Sampling.Bounds` 里）请求一次 checkpoint。Random / Grid / Bridson / CSV 会在**下一个
完整提交批次**捕获轻量 generator 状态；Dynesty / MultiNest 用同一间隔写
`nested_engine.pkl` + `state.pkl`。MCMC/自适应采样则仍在每个安全代际屏障保存。
**Ctrl-C 时也会先强制写一份再退出**。`--resume` 读取检查点中的安全恢复点，恢复采样器
状态和随机数序列，并只将尚未落盘的逻辑索引重新交给 Worker。

是否完成只看 `DATABASE/samples.hdf5`，不看 checkpoint 或 Redis 计数。枚举型采样器的
每条记录都有单调的 `sample_index`；Archiver 只在 HDF5 fsync 后发布连续前缀水位。
因此已经落盘的点不会重复写入；即使 checkpoint 丢失，枚举型扫描也会在控制进程本地
快进到 DATABASE 前缀，而不把这些物理计算重新交给 Worker。恢复后的结果轨迹和没中断过
一样（与 Worker 数无关）。

例如，对单个计算很昂贵的任务可以把重放窗口调小：

```yaml
EnvReqs:
  V2:
    checkpoint_heartbeat_sec: 10
```

## 中断后的善后三件套

```bash
Jarvis2 ps R1       # 查看指定 runtime（control/worker/archiver/redis）
Jarvis2 kill R1 -y  # 清掉指定 runtime 的残留进程
Jarvis2 ps ZP       # 查看所有没有对应 live Control 的 Jarvis 孤儿进程
Jarvis2 kill ZP -y  # 一键清掉 ZP 中的全部无归属进程
Jarvis2 monitor     # 确认没有残余扫描在动
```

控制进程被 SIGKILL 后，Worker/Archiver 会从 Redis lease 发现控制端消失并退出；
Worker 自己被强杀时，FileOperation 即使正忙于 copy/delete，也会通过独立 owner-watch
退出，并连同它启动的 `rm` 子进程一起清理。`ps`/`kill` 也允许识别和清理 control PID
已死的 stale runtime。旧版本遗留的无 scan 名 FileOperation，以及任何没有对应 live
Control 的裸/残留 Jarvis Worker/Archiver/Redis，统一显示为固定 `ZP`，不占用 R#；即使
进程标题还留着旧 `:<scan>` 后缀也一样。可用
`Jarvis2 kill ZP -y` 集中清理。`--resume` 会清理同名孤儿进程，
并在需要时自动修复 HDF5 的陈旧 SWMR / writer flag。

### HDF5 陈旧 writer flag：自动修，**不需要装任何东西**

进程被 SIGKILL 后，`samples.hdf5` 的 superblock 上会留下"仍被写入中"的标志，
此时**连只读都打不开**（`file is already open for write …`）。

resume 会自己处理：以 SWMR 只读方式打开原文件，把全部内容复制进一个 superblock
干净的新文件，再原子替换回去——**纯 h5py 实现，不依赖 `h5clear` 命令行工具**
（后者不在 pip 的 `h5py` wheel 里）。所以 `pip install h5py` 的标准环境即可恢复，
日志里会有一条汇总的 recovery 记录。

需要人工介入的只有一种情况：**确实还有活着的 Archiver 在写这个文件**。
先用 `Jarvis2 ps` / `Jarvis2 kill` 清干净，再 resume。

Jarvis-Lit 是独立后台程序，不属于 Jarvis-HEP 的 `ps`/`kill` 范围；其进程标题、工作目录
和 `JarvisLit.app` 路径会被显式排除。

## 常见坑

- **换了 Worker 数再 resume** → 完全没问题，结果轨迹不变（这是设计保证，放心调）。
- **改了 YAML 再 resume** → 采样相关的改动会导致状态对不上；resume 只该用于
  "同一张卡接着跑"。想改配置就开新 scan（换 `Scan.name`）。
- **Redis 被清空 / 换了机器** → DATABASE 才是完成事实的权威，broker 丢了可以重建；
  checkpoint 负责恢复 generator 的精确状态。`outputs/<scan>/` 与 checkpoint 目录必须
  仍能从任务路径找到。
- **checkpoint 丢了** → Random/Grid/Bridson/CSV 等枚举型扫描仍可用 DATABASE 对账续跑；
  自适应/反馈型扫描需要 checkpoint 才能恢复严格相同的世代轨迹。
- **想彻底重跑** → 换 `Scan.name`，或删掉 `outputs/<scan>/` 后普通 `run`。
