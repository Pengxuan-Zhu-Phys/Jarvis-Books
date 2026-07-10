# Jarvis-HEP V2 设计原则评审与重构计划（DESIGN_PRINCIPLES_REVIEW_2.0）

> 评审对象：`jarvishep2` @ `jarvis2` `d0de31a`（207 测试全绿）。
> 评审维度：**SOLID + DRY + GoF 模式使用是否恰当 + 模块分层**——与
> [`CODE_REVIEW_2.0.md`](CODE_REVIEW_2.0.md)（功能完整度评审）互补，两份合起来才是完整评审。
> 产出：§4 的重构工作包（里程碑 **D9 "Architecture Hardening"**），已登记进
> [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md)。
> 硬约束：**全部工作包必须行为保持**——207 测试不改断言全绿、V1 golden parity 不动、
> 冻结契约（YAML/CLI/输出）不动；与 D8（Agent Bridge）的执行顺序见 §4.3。
> 日期：2026-07-04。

---

## 1. 总体判断

**架构层（进程间）的模式运用是好的，债务集中在类/模块内部。**

V2 最难的设计问题——跨 spawn 进程的对象生命周期——解得干净且纪律贯彻到底：活对象
（Redis client、scheduler、logger、calculator）永远在子进程 `run()` 里构建，跨界只传
picklable 配置（`connection_config()` / `from_picklable()` / worker blueprint）。这实质上是
**进程边界上的依赖倒置**，也是整个代码库最值得保留的设计资产。

问题在类内部：三个"上帝文件"（`CalculatorModule` 485 行 / `Jarvis2Core` 676 行 /
`TaskFactory` 471 行）各自混装 4–6 个职责；同一段逻辑存在 2–3 份拷贝（token 解析 ×3、
sync/async 双胞胎、CommandParser 序列化 ×2、Archiver 构造/主循环 ×2、采样器 propose 循环 ×3）；
`Sample.info` 字典是一块跨模块共享的可变黑板，与 dataclass 字段构成双重真相。

按 SOLID 逐项打分：

| 原则 | 判定 | 一句话 |
|---|---|---|
| **S**RP | ✗ | 三个上帝文件（§3.3–§3.5）+ `Sample.info` 黑板（§3.6） |
| **O**CP | ⚠ | 两处硬编码分派点：`Distributor` match、`type == "JSON"` ×4（§3.7–§3.8）——恰好挡在两个已知扩展方向（MCMC 采样器、SLHA I/O）上 |
| **L**SP | ✓（小瑕疵） | 继承层次浅、行为一致；仅 `at_safe_barrier` 用裸 `NotImplementedError` 而非 ABC（§3.10） |
| **I**SP | ✓ | 接口普遍小而精（`UMapperProtocol` 单方法、`SubprocessJob` 数据类）；瑕疵是 `stateless_batch` 以"友元模块"姿态戳采样器私有成员（§3.9） |
| **D**IP | ✓/⚠ | 进程边界上优秀；进程内 `Worker._init_runtime` 硬编码全部具体类——spawn 场景下可接受，不强求改 |

---

## 2. 做对的地方（重构时**不要**动的资产）

列出这些是为了给 §4 的重构划红线——以下每一项都是有意识的正确设计，返工它们属于倒退：

1. **两阶段 `CommandParser`**（Pipeline/职责分离）：静态 token 控制端一次解析，`@SampleID/@Sdir/@PackID`
   Worker 端解析，阶段错配硬报错。
2. **Strategy 落地三处**：`mapper.py`（distribution/flat/identity + `build_mapper` 工厂 +
   `UMapperProtocol`）、`file_ops` 删除后端（shutil/rm）、Archiver 双形态（thread/process 共享
   `ArchiveProcessor` 核心——组合优于继承的正确示范）。
3. **spawn 可序列化纪律**（进程边界 DIP）：`Worker.__init__`/`ArchiverProcess.__init__` 只存
   picklable；`mp_context.get_spawn_context()` 是唯一 spawn 入口。
4. **`op_count` 门控监控**（Observer + 缓存）：写者 INCR、读者比对计数惰性刷新，快照读是纯内存
   deepcopy——读写分离贯彻。
