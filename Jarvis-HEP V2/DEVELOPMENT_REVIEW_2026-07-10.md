# Jarvis-HEP V2 开发状态与改进报告（2026-07-10）

> 评审基线：`Jarvis-HEP-v2` 分支 `jarvis2`，HEAD `63f012f`。
> 评审范围：当前源码、测试、安装/CLI 表面、D0–D9 计划、Agent Bridge、组件文档及既有代码评审。
> 方法：设计—代码逐项对照、提交差异审查、静态扫描、Python 编译检查、依赖检查、CLI 冒烟、全量 pytest。

## 1. 结论先行

Jarvis-HEP V2 的**分布式执行主链路已经可用**：D0–D7 全部闭环，Redis + spawn Worker +
异步 Archiver、断点续跑、watchdog、监控和慢任务验收均已有实现与测试。D9 架构加固也已完成
大部分工作，尤其是 Calculator 拆分、Sampler/IO 注册表和 FixedSetSampler 抽取。

当前项目不应再被描述为“JSON-only 的早期原型”。Calculator I/O 已切换到
Jarvis-HEP-Portal，当前安装环境可发现 `CSV/DAT/JSON/TSV/Wolfram` 五种双向格式；Operas 和
Jarvis-PLOT 也已有桥接层。不过，距离“Agent 可稳定驱动、真实 HEP 工作流功能完整”的目标仍有
三个主要缺口：**D8 Agent Bridge 尚未开始、D9.4/D9.6 未收尾、MCMC/SLHA/nuisance/schema
校验等领域能力仍缺失**。

综合判断：

- **运行时成熟度：约 80%**（核心分布式运行时已完成，生产硬化仍需补强）。
- **D 系列计划完成度：D0–D7 = 100%，D8 = 0/4，D9 = 6/8 完成 + 2/8 部分完成。**
- **面向 Jarvis-Agent 的可集成度：偏低**，因为机器可读 API、run state 和控制进程停机契约均未落地。
- **文档可信度：中等偏低**，主计划 ledger 接近当前状态，但大量组件文档仍冻结在旧提交
  `d0de31a`，会误导后续开发。

## 2. 当前开发进度

| 里程碑 | 当前状态 | 评审判断 |
|---|---:|---|
| D0 Foundations | 完成 | Sample、RedisQueue、日志与 spawn 序列化契约已落地。 |
| D1–D4 Execution | 完成 | Worker/Calculator、多 Worker、并发、staging、Archiver 与输出闭环已落地。 |
| D5 Monitoring | 完成 | `op_count` 门控快照、monitor、run_summary 已落地；部分指标仍未采集。 |
| D6 Recovery | 完成 | heartbeat、worker respawn、in-flight requeue、checkpoint/resume 已落地。 |
| D7 Acceptance | 完成 | 慢任务 scaling/backlog/chaos/parity 验收存在；当前 benchmark JSON 有本地刷新结果。 |
| D8 Agent Bridge | **未开始** | 4 个 WP 均为 todo，是 Jarvis-Agent 集成的当前阻塞项。 |
| D9 Architecture Hardening | **大部完成** | D9.1/2/3/5/7/8 完成；D9.4/9.6 仅部分完成。 |

自旧评审基线 `d0de31a` 以来，当前 HEAD 新增/修改 36 个文件，约 `+2169/-911` 行。主要成果：

1. `CalculatorModule` 拆为 `CalculatorSpec`、`RuntimePreparer` 与较薄的执行编排层。
2. I/O 改由 Jarvis-HEP-Portal 注册表提供，未知格式显式报错。
3. Distributor 改为可扩展的 Sampler 注册表。
4. Bridson/Random/Grid 共享 `FixedSetSampler`。
5. Operas 支持注册表名称与 Python dotted callable；PLOT 通过独立 bridge 接入。
6. check-modules 的 golden 校验/列推断迁至 `jarvishep2.testing`。
7. `TaskFactory` 生产调用改为显式实例持有，run_summary 不再用假零值冒充未采集指标。

## 3. 本次验证结果

| 检查 | 结果 |
|---|---|
| Python 编译 | `python3 -m compileall -q jarvishep2` 通过 |
| 依赖一致性 | `python3 -m pip check`：无破损依赖 |
| CLI 冒烟 | `python3 -m jarvishep2.client --help` 正常 |
| 测试发现 | 30 个 `test_*.py`，共收集 236 个测试 |
| 全量测试 | 235 passed、1 skipped（flowchart export 尚未实现），含分布式/验收测试 |
| Portal 格式发现 | input/output 均为 CSV、DAT、JSON、TSV、Wolfram |
| V1 隔离 | 未发现 `jarvishep` V1 包导入；V2 仍保持独立包边界 |

