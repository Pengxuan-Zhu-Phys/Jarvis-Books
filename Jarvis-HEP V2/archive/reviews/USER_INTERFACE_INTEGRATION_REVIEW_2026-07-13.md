# Jarvis-HEP V2 用户接口与三桥接评审

**评审日期**：2026-07-13
**代码基线**：`Jarvis-HEP-v2` / `jarvis2` / `6b9841e`
**对照基线**：冻结的 `Jarvis-HEP` V1 用户接口与其 PLOT、Portal、Operas 接入代码
**受众**：V2 维护者、CLI/API 设计者、Jarvis-Agent 与插件包维护者

## 技术摘要

V2 的分布式执行主线已经稳定，D0–D7 关闭，D9 的主要解耦已经落地，D10
AdaptiveLevelSet 核心也已可运行。2026-07-13 的验证基线为：

- 用户接口与三桥接定向测试：**55 passed**。
- 排除会改写当前用户 benchmark 工件的 acceptance 文件后：**245 passed, 1 skipped**。
- `python3 -m compileall -q jarvishep2` 与 `python3 -m pip check` 均通过。
- 唯一 skip 是 workflow flowchart export 尚未实现；这和 PLOT 接入深度的结论一致。

三套外部组件都已真实连接，但不能统一写成“集成完成”：

| 组件 | 当前判定 | 已经可用 | 尚未接通 |
|---|---|---|---|
| Jarvis-Portal | **运行路径已接通** | calculator 输入/输出走真实 Portal registry；CSV/DAT/JSON/TSV/Wolfram 可发现 | SLHA/xSLHA/File/HepMC 暂未由已安装 Portal registry 暴露；V2 无格式发现命令 |
| Jarvis-Operas | **运行与表达式桥已接通** | registry/importlib 算子；动态注册函数 qualified name；真实 Eggbox smoke | 签名过滤、sample logger、严格 `call_mode` 校验和发现 UI |
| JarvisPLOT | **render-only 已接通** | `Jarvis2 <plot.yaml> --plot` 调用真实 JarvisPLOT 并产出 PNG | scan YAML → plot YAML、run 后自动绘图、workflow flowchart、D10 level-set overlay |

当前最高优先级不是继续增加命令，而是先固定“一个命令只表达一种意图”的用户接口：
`run/check/monitor/plot/portal/operas` 应成为明确子命令，同时保留
`Jarvis2 TASK.yaml` 作为兼容别名。现有 `--plot` 会把同一个位置参数从 scan YAML
改解释成 plot YAML；多个模式旗标也没有互斥校验，这是最容易让用户误操作的接口缺陷。

## 关键发现与影响

### 1. PLOT 已能画图，但尚未恢复 V1 的 scan-driven 语义

真实 smoke：

```text
python3 -m jarvishep2.client /tmp/jarvis2_plot_smoke.yaml --plot
→ /tmp/jarvis2-plot-smoke-output/bridge_smoke.png
→ exit 0
```

V2 的 `plot_bridge.py` 做的是正确而干净的薄委托：导入 `jarvisplot`、注入参数并调用
`jarvisplot.core.JarvisPLOT().init()`。这证明依赖和执行链都不是 mock。

但 V1 的设计目的更深：

1. 从 HEP scan 配置与 DATABASE 生成 JarvisPLOT scene；
2. 在 HEP 工作流中渲染模块依赖图；
3. 让 `Jarvis scan.yaml --plot` 保持“针对这次 scan 绘图”的语义。

V2 当前的同形命令 `Jarvis2 scan.yaml --plot` 会把 `scan.yaml` 当作 plot scene。
因此这不是简单的“功能较少”，而是**同样外观的命令具有不同语义**。建议立即在帮助和文档中标记
render-only，并用 `Jarvis2 plot PLOT_YAML` 固定该语义；scan-to-plot 另设
`Jarvis2 plot --from-run ...`，或在实现完整后恢复 `Jarvis2 run TASK --plot`。

D10 已输出 `levelset.json`，但 D10.3 的 PLOT overlay hook 尚未实现。当前用户需要自行写
JarvisPLOT YAML 才能可视化 level set。

### 2. Portal 桥接架构正确，缺口在格式暴露与发现体验

V2 calculator 的 production path 已经不再硬编码 JSON，而是调用
`create_entry_point_registry()` 并按方向解析 adapter。未知格式会 hard-fail 并列出可用格式；
CSV 与 JSON 的同步/异步和 calculator 端到端测试均通过。

当前实际可发现集合为：

```text
input/output/all = CSV, DAT, JSON, TSV, Wolfram
```

原 V1 在 HEP 侧额外注册 SLHA input/output、xSLHA output、File output。当前 Portal
仓库虽然存在 SLHA/xSLHA adapter 类，但其 builtin/entry-point 暴露集合仍只含上面五种格式。
因此：

- V2 的桥接无需为每种格式重写；
- 应先在 Jarvis-Portal 正式暴露 HEP adapters；
- 随后在 V2 加一个真实 SLHA fixture 证明端到端可用；
- 在此之前不能对用户宣称“已覆盖真实 HEP I/O”。

