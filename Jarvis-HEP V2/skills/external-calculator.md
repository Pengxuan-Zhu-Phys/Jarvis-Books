---
name: external-calculator
title: 接入我自己的计算程序
intent: "每个参数点要跑一个外部程序（SPheno / micrOMEGAs / 自写脚本），怎么接？"
triggers: [外部程序, calculator, 计算器, SPheno, micrOMEGAs, 接程序]
level: intermediate
verified: 2026-07-21 @ jarvis2/2daf417
---

# 接入我自己的计算程序

## 目标

让 V2 对每个采样点：写输入文件 → 跑你的程序 → 读输出文件，全自动。

## 前提

- 你的程序能命令行运行，输入输出走**文件**（JSON/SLHA/DAT/CSV 都支持）
- 先跑通过 [first-scan](first-scan.md)

## 复制即用

假设你的程序是 `mycalc.py`，读 `input.json`、写 `output.json`。
在任务卡里加一个 `Calculators` 块（Sampling 部分沿用 first-scan 的即可）：

```yaml
Calculators:
  make_paraller: 4                     # 同时最多 4 个实例（V1 拼写，别改）
  Modules:
    - name: MyCalc
      required_modules: []
      clone_shadow: True               # 每个实例一份独立工作目录（防互踩）
      path: "&J/calculators/MyCalc/@PackID"   # &J=项目根, @PackID=实例号 001..N
      source: "&J/src/mycalc"          # 程序源目录
      installation:
        - "cp -r ${source}/* ${path}"  # 首次安装：把程序复制进工作目录
      execution:
        path: "&J/calculators/MyCalc/@PackID"
        commands:
          - "python3 mycalc.py"        # 字符串命令即可（V1 写法）
        input:
          - name: inp
            path: "@Sdir/input.json"   # @Sdir = 本样本点的专属目录
            type: "JSON"
            actions:
              - type: "Dump"
                variables:
                  - {name: "x", expression: "x"}
                  - {name: "y", expression: "y"}
        output:
          - name: oup
            path: "@Sdir/output.json"
            type: "JSON"               # 读出的键自动成为 observables
```

之后 `LogLikelihood` 里就能直接用输出文件里的量（比如程序写了 `z`）：

```yaml
  LogLikelihood:
    - {name: "LogL_Z", expression: "LogGauss(z, 100, 10)"}
```

先用一个点冒烟，再全量跑：

```bash
Jarvis2 check my_card.yaml     # 单点冒烟：装程序、跑一遍、读输出
Jarvis2 run my_card.yaml
```

## 它做了什么

启动时按 `installation` 把程序装进 `@PackID` 编号的工作目录（001…N，可复用）；
每个点：按 `input` 规则把参数写成文件 → 在工作目录跑 `commands` →
按 `output` 规则读回结果并入 observables → 算 LogL。

## 常见改动

| 想要…… | 改这里 | 详见 |
|---|---|---|
| SLHA 格式输入/输出 | `type: "SLHA"`（需 Portal ≥1.4） | DESIGN_PORTAL_IO_2.0 |
| 程序要先 source 环境 | 模块加 `env_setup: [{source: "&J/env.sh"}]` | YAML_REFERENCE §9.4 |
| 输入里写表达式 | `{name: "mx", expression: "x * 1000"}` | YAML_REFERENCE §7 |
| 限时 | 模块加 `timeout: 600`（秒） | YAML_REFERENCE §9.4 |
| A 跑完才能跑 B | B 的 `required_modules: [A]` | YAML_REFERENCE §9.4 |

## 常见坑

- **installation 命令必须幂等**：工作目录跨扫描复用，新 Worker 会对已存在的目录
  重跑安装（覆盖式 `cp -r` 天然安全，`mkdir` 记得加 `-p`）。
- **程序留了旧输出** → 在 `execution.commands` 前面加一条 `"rm -f output.json"`，
  或在 initialization 里清理。
- **`Command failed [execution#...]`** → 进 `SAMPLE/.../<uuid>/` 看该点日志
  （见 [find-your-results](find-your-results.md)），命令原样在里面，可手工复跑。