说明：全量测试中的 acceptance 会临时刷新 benchmark artifact；评审结束前已恢复该文件到评审
开始时的用户工作区版本，没有覆盖其原有未提交修改。

## 4. Bug 与风险清单

### P0：阻断级

本次未发现会使 D0–D7 主链路整体不可运行的 P0 缺陷；全量测试与编译检查均通过。

### P1：应在下一个开发周期处理

#### P1-1：失败结果丢失机器可读错误原因

`Worker.process_task()` 捕获异常后只把状态设为 `Failed`，随后提交
`sample.to_info_dict()`；该投影没有 `error`/`error_type` 字段。调用方只能看到失败，无法从
Redis result、未来 `--results --json` 或 Agent 工具中得到原因，必须翻 Sample 日志。这与
Agent Bridge “errors are data”的原则冲突，也使自动重试/分类诊断困难。

**建议**：在 Sample 结果模型中加入结构化失败对象，例如
`{"type": exc.__class__.__name__, "message": str(exc), "stage": step}`，同时保留日志路径；补充
calculator/opera/likelihood 三类失败结果测试。

#### P1-2：D9.6 的“双重真相”尚未真正消除

虽然已经增加 `merge_observables()`、`set_status()`、`record_pack_id()`，但 Worker 仍直接写
`sample.info["params"]`、`sample.info["observables"]`、`sample.info["status"]`；Likelihood 也直接
修改 `sample_info["observables"]`。因此 `Sample` 字段与 `info` 仍可能漂移，ledger 中“Worker uses
them”的表述只部分成立。

**建议**：完成 D9.6：字段是唯一真相，`info` 只作为兼容投影；Worker/likelihood 不再直接写
关键键。增加一条不变量测试：任意成功/失败路径后 `to_info_dict()` 与字段一致。

#### P1-3：D8 Agent Bridge 0/4，Agent 集成仍被阻塞

`api.py`、`run_state.py`、`--validate/--results/--status/--monitor-json/--version-json` 和控制进程
SIGINT/SIGTERM checkpoint 契约均未实现。`--pid` 虽出现在 CLI 中，但仍是保留的死参数。

**建议**：下一里程碑先做 D8.1 + D8.3，再做 D8.2，最后做 D8.4；不要继续扩展 Agent 侧工具，
直到 V2 机器接口稳定。

#### P1-4：配置校验仍不够 fail-fast

未知键、拼写错误和部分静默类型规整需要到运行期才暴露；长扫描会把简单配置错误变成昂贵失败。

**建议**：把 D8.4 strict diagnostics 作为正式启动门：错误阻止运行，警告保留原始值、规范值和
YAML 路径；先覆盖 Redis 配置、Sampling.Method、Calculator IO type 和死键。

### P2：重要但可排在 P1 之后

#### P2-1：D9.4 仍未达到验收条件

`TaskFactory` 仍为 477 行，monitor/watchdog 尚未拆为协作对象；大量测试仍依赖
`get_instance()/reset_instance()`，所以 singleton 只是退出生产主路径，并未真正退役。

**建议**：把测试迁到显式 fixture，再删除 singleton shell；随后拆 `_MonitorLoop` 与
`_Watchdog`。不要在 D8 改动 `core.py/client.py` 期间同时做大规模拆分。

#### P2-2：组件文档大面积过期

`components/` 多数页面仍标记 `As-built @ d0de31a`，并保留已删除的 `Sample.resolve_token`、
`io_json`、旧行数和旧测试数量；`CODE_REVIEW_2.0.md` 仍把系统描述成 JSON-only。读者会据此重复
已经完成的工作或作出错误架构判断。

**建议**：以本报告为当前入口；按改动面优先刷新 `sample/calculator/io_system/factory/core/cli/
samplers_catalog/operas/datarecorder`，之后再做全目录机械状态刷新。每页增加“基线提交 + 最近验证
日期”，避免只写模糊的 as-built。

#### P2-3：真实 HEP 能力仍有明显缺口

Portal 扩展点已经落地，但当前注册格式没有 SLHA/xSLHA/HepMC；采样器仍只有 4 个无状态实现；
nuisance 是脆弱 stub，字段缺失会触发 KeyError；project tools 和完整 benchmark 模块仍未移植。

