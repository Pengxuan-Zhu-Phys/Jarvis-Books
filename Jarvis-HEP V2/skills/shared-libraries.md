---
name: shared-libraries
title: 构建并复用共享第三方库
intent: "我的扫描要用 Delphes、HepMC 等共享库，怎样只装一次？"
triggers: [LibDeps, Delphes, HepMC, 共享库, library, 安装依赖]
level: intermediate
verified: 2026-08-01 @ jarvis2/D18
---

# 构建并复用共享第三方库

## 目标

在 Worker 启动前构建依赖库一次；后续同配置扫描自动复用，而不是每个样本重装。

## 前提

- 先用 `Jarvis2 validate my_card.yaml` 确认任务卡通过校验。
- 第一次构建可能很久；输出在 `logs/<scan>/library-<name>.log`。
- 若命令使用 `@{ROOT path}`，在 `EnvReqs.CERN_ROOT` 里配置 `path` 或
  `get_path_command`。

## 复制即用

下面是一张最小有效卡。这里的 `true` 是一个无副作用的最小烟雾构建；把它替换成你的
解压、configure、make 命令，并把你已有的 `Calculators` / `Operas` 块加回卡片。

```yaml
Scan:
  name: shared-libraries-skill
Sampling:
  Method: Random
  Bounds:
    Point number: 1
  Variables:
    - name: x
      distribution:
        type: Flat
        parameters: {min: 0.0, max: 1.0}
LibDeps:
  path: "&J/deps/library"
  make_paraller: 4
  Modules:
    - name: ExampleSharedLibrary
      required_modules: []
      installed: false             # V1 兼容；V2 以 stamp 为准
      installation:
        path: "${LibDeps:path}/ExampleSharedLibrary"
        commands:
          - "mkdir -p ${path}"
          - "true"
```

```bash
Jarvis2 validate my_card.yaml
Jarvis2 run my_card.yaml
```

## 它做了什么

控制进程按 `required_modules` 排序构建；同一层独立的库最多并发
`make_paraller` 个。成功后会写 `jarvis_install.json` 和每个库目录中的
`.jarvis_install_stamp.json`。下一次配置与 source 根状态一致时直接复用。

## 常见改动

| 想要…… | 改这里 | 详见 |
|---|---|---|
| 用另一个库的安装目录 | `${LibDeps:OtherLibrary}` | YAML_REFERENCE §8.1 |
| 用 ROOT 环境构建 | `source @{ROOT path}/bin/thisroot.sh` | YAML_REFERENCE §8.1 |
| 强制全部重装 | `jarvis_install.json` 的 `"reinstall": true` | YAML_REFERENCE §8.1 |
| 只验证已有安装 | `--skip-library-installation` | YAML_REFERENCE §8.1 |

## 常见坑

- **第一次运行看起来停在预检** → 这是共享库正在构建；查看
  `logs/<scan>/library-<name>.log`。
- **改了 source 树内文件但仍复用旧库** → 这是有意的 stamp 规则；在
  `<LibDeps.path>/jarvis_install.json` 中把 `"reinstall"` 设为 `true` 后再运行。
- **`--skip-library-installation` 报路径不存在** → 先不带该 flag 完成一次构建；skip
  从不替你创建或修复库目录。
- **库构建失败** → Run 会在 Redis 和 Worker 启动前停止；查看错误中给出的
  `library-<name>.log`，修命令后重跑。