V1 提供 `Jarvis portal formats`；V2 没有等价发现面。Portal 自身的 `jportal` 可作为临时入口，
但 HEP 用户不应该依赖知道每个插件包的独立 CLI。

### 3. Operas 能调用真实 registry，尚未达到 V1 的运行语义

当前解析顺序为：

1. importlib dotted callable；
2. 失败后尝试 Jarvis-Operas registry。

这个顺序适合 Worker 快速载入本地算子，也已经由代码和测试固定。旧设计文档中
“两者同时存在时 Operas wins”的表述与实现矛盾，应以 **importlib-first** 为准。

真实 registry smoke：

```text
resolve_operator("helper.eggbox2d")(x=0.2, y=0.3)
→ 成功返回结果
```

原 V1 还承担了四类语义；2026-07-13 已完成最后一项：

- 严格验证 `call_mode`；
- 按函数签名过滤/适配 kwargs；
- 把 sample logger 传入算子执行链；
- 把动态注册函数注入表达式运行时。**已完成**：统一使用
  `namespace.function(...)`；Worker 启动快照后热路径不再查询 registry。

前三项目前仍未完成。尤其是非法 `call_mode: typo` 会静默落到同步调用，这属于配置错误被吞掉的
用户接口 bug，应在 preload/validation 阶段失败。Jarvis-Operas 已提供 `jopera list/info/call`，
V2 后续只需做薄发现代理，不应复制 registry。

### 4. 当前 CLI 是可用的工程入口，还不是稳定的用户契约

当前 surface：

```text
Jarvis2 [task_yaml]
  --resume
  --check-modules
  --monitor
  --plot
  --pid
  --redis-host/--redis-port/--redis-db
```

已确认的接口问题：

| 优先级 | 问题 | 用户影响 | 建议 |
|---|---|---|---|
| P0 | 运行结果按“提交/归档数量”判成功，失败 sample 也会进入归档 | 全部 sample 失败时 CLI 仍可能 exit 0 | 引入 `RunOutcome`，按 completed/failed/cancelled 决定退出码 |
| P0 | failed result 没有稳定的 `error_type`/`error` 字段 | Agent/脚本只能解析日志，无法可靠诊断 | 与 D8 一起定义机器可读 failure envelope |
| P1 | `--plot` 改变位置参数含义 | scan YAML 与 plot YAML 易混淆 | 独立 `plot` 子命令，帮助中明确 render-only |
| P1 | `--monitor/--plot/--check-modules` 不互斥 | 多 flag 时按代码顺序静默选择 | argparse mutually-exclusive group 或子命令 |
| P1 | Redis flag 用实际默认值表示“未传” | 用户显式传 `127.0.0.1:6379/0` 时无法覆盖 YAML 的非默认值 | parser 默认 `None`；monitor 单独填运行默认值 |
| P1 | `--pid` 被解析但未消费 | 命令看似成功、实际无效果 | 实现 attach contract 前删除或 hard-fail |
| P1 | 缺 Portal/Operas discovery | 用户无法从 HEP 入口查看可用格式/算子 | 添加薄代理子命令 |
| P2 | 缺 `--version`、`validate/status/results` | 人与 Agent 都缺稳定 introspection | 完成 D8，并和新子命令树统一 |

推荐目标接口：

```text
Jarvis2 run TASK.yaml [--resume]
Jarvis2 check TASK.yaml
Jarvis2 monitor [--run-id ID] [redis options]
Jarvis2 plot PLOT.yaml
Jarvis2 portal formats
Jarvis2 operas list
Jarvis2 operas info NAME
Jarvis2 validate TASK.yaml [--json]
Jarvis2 status RUN [--json]
Jarvis2 results RUN [--json]
Jarvis2 version [--json]
```

兼容规则：

- `Jarvis2 TASK.yaml` 在至少一个稳定版本内等价于 `Jarvis2 run TASK.yaml`；
- 旧 `Jarvis2 PLOT.yaml --plot` 保留但发 deprecation warning；
- 任何模式冲突都应 exit 2，不再静默按 precedence 路由；
- 人类输出走 stdout/stderr，机器接口只输出单个稳定 JSON envelope。

### 5. D10 已推进，但 D8 与用户控制面成为新的主路径阻塞

截至本次评审：

- D8 Agent Bridge：0/4；
- D9：6 done + 2 partial；
- D10：core/feedback done，output/test/dimension extensions partial。

D10 引入反馈型 sampler 后，过去“提交完固定集合再等待”的成功判定已经更不够用。
因此 `RunOutcome`、机器可读失败、run state 与 graceful stop 不应继续被视为独立附加功能；
它们是 D8、D10 和最终用户接口共同依赖的控制面。

## 评审范围、证据与方法

本报告直接检查了：