此外，测试套件唯一的 skip 明确记录了 flowchart export 尚未实现（V1 parity deferred）。

**建议**：D8 稳定后，按“SLHA → 一个代表性 MCMC → nuisance 显式收口”的顺序推进。nuisance
若短期不实现，应在配置校验阶段明确报 `NotImplementedError`，不要保留看似可用的半实现。

#### P2-4：可观测性指标仍不完整

`mean_active_workers`、`total_point_eval_sec`、`completed_durations_sec`、`retry_count` 现在诚实地
返回 `None`，比旧的假零值正确，但 run_summary 尚不能回答“时间花在哪里、扩展效率为何变化”。

**建议**：在 Worker/Factory 记录点级 duration、外部工具耗时、重试数和活跃 Worker 时间积分；
指标定义先写入 design，再实现采集，避免再次出现含义漂移。

#### P2-5：watchdog 仍存在重复执行窗口

心跳过期与 Worker 正在收尾可能竞态，任务可能被重入队；当前依赖 UUID/Archiver 幂等兜底。

**建议**：增加 stale-heartbeat-but-worker-finishes 的定向压力测试，并记录 duplicate suppression
计数。生产环境启用 Redis AOF/RDB 的策略也需形成运维文档。

## 5. 文档漂移与计划问题

1. 评审前 `V2_DISTRIBUTED_PLAN.md` 顶部日期仍是 2026-06-29，与 ledger 冲突；本次已同步为
   2026-07-10 并写明 D8/D9 状态。
2. 评审前 D9 milestone 仍写“保持 207 tests”；本次已更新为 236 collected / 235 passed /
   1 skipped。
3. D9.4、D9.5 原本依赖 D8.3/D8.2，但当前在 D8 未完成时已部分/全部落地；这是计划顺序偏差，
   应明确记录为“提前完成无冲突子集”，而不是假装依赖已满足。
4. `CODE_REVIEW_2.0.md`、`YAML_REFERENCE_2.0.md` 及多个 component 页面仍使用旧 HEAD 和旧能力描述。
5. 评审前根 README 缺少当前 review 入口；本次已新增本报告链接，但“当前支持/当前不支持”仍建议
   在代码仓库 README 中进一步明确。

## 6. 推荐执行路线

### Phase A（P0/P1，先恢复机器接口与错误可诊断性）

1. 给失败 result 增加结构化 error，并补测试。
2. D8.1：实现统一 JSON envelope、validate/results/version-json。
3. D8.3：控制进程 graceful stop + checkpoint，覆盖 SIGINT/SIGTERM 集成测试。
4. D8.2：实现原子 `run_state.json` 与 status/monitor-json。
5. D8.4：strict diagnostics，阻止明显错误配置进入长扫描。

### Phase B（收尾 D9，降低后续扩展成本）

1. 完成 D9.6，移除 `Sample.info` 关键字段的直接写入。
2. 完成 D9.4，测试脱离 singleton，再拆 monitor/watchdog。
3. 刷新受 D9 影响的组件文档，并把 236-collected 基线写入 plan。

### Phase C（领域能力）

1. Portal 增加 SLHA/xSLHA adapter 与真实 calculator golden fixture。
2. 选择一个代表性 MCMC（建议先做可 checkpoint 的 Metropolis-Hastings）验证注册表扩展点。
3. nuisance：实现完整契约，或在 validate 阶段显式拒绝。
4. 补齐 point duration / external tool duration / retry / worker utilization 指标。

## 7. 下一阶段完成定义（Definition of Done）

- 机器接口：所有 JSON 命令成功/失败均遵循统一 envelope，失败含结构化 error。
- 生命周期：SIGINT/SIGTERM 后有可恢复 checkpoint、最终 run_state、确定性退出码。
- 模型一致性：Worker 与 Likelihood 不直接写 `Sample.info` 的 params/observables/status。
- 配置：未知方法、未知 IO、Redis 缺项、死键在启动前被诊断。
- 回归：全量测试、真实 Redis 冒烟、D7 acceptance、Agent API contract tests 全绿。
- 文档：plan ledger、README、核心 component 页面与同一 HEAD/测试数同步。

## 8. 维护规则建议

以后每个 WP 合并时同时更新三处：`V2_DISTRIBUTED_PLAN.md` ledger、受影响 component 的
as-built 基线、本报告或其后继 review 的风险状态。测试数不要写成永久硬编码目标；建议写成
“当前基线 N，禁止减少既有契约覆盖”，并由 CI 产出 collection count 与 acceptance artifact。
