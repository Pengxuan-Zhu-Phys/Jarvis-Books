---
name: first-scan
title: 第一次跑通一个扫描
intent: "我想快速跑一个最小扫描，确认 Jarvis-HEP V2 装好了、能出结果"
triggers: [快速开始, quickstart, 第一次, 安装验证, hello world]
level: beginner
verified: 2026-07-21 @ jarvis2/2daf417
---

# 第一次跑通一个扫描

## 目标

5 分钟内完成一次 40 个点的 2D 扫描（内置 EggBox 玩具函数，不需要任何外部程序），
在 `outputs/` 里看到 HDF5/CSV 结果和一张图的 YAML。

## 前提

- `Jarvis2 -v` 能打出 logo（说明安装成功）
- 本地 Redis 在跑：`redis-cli ping` 返回 `PONG`
  （没装：macOS `brew install redis && brew services start redis`，
  Linux `docker run -d -p 6379:6379 redis:7`）

## 复制即用

最省事的方式是用脚手架，模板已经带好这张卡：

```bash
Jarvis2 project create MyFirstScan
cd MyFirstScan
Jarvis2 run bin/quickstart_bridson_operas.yaml
```

或者手工保存下面这张卡为 `first.yaml`（任意目录）：

```yaml
Scan:
  name: "first"

Sampling:
  Method: "Bridson"          # 蓝噪声均匀撒点；换 "Random" 也行
  Variables:
    - name: x
      distribution: {type: Flat, parameters: {min: 0.0, max: 5.0, length: 40}}
    - name: y
      distribution: {type: Flat, parameters: {min: 0.0, max: 5.0, length: 40}}
  Bounds:
    Radius: 2.0
    MaxAttempt: 20
  LogLikelihood:
    - {name: "LogL_Z", expression: "LogGauss(z, 100, 10)"}

Operas:
  Modules:
    - name: EggBox
      operator: "jarvishep2.testing.eggbox.eggbox2d_numpy"   # 内置玩具函数
      call_mode: call
      required_modules: []
      input:
        - {name: x, expression: "x"}
        - {name: y, expression: "y"}
      output:
        - {name: z, entry: z}
```

```bash
Jarvis2 validate first.yaml   # 先体检（可选但推荐）
Jarvis2 run first.yaml
```

## 它做了什么

采样器在 (x,y) 平面撒点 → Worker 进程对每个点调用 EggBox 函数得到 z →
按表达式算 LogL → 结果归档到 `outputs/first/DATABASE/samples.{hdf5,csv}`。
控制台会显示进度条，结束时打印 `[Scan Performance]` 摘要。

## 常见改动

| 想要…… | 改这里 | 详见 |
|---|---|---|
| 更多/更少的点 | `length: 40`（每维目标点数）与 `Radius` | YAML_REFERENCE §6 |
| 多开几个 Worker | 加 `EnvReqs: {V2: {workers: 4}}` | YAML_REFERENCE §5 |
| 对数扫描某个变量 | `distribution: {type: Log, ...}` | YAML_REFERENCE §6.1 |
| 换真实计算程序 | 见 [external-calculator](external-calculator.md) | — |

## 常见坑

- **卡在启动、报 Redis 连接失败** → 本地 Redis 没跑 → `redis-cli ping` 检查，见前提。
- **`unknown sampler method`** → Method 名拼写错误 → 报错里会列出全部可用方法名，照抄。
- **结果目录找不到** → 见 [find-your-results](find-your-results.md)。