5. **Checkpoint = Memento + 安全屏障**：原子写（temp + `os.replace`）、跨格式拒绝、
   `SeedSequence` 派生子流使结果与 Worker 数无关。
6. **`workflow.py` 纯函数**：无状态的 DAG 分层，最易测试的模块。
7. **`CheckpointedSampler` mixin**：checkpoint 关切与采样算法正交叠加，方向正确（§3.9 只是要求
   把更多共性也收进来）。
8. **错误信息带上下文**：`@PackID requires pack_id during stage …`、
   `registered executable … source does not exist` 这类可行动报错是普遍风格。

---

## 3. 发现清单（按严重度排序，全部带证据）

### 3.1 【DRY，高】token 解析存在三份实现

同一套 `@SampleID/@Sdir/@PackID` 替换逻辑写了三遍：

| 位置 | 状态 |
|---|---|
| `command_parser.py:195-243` `CommandParser.resolve_sample` | **正主**，最完整（含 Phase-1 残留检查） |
| `Module/calculator.py:195-228` `_resolve_runtime_tokens` 的 `parser is None` 回退分支 | 逐行复刻，但**没有** Phase-1 残留检查——两条路径对同一输入行为不同 |
| `sample.py:410-448` `Sample.resolve_token` | 第三份拷贝；生产代码**零调用**，只有 `tests/test_sample_taskdict.py:152,159` 在用 |

风险：改 token 语义（比如未来加 `@RunID`）要改三处，漏一处即静默不一致——这正是 V2 现在
`Sample.resolve_token` 缺 `has_static_tokens` 检查的现状。
**改法**：`CommandParser` 是唯一所有者；Worker 已无条件注入 parser（`worker.py:125-128`），
删掉 calculator 的回退分支（parser 缺失直接 `RuntimeError`，fail-fast）；删除
`Sample.resolve_token`，对应测试改为测 `CommandParser.resolve_sample`。

### 3.2 【DRY/YAGNI，高】`CalculatorModule` 的 sync/async 双胞胎是死代码

`load_input`/`_load_input_sync`（`calculator.py:337-346` vs `399-408`）、
`read_output`/`_read_output_sync`（`348-356` vs `410-418`）、
`execute_commands`/`_execute_commands_sync`（`377-397` vs `358-375`）、
`run_command` 的无 scheduler 分支（`309-335`）——每对函数体逐行相同或近似。
而 `Worker._init_runtime` **无条件**给每个 module `attach_scheduler`（`worker.py:127`），
V2 又只有一条运行路径，所以 `execute()` 里 `asyncio.run(self._execute_async(...))`
（`calculator.py:464`）在生产中**不可达**，仅个别单测直接构造无 scheduler 的模块时踩到。
约 120 行纯重复。**改法**：删除 async 双胞胎与无 scheduler 分支，scheduler 变成必备依赖；
受影响单测补 `attach_scheduler`。

### 3.3 【SRP，高】`CalculatorModule` 是上帝类

485 行一个类装了六种职责：配置解析（`__init__` 直接吃 raw dict）、clone_shadow 物理安装
（`ensure_shadow_installed`）、symlink 运行时（`ensure_symlink_runtime`）、env 绑定、token
解析回退（§3.1）、命令执行（§3.2 双路径）、JSON 文件 I/O（`_load_input_sync`/`_read_output_sync`）。
**改法**（在 §3.1/§3.2 清完重复之后做，改动面小得多）：拆成
`CalculatorSpec`（picklable 配置解析/校验）+ `RuntimePreparer`（shadow/symlink/env，策略二选一）+
瘦 `CalculatorModule`（编排 + 命令执行）；文件 I/O 收进 §3.8 的格式注册表。

### 3.4 【SRP + 模式误用，高】`TaskFactory` 单例 + 四职责混装

- **单例**：`get_instance`/`reset_instance`（`factory.py:54-68`）——`reset_instance` 这个
  "测试助手"的存在本身就是单例伤害可测试性的自供状态。更糟的是 `get_instance(redis_config)`
  对已存在实例做 `update`（`factory.py:60-61`）：第二个调用者传不同配置会**静默合并**进全局
  实例。全代码只有两个消费者（`core.init_factory`、`client.dispatch_monitor`），根本不需要
  全局态。
