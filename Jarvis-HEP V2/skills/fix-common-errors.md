---
name: fix-common-errors
title: 常见报错急救表
intent: "报错了，我想最快知道是什么问题、怎么修"
triggers: [报错, 错误, error, 失败, 排错, validate]
level: beginner
verified: 2026-07-21 @ jarvis2/2daf417
---

# 常见报错急救表

## 第一反应：先体检

```bash
Jarvis2 validate my_card.yaml          # 不跑任务，只查 YAML；错误带编号和位置
Jarvis2 validate my_card.yaml --strict # 把警告也当错误（推荐提交前用）
```

大多数配置类问题在这一步就能定位——报错信息里带 `JV2-…` 错误码、出错的键路径
和修法提示。**任何时候改完 YAML，先 validate 再 run。**

**校验是严格的，而且发生在一切之前。** 卡片里出现不认识的键或类型不对，
`Jarvis2 run` 会直接以退出码 2 停下——**不连 Redis、不起 Worker、不建任何目录**，
不会留下半个残缺的 `outputs/`。所以校验报错时你什么都不用清理，改完卡片重跑即可。

拼错键会直接告诉你想写的是什么：

```
[error] JV2-SCH-001  $.Sampling.AdaptiveBridson
        Additional properties are not allowed ('initial_radiuz' was unexpected)
        suggestion: Remove or rename 'initial_radiuz'. Did you mean 'initial_radius'? …
```

一次会把**所有**错误报全（按路径排序），改一轮就行，不用反复试。

## 急救表

| 症状（关键词） | 原因 | 修法 |
|---|---|---|
| `internal Redis is unavailable at 127.0.0.1:6379` | 本地 Redis 没启动 | `brew services start redis` 或 `docker run -d -p 6379:6379 redis:7`；远程 broker 用 `EnvReqs.V2.redis` |
| `sampler method 'Xxx' …` + 一串可用名 | Method 拼写错 | 从报错列出的名单里照抄；选法见 [choose-sampler](choose-sampler.md) |
| `JV2-SCH-001` + `Additional properties are not allowed` | 键名拼错或该块不认识这个键 | 按提示里的 **Did you mean** 改；`Allowed keys` 列出了该块全部合法键 |
| `JV2-SCH-001` + `is not valid under any of the given schemas` | 值的类型不对（如整数位给了字符串） | 对照 YAML_REFERENCE 改成正确类型。数字可以写 `0.05` 或 `1.0e-5`；注意 YAML 里 **`1e-5` 会被当成字符串**（少个小数点），虽然目前能通过，仍建议写 `1.0e-5` |
| `JV2-ENC-001`（卡片里有中文/非 ASCII） | 任务卡的**键名和字符串值只能用 ASCII** | 把中文改成英文；**注释里的中文完全不受限制**（`# 这是暗物质扫描` 随便写），想写说明就写在注释里 |
| `JV2-SCH-002`（不支持的 IO 格式） | Portal 没有这个格式的适配器 | 报错会列出 **Portal 当前真实支持的格式**，照着选；升级 Jarvis-Portal 后新格式会自动可用，无需改 HEP |
| `JV2-BND-012`（Bounds.dynamic 被拒） | 嵌套采样没有 dynamic 开关 | 删掉该键；动态=`Dynesty`，静态=`MultiNest` |
| `unsupported EnvReqs.V2 setting(s): …` | V2 块里写了不支持的键 | 报错列出全部支持键；对照 YAML_REFERENCE §5 |
| `top-level Runtime is no longer a V2 YAML interface` | 用了旧版 Runtime 块 | 把 workers/batch_size 挪进 `EnvReqs.V2` |
| `Command failed [execution#…] rc=…` | 外部程序在某个点上挂了 | 进该点的 `SAMPLE/…/<uuid>/` 看现场日志，命令可手工复跑；见 [external-calculator](external-calculator.md) 常见坑 |
| `timed out acquiring calculator slot` | 计算器实例全忙且等待超时 | 加 `Calculators.make_paraller`，或查是不是有实例卡死（`Jarvis2 ps`） |
| `Operas … requires Jarvis-Operas` / 算子找不到 | 算子名拼错或包太旧 | `Jarvis2 operas list` 看已注册算子；升级 Jarvis-Operas |
| `LogLikelihood expression … misses observables: [z]` | 表达式用了不存在的量 | 检查 calculator/Operas 的 `output` 是否真的产出该键；名字大小写敏感 |
| 扫描结束 exit code 1、日志有 failed 计数 | 部分/全部点失败 | 这是**故意的诚实设计**；按 [find-your-results](find-your-results.md) 查失败点现场 |
| 大量 `worker recovered / respawn` 日志 | 长计算+看门狗阈值太紧（罕见） | 心跳线程默认已覆盖长计算；仍出现就调 `EnvReqs.V2.factory.watchdog.stale_sec` |

## 还是没辙？

1. `logs/<scan>/` 按进程翻日志（core/sampler/worker-NN/archiver 分文件）；
2. 单点问题一律看 `SAMPLE/…/<uuid>/`——失败点**保证**留有可读现场（设计不变量）；
3. 带着 `Jarvis2 validate --json` 的输出和相关日志段提 issue。
