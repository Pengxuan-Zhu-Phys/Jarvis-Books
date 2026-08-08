---
name: external-calculator
title: 接入我自己的计算程序
intent: "每个参数点要跑一个外部程序（SPheno / micrOMEGAs / 自写脚本），怎么接？"
triggers: [外部程序, calculator, 计算器, SPheno, micrOMEGAs, 接程序]
level: intermediate
verified: 2026-08-02 @ jarvis2/D20 working tree
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
| 同一安装需切换互斥构建模式 | 父模块加 `modes` | YAML_REFERENCE §9.6 |

## 同一程序的互斥构建模式

只有在两种用法**不能同时存在于一份安装里**时才用 `modes`。例如 Prospino
切换过程需要改 config 并原地重新 make。若两个二进制或子目录能共存，直接写两个普通
calculator 模块。

```yaml
Calculators:
  Pools: {Prospino: 8}
  Modules:
    - name: Prospino
      path: "&J/calculators/Prospino/@PackID"
      source: "&J/src/Prospino"
      clone_shadow: true
      installation: ["cp -R ${source}/. ${path}"]
      modes:
        - name: ng
          installation: ["python configure.py ng", "make"]
          execution:
            commands: ["./prospino"]
            output: [{name: xsec_ng, path: "@Sdir/ng.json", type: JSON}]
        - name: ns
          installation: ["python configure.py ns", "make"]
          execution:
            commands: ["./prospino"]
            output: [{name: xsec_ns, path: "@Sdir/ns.json", type: JSON}]
```

所有 mode 共用 `Prospino/001` 这类物理 PackID 目录。父级 `installation`
只在从零构建/强制重装时执行；切 mode 只执行目标 mode 的 `installation`。
同一父模块的 mode 在一个样本内串行，不同 calculator 仍可并发。

依赖写法：`required_modules: [Prospino]` 表示全部 mode；
`[Prospino.ng]` 表示只依赖一个 mode。不存在 `@Mode` 或 `${mode_dir}`。
建议池大小不低于 `mode 数 + Worker 数`；池小于 mode 数时反复重建不可避免。

## 常见坑

- **⚠ 改了程序源码，必须主动要求重装。** 安装是**复用**的：指纹只认配置变化
  （命令、路径、source 位置），**故意不去扫描源码内容**。所以你改完程序直接重跑，
  跑的还是上次装进 pack 目录的旧版本，而且没有提示。

  在 **calculator 文件夹**（`00*` 之外）的 `jarvis_install.json` 里把
  `"reinstall"` 设为 `true`，重跑一次即可——该模块的所有 pack 都会重装一次。
  控制进程会把它转换成单调递增的 `reinstall_epoch`，Worker 只更新自己 pack
  内的 `.jarvis_install_stamp.json`；不要手动修改 epoch，也不要在 pack 目录里
  写控制文件。旧的全局应急开关仍兼容：

  ```bash
  JARVIS_FORCE_CALC_INSTALL=1 Jarvis2 run my_card.yaml
  ```

  设计见 `../DESIGN_CALC_INSTALL_CONTROL_2.0.md`。
- **installation 命令必须幂等**：工作目录跨扫描复用，可能对已存在的目录重跑安装
  （覆盖式 `cp -r` 天然安全，`mkdir` 记得加 `-p`）。
- **程序留了旧输出** → 在 `execution.commands` 前面加一条 `"rm -f output.json"`，
  或在 initialization 里清理。
- **`Command failed [execution#...]`** → 进 `SAMPLE/.../<uuid>/` 看该点日志
  （见 [find-your-results](find-your-results.md)），命令原样在里面，可手工复跑。
- **selection 必须写在具体 mode 或父模块上。** V2 会在取 Redis PackID 之前判断；
  被跳过的 mode 不安装、不执行，也不会污染 mode 亲和。