- **四职责**：Worker 生命周期（start/stop/respawn）、监控快照（op_count 门控）、watchdog
  判死/重入队、run_metrics 投影，全在一个 471 行类里。
- **诚实性问题**：`get_run_metrics` 硬编码 `total_point_eval_sec: 0.0`、
  `completed_durations_sec: []`、`mean_active_workers = float(alive)`（瞬时值冒充均值）
  （`factory.py:143-145`）——宁可返回 `None`/缺省字段也不要编数字，run_summary 消费端要能
  区分"没采集"和"确实是 0"。

**改法**：去单例——`Jarvis2Core` 持有普通实例并传递；`client.dispatch_monitor` 显式建实例；
`get_instance` 保留为 deprecated 薄壳一个版本。内部按职责拆 `_MonitorLoop`/`_Watchdog`
两个私有协作对象（组合，不必公开新类）。假指标改为 `None` 并在 run_summary 渲染端容错。

### 3.5 【SRP，中】`Jarvis2Core` 携带测试专用逻辑与硬编码

- `_verify_check_modules_golden`/`_normalize_check_module_records`/`_sample_tree_file_sets`
  （`core.py:196-242`）是**golden-parity 断言逻辑**，只被测试场景消费，却住在生产编排类里。
  → 迁到 `tests/`（或 `jarvishep2/testing/`）。
- `_build_check_module_samples` 硬编码 `row["x"], row["y"]`（`core.py:157`，即
  YAML_REFERENCE A.7）→ 从 CSV 表头/`Sampling.Variables` 推导列名。
- 小件：`command_parser` 重复 import（`core.py:19,21`）；`_command_parser_payload` 与
  `worker_config.py:111-124` 的序列化重复（见 §3.10 F10）。

### 3.6 【数据模型/封装，高·中期】`Sample.info` 是共享可变黑板，与字段构成双重真相

- `set_config()` 把 worker 的 `sample_config` **整个别名成** `self.info`（`sample.py:251`），
  之后配置键、运行时状态、logger 活句柄（`info["logger"]`）、路径、observables 全部混在
  这个 dict 里跨模块传递（Worker、CalculatorModule、CommandParser 都直接读写它）。
- 双重真相要手工同步：`self.status` vs `info["status"]`（`sample.py:454-455`）、
  `self.observables` vs `info["observables"]`（worker.py 在 `_merge_calculator_observables`/
  `_run_opera_step`/`_run_likelihood` 三处手工回写）、`_sync_logger_handles` 专职同步 logger。
  漏同步即静默数据错，且无类型检查兜底。
- `_attach_sample_for_materialize`（`sample.py:586-597`）为了让"只有 info dict 的调用方"
  复用物化逻辑，凭空造一个壳 Sample 再挂 info——是黑板模型逼出来的变通。
- `format_summary`（60 行纯文本渲染）住在领域模型里——表现层混进模型，小件。

**改法**（分两步走，见 D9.6）：第一步把"谁拥有哪些键"写成契约注释 + 收敛写入口
（新增 `Sample.merge_observables()` 之类的方法，Worker 不再徒手改 dict）；第二步让
`info` 变成**投影**（`to_info_dict` 已存在，方向就是它）——字段为唯一真相，黑板只读。
这是全库最大的一笔债，也最碰不得急——放 P2。

### 3.7 【OCP，中】`Distributor.set_method` 是硬编码 match

`distributor.py:24-45`：新增采样器 = 修改 match + 修改 `STATELESS_METHODS` + 修改
`RESUME_SUPPORT_STATUS` 三处。设计文档承诺"复用 V1 全采样器家族"（MCMC/Dynesty/MultiNest
是 CODE_REVIEW §7.2 的头部缺口），这个分派点是必然要过的门。
**改法**：注册表 + 装饰器（`@Distributor.register("Bridson", stateless=True, resume="implemented")`），
match 消失，三份清单收敛成注册项属性。

### 3.8 【OCP/Strategy，中】I/O 类型硬编码 `type == "JSON"` 且未知类型静默跳过