- V2：CLI、core、Worker、Portal bridge、Operas bridge、PLOT bridge、打包 extras、测试与现有设计文档；
- V1：`jarvishep/client.py`、`jarvishep/plot.py`、IO registry、Operas registry/runtime；
- 外部包：本地 Jarvis-Portal、Jarvis-Operas、JarvisPLOT 的注册/入口和安装版本；
- 文档基线：V2 plan、YAML reference、component docs 和 2026-07-10 review。

验证命令与结果：

```text
python3 -m pytest -q -rs +  tests/test_cli.py tests/test_plot_bridge.py tests/test_io_portal.py +  tests/test_operas_bridge.py tests/test_worker_calculator.py +  tests/test_adaptive_level_set.py
→ 55 passed

python3 -m pytest -q -rs --ignore=tests/test_distributed_acceptance.py
→ 245 passed, 1 skipped

python3 -m pytest --collect-only -q
→ 252 tests collected

python3 -m compileall -q jarvishep2
→ passed

python3 -m pip check
→ No broken requirements found
```

没有重跑 6 个 distributed acceptance tests，因为工作区已有用户修改的
`docs/benchmarks/d7_1_acceptance.json`，重跑会改写该证据文件。本次没有覆盖、清理或暂存该文件；
也没有触碰用户现有的 `checkpoints/`。

## 限制、稳健性与未决风险

- 本次 broad suite 的单个 skip 明确指向 flowchart export 未实现，不是随机测试失败。
- 定向 suite 首次在受限 sandbox 中有 5 个 fakeredis TCP bind `PermissionError`；在允许本地
  socket 的同一环境重跑后 55/55 通过，因此归类为执行环境限制，不是代码回归。
- Operas smoke 出现 `~/.jarvis-operas/curve-cache` 不可写 warning，但 registry 调用成功；
  生产环境仍应确认 cache 目录策略。
- PLOT smoke 证明 render path；它不能证明 scan-to-plot，因为该路径当前不存在。
- Portal 的 SLHA 结论基于当前本地包暴露表；一旦 Portal 新版本注册 adapters，V2 bridge
  理论上无需改动，但仍须用 fixture 验证。
- 本报告是代码与本地包状态的快照；没有修改原 V1 repo，因为其工作区已有大量用户变更。

## 建议执行路线：D11 用户接口与集成闭环

### D11.1：先修运行结果与错误契约

- 新增 `RunOutcome(submitted, completed, failed, cancelled, archived, run_id)`；
- CLI、run summary、D8 JSON 共用同一结果来源；
- failed sample 写入稳定的 `error_type`、`error`、`failed_module`；
- 明确 partial failure 策略，并为“全部失败仍 exit 0”补回归测试。

### D11.2：重构 CLI 意图模型

- 建立 `run/check/monitor/plot/portal/operas` 子命令；
- 保留旧 run 与 plot alias，给迁移期 warning；
- 删除模式 precedence，加入互斥与缺参测试；
- Redis override parser 默认 `None`；
- `--pid` 在实现前 hard-fail 或移除。

### D11.3：关闭 Portal 的 HEP 格式缺口

- 在 Jarvis-Portal 暴露 SLHA input/output、xSLHA output 与需要的 File adapter；
- `Jarvis2 portal formats` 代理 registry discovery；
- 加一个不 mock adapter 的 SLHA calculator fixture；
- 文档清楚区分“bridge ready”和“format exposed”。

### D11.4：补齐 Operas 运行语义

- preload 时严格验证 `call_mode in {call, acall}`；
- 和 Operas registry 对齐 signature filtering 与 logger context；
- 动态 expression function 已由 D11.4d 完成；保持 V1 YAML 兼容与热路径测试；
- `Jarvis2 operas list/info` 只做 registry 输出适配。

### D11.5：完成 PLOT 的 scan-driven 用户旅程

- 设计 `emit_plot_scene_from_run`，由 HEP 只负责 DATABASE/metadata 到 scene 的映射；
- JarvisPLOT 继续独占 figure/layer/render 算法；
- 完成 workflow flowchart export；
- 为 D10 `levelset.json` 添加标准 overlay scene；
- 端到端测试覆盖 scan → outputs → plot YAML → PNG。

建议顺序为 **D11.1 → D11.2 → D11.3/D11.4 → D11.5**。D8 可与 D11.1/D11.2 合并实施，
避免先做一套 JSON flag、随后又重写一次 CLI。

## 进一步问题

1. partial failure 的产品语义是“有一个成功即 exit 0”，还是“任何失败都非零”？建议：
   普通 run 有部分失败时 exit 1，但结果仍完整落盘；`check` 任何 module 失败都 exit 1。
2. `Jarvis2 run TASK --plot` 最终应仅生成 scene，还是也同步 render？建议生成并 render，
   同时把 scene 留在 outputs 供复现。
3. SLHA/xSLHA adapters 应作为 Portal core 暴露，还是 Portal 的 `hep` extra？建议 adapter
   registration 在 core 可发现，缺可选解析依赖时给可执行安装提示。
4. `--pid` 是否仍有价值？若 Redis/run_id 已成为稳定身份，建议废弃 PID attach，避免远程
   与容器环境下的错误抽象。