`calculator.py:340,351,402,413` 四处字符串比对；非 JSON 的 input/output spec **静默跳过**
（YAML_REFERENCE A.8 记录过）。SLHA/xSLHA 后端是 CODE_REVIEW 排名第一的功能缺口，现在的
写法意味着加 SLHA 要改四处 if。
**改法**：`io_registry: dict[str, IoBackend]`（`IoBackend = write_input(spec, params) +
read_output(spec)` 双方法协议），JSON 是首个注册项；未知 `type` 报
`ValueError("unknown IO type 'SLHA'; registered: JSON")`——静默跳过同时被修掉。
这是给 SLHA 后端**预开的插座**，不是过度设计。

### 3.9 【DRY/ISP，中】三个采样器复制同一套 propose 循环；`stateless_batch` 戳私有成员

- `bridson.py`/`randoms.py`/`grid.py` 各自手写同构的
  `_index/_accepted_index/_uuid_by_accepted_index/_u_by_uuid` + `propose_next` 的
  selection-过滤循环 + `set_config` 里的 `Seed/seed`、batch_size、selection 解析——三份拷贝
  （YAML_REFERENCE A.11 的 `length` 不一致正是这种复制漂移的产物）。
- `stateless_batch.flush_batch(sampler, …)` 以 `Any` 类型戳 `sampler._submit`/
  `sampler._submit_group`/`sampler._submitted_uuids`（`stateless_batch.py:28-34`）——
  模块级函数当"友元"，绕开封装且无协议约束。

**改法**：抽 `FixedSetSampler(CheckpointedSampler)` 模板方法基类（钩子：`_generate_candidates()`、
`_candidate_to_u(row)`、`_uuid_prefix`），三个采样器各剩生成算法本身；`flush_batch`/
`run_stateless_distributed` 变成基类方法，私有成员访问回到类内部。

### 3.10 【小件打包】F10–F13

- **F10 序列化重复**：`CommandParser` 有 `from_picklable` 无 `to_picklable`，于是
  `core.py:424-439` 和 `worker_config.py:111-124` 各写了一份 payload 组装。加
  `to_picklable()` 成对，两处调用它（对称性缺失的教科书案例）。
- **F11 分层结**：`command_parser._registered_specs` 函数内延迟 import `runtime_config`
  （`command_parser.py:284-287`）躲循环依赖——`parse_registered_executables` 唯一消费者就是
  command_parser，把函数搬过去，环消失；`sample.py:14` import `runtime_config`（模型 → 配置层）
  顺手在 D9.6 里解。
- **F12 Archiver 重复**：`SimpleArchiver.__init__`（`archiver.py:159-167`）与
  `ArchiverProcess.run`（`256-264`）重复展开同一份 archiver_config → 加
  `ArchiveProcessor.from_config(writer, cfg)`；`SimpleArchiver._run_loop:194` 直接读
  `processor._batch` 私有成员 → 给 processor 加 `has_pending()`；两个 drain 主循环
  （`_run_loop` vs `ArchiverProcess.run`）可共享一个模块级 `drain_loop(redis, processor, stop)`。
- **F13 杂项**：`factory.request_worker_shutdown` 循环内重复 local import `os/signal`
  （`factory.py:168-169`，顶层已 import）；`worker_config.py:78` 的 `__import__("os")`；
  `CheckpointedSampler.at_safe_barrier` 裸 `NotImplementedError` → `abc.abstractmethod`；
  `Worker._run_calculator_steps` 每层每样本新建 `ThreadPoolExecutor`（`worker.py:205`）——
  慢 regime 下开销可忽略，只登记不强改。

---

## 4. 重构计划 — 里程碑 D9 "Architecture Hardening"

### 4.1 总原则

1. **行为保持**：每个 WP 单独提交，提交后 207 测试全绿 + `tests/test_distributed_acceptance.py`
   的 parity 门不动；测试只允许"搬家/补充"，不允许改断言语义。
2. **先删重复，再拆结构**：D9.1/D9.2 是机械修改，先做——它们把 D9.3 的拆分面积削掉一半。
3. **红线**：§2 清单里的资产不返工；不引进 DI 框架、不做 ABC 化运动、
   **保持扁平包布局**（plan §4 硬不变量）。

### 4.2 工作包

| WP | 内容（对应发现） | 优先级 | 风险 | 验收 |
|----|----|----|----|----|
| **D9.1** | token 解析收敛到 `CommandParser`（删 calculator 回退 + `Sample.resolve_token`）；删 sync/async 双胞胎；`CommandParser.to_picklable()` 替换两处手写 payload（F1, F2, F10） | **P0** | 低（删死代码 + 等价替换） | 全绿；`calculator.py` 减 ≥150 行；grep 确认 `@SampleID` 替换逻辑仅存一处 |
| **D9.2** | `Distributor` 注册表化；I/O 格式注册表 + 未知类型报错（F7, F8） | **P0** | 低 | 全绿；新增"未知 IO 类型报错"与"注册新采样器不改 Distributor 源码"两条单测 |
| **D9.3** | `CalculatorModule` 拆分：`CalculatorSpec` + `RuntimePreparer` + 瘦执行编排（F3） | P1 | 中 | 全绿 + clone_shadow/symlink/env 套件不动；单类 ≤200 行 |
| **D9.4** | `TaskFactory` 去单例（core 持有实例、client 显式构造、`get_instance` 弃用壳）；内部拆 `_MonitorLoop`/`_Watchdog`；`get_run_metrics` 假零值改 `None`（F5） | P1 | 中（`--monitor` 路径联动） | 全绿；`reset_instance` 从测试里消失；run_summary 渲染端容 `None` |
| **D9.5** | `Jarvis2Core` 清理：golden 验证逻辑迁 `tests/`；check_modules 列名从 CSV 表头推导（修 A.7）；重复 import 清理（F6） | P1 | 低 | 全绿；`core.py` 不含 `_verify_*`；check_modules 支持任意参数名 CSV（新测一条） |
| **D9.6** | `Sample` 单一真相：写入口收敛为方法 → `info` 降级为投影；`format_summary` 迁出模型；解 `sample→runtime_config` 依赖（F4, F11 部分） | P2 | **高**（触所有链路） | 全绿 + parity；`worker.py` 不再徒手写 `info["observables"]` |
| **D9.7** | `FixedSetSampler` 模板基类；`stateless_batch` 函数收编为基类方法；顺手统一 `length` 默认值语义（联动 YAML_REFERENCE A.11 的修复决策）（F9） | P2 | 中 | 全绿 + `test_samplers_catalog` 不改断言；三采样器合计减 ≥150 行 |
| **D9.8** | 小件清扫：`parse_registered_executables` 搬家解环、Archiver `from_config`/`has_pending`/共享 drain 循环、factory 死 import、`__import__("os")`、`at_safe_barrier` ABC 化（F11–F13） | P2 | 低 | 全绿；`command_parser` 无函数内 import |

### 4.3 与 D8 的排序

D9.1/D9.2/D9.7/D9.8 与 D8 无文件交集，可并行。**D9.4/D9.5 触碰 `core.py`/`client.py`，
必须排在 D8.1–D8.3 落地之后**（Agent API 动词也要加在这两个文件上，先加功能后做结构，
避免双向 rebase）。D9.6 最后做。

### 4.4 刻意不做（含回补条件）

| 不做项 | 理由 | 回补条件 |
|---|---|---|
| DI 容器 / 服务定位器 | spawn 边界的 picklable 配置就是本项目的注入机制 | 不回补（立场性拒绝） |
| 全面 ABC 化 / 类型体操 | 协议小而鸭子类型工作良好；只修 `at_safe_barrier` 一处 | 不回补 |
| 包目录重组（分层子包） | plan 硬不变量：扁平布局 | 模块数 >60 时重议 |
| 重写 `async_subprocess.py` | 679 行但自洽、有测试、无重复 | 出现真实缺陷时 |
| 配置层 jsonschema 化 | 属于功能补全（CODE_REVIEW 建议 ③ / D8.4 诊断），不是重构 | 走 D8.4 |
| YAML 键名清理（`make_paraller` 等） | 用户可见表面，需单独决策 | YAML_REFERENCE A.4/A.5 评审后 |

---

## 5. 结论

V2 不需要"推倒重来"级别的重构：**进程架构是对的，模式误用只有一处（TaskFactory 单例），
其余都是可机械消除的重复与可渐进的拆分**。按 D9.1→D9.8 的顺序执行，预计净删 400–500 行、
新增 3 个小类 + 2 个注册表，同时为 MCMC 采样器与 SLHA I/O 这两个头号功能缺口预开好扩展点
——重构与功能路线图（CODE_REVIEW §8）在 D9.2 上正好合流。
