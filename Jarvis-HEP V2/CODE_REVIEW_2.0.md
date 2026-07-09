# Jarvis-HEP V2 代码评审报告（CODE_REVIEW_2.0）

> 评审对象：`jarvishep2`（独立包，`2.0.0.dev0`），分支 `jarvis2`，最新提交 `d0de31a`。
> 评审方式：对 `jarvishep2/` 全量源码（50 个模块、约 75 个类）的静态通读 + 结构清单核对，
> 配合 `tests/`（26 个测试文件、约 210 个 `test_` 函数）。本次评审同时把
> [`components/`](components/) 下的全部组件文档刷新为 **as-built**（按实际代码逐类/逐函数/逐属性
> 记录），本报告与那批文档的「Drift from design」小节保持同一口径。

---

## 1. 概览

| 项 | 数值 |
|----|------|
| 主包源码 | `jarvishep2/`，50 个 `.py`，约 9300 行 |
| 类 | 约 75 个（含 dataclass / 异常类） |
| 公开 API | `jarvishep2/__init__.py` 导出 17 个符号 |
| 测试 | 26 个 `tests/test_*.py`，约 210 个用例 |
| 依赖 | 核心 `numpy/scipy/sympy/h5py/PyYAML`；分布式可选 `redis/msgpack/aiofiles`；开发 `pytest/fakeredis/colorlog` |
| 入口 | `Jarvis2 = jarvishep2.client:main` |

**总体判断**：项目已经是一个**可运行、可复现、可断点续跑的分布式扫描运行时**——
Redis 单一 broker + spawn 进程 Worker + 异步 Archiver 的主链路完整闭合，并配齐了
监控快照、运行总结、watchdog 重生、SeedSequence 可复现 checkpoint。**核心工程质量高**。
但它实现的是设计文档里**一个收窄的子集**：采样器只有 4 个无状态实现，I/O 后端只有 JSON，
若干设计组件（benchmark、nuisance、project tools、utils、jsonschema 校验、MCMC/嵌套/梯度采样器）
**未落地**。下面逐项展开。

---

## 2. 架构落地 vs 设计

参照记忆 [[jarvis-v2-redis-pivot]]：V2 放弃了 throughput-core（M2–M5），转向
**Redis + 进程 Worker + 异步 Archiver**，当时只交付了 M0/M1。**本次评审确认实际进度远超 M0/M1**，
对照 `V2_DISTRIBUTED_PLAN.md` 的 D 系列里程碑，已落地的有：

- **D0** Sample/RedisQueue/Logger/UMapper 基座；
- **D1** Worker MVP（opera）+ Calculator + TaskFactory + Archiver MVP；
- **D2** Worker 同层 calculator 并发（`ThreadPoolExecutor`）、Redis 自由池并发上限、
  `clone_shadow`/symlink 隔离；
- **D3** 两阶段 `CommandParser`、`env_setup` 捕获、`registered_executables`、`delete_method`；
- **D4** staging 交接 + Archiver 批处理 + 线程/进程两种 Archiver；
- **D5** Factory `op_count` 门控监控快照 + `--monitor` + 冻结 `run_summary`；
- **D6** watchdog 重生 + 在飞任务重入队 + SeedSequence 可复现 checkpoint/resume；
- **D7** 慢regime 分布式验收门（`tests/test_distributed_acceptance.py` + `docs/benchmarks/d7_1_acceptance.json`）。

主数据链路（**采样器 propose → `hep:task_queue` → Worker 执行 → `hep:archive_queue` →
Archiver → HDF5 DATABASE + `SAMPLE/<uuid>/`**）端到端打通，`Jarvis2Core.run()` 调度，
`--resume`/`--monitor`/`--check-modules` 已接入 CLI。**不变量基本守住**：`logger` 不跨进程、
任务字典是「轻量 id + dict」、全程 `spawn`、`jarvishep2` 不 import V1（已核验，零泄漏）。

---

## 3. 设计 ↔ 代码漂移清单

这是本次评审最重要的产出：**文档库原本是设计稿，与落地代码已大面积脱节**。已在各组件文档
逐条标注，汇总如下。

### 3.1 重命名 / 改址（文档路径 → 实际路径）

| 设计文档引用 | 实际代码 | 备注 |
|--------------|----------|------|
| `mapping.py` | `mapper.py` | 而且由单一 `UMapper` 拆成 3 个 mapper 类 + `build_mapper` 工厂 |
| `hdf5writer.py`（`GlobalHDF5Writer`/`observable_io`） | `database.py`（`SimpleHDF5Writer`） | 改为「一行 JSON 编码 observables 存 HDF5」 |
| `expression.py`（`ExpressionEngine`） | `inner_func.py` + `Sampling/sampling_utils.py` | 无编译缓存引擎，只有 `LogGauss` + `evaluate_selection` |
| `config.py`（`ConfigLoader` + jsonschema） | `runtime_config.py` + `task_config.py` + `worker_config.py` | **无 jsonschema 校验、无环境检查** |
| `IOs/`（SLHA/xSLHA/portal + registry） | `io_portal.py` + `file_ops.py` | **Portal 注册表**（JSON/CSV/TSV/DAT/Wolfram；格式逻辑在 Jarvis-Portal） |
| `Module/operas.py` / `Module/likelihood.py` | `operas.py` / `likelihood.py` | 顶层模块，独立类 |
| `Module/module.py`（`Module` ABC） | 无 | **无共享基类**，Worker 按 step 类型分派 |
| `base.py`（`Base` 类） | `base.py`（一组函数） | 无 `Base` 继承体系 |
| `card/argparser.json` | 无 | CLI 用普通 `argparse` 写在 `client.py` |

### 3.2 设计有、代码无（未建 / 仅 stub）

| 组件 | 实际情况 |
|------|----------|
| `benchmark.py` / `--benchmark` / `parallelism_efficiency` | **未建**。最接近的是 `core.run_check_modules()` + `docs/benchmarks/d7_1_acceptance.json` |
| nuisance（`Module/nuisance_*`、`nuisance_sampler`、`profile1d`） | **仅 stub**：`Sample.combine_nuisance_card()`/`gather_nuisance()` 近乎 no-op；`nuisance_optimize` 只在 step 类型枚举里 |
| project tools（scaffold/packager/official_library + `Jarvis2 project …`） | **未移植** |
| utils/versioning/dataconvert/plot/observable_io（含 HDF5→CSV `--convert`、绘图、版本横幅） | **未移植** |
| **MCMC / 嵌套 / 梯度采样器全家族**（`mcmc_*`、`tpmcmc`、`dream*`、`ess`、`dynesty`、`multinest`、`mala/hmc/nuts`…） | **完全未建**。V2 只有 4 个无状态采样器 |
| jsonschema 校验 + 环境检查（`check_ROOT`/`check_PYTHON_env`/OS 检查） | **未建**，配置仅做类型规整 + 默认值 |

### 3.3 代码有、设计无（新增、未在设计文档登记）

`archive_handoff.py`、`calculator_pools.py`、`mp_context.py`、`stateless_batch.py`、
`seeded_sampler.py`、`checkpointed_sampler.py`、`monitoring/run_summary.py`、`testing/eggbox.py`、
以及 RedisQueue 新增的 `CALC_BUSY_PACKS`/`sweep_held_calc_slots`/心跳任务编解码等。

---

## 4. 各组件实现完整度

| 组件 | 状态 | 一句话 |
|------|------|--------|
| Sample / ExecutionStep | ✅ done | 富 dataclass，序列化契约 + 懒物化 + token 解析齐全 |
| RedisQueue + calculator_pools | ✅ done | key 命名空间、编解码、校验、监控读、自由池均完整 |
| Logger（toplevel + sample_logger + log_kv） | ✅ done | 两层日志 + 队列监听 + 失败回放 |
| UMapper（mapper.py） | ✅ done | 3 个 mapper（distribution/flat/identity）+ 工厂 |
| Variables | ✅ done | Flat/Log/Normal/Log-Normal/Logit，边界已 clip |
| Workflow | ✅ done | 纯函数，分层执行计划（calc→opera→likelihood） |
| Worker | ✅ done | 全链路；同层并发用 ThreadPool；staging + cleanup + 心跳 |
| CalculatorModule | ✅ done | 模板预载、异步/同步执行、clone_shadow、token 解析 |
| OperasModule | ✅ done | preload-per-Worker，sympy lambdify 输入表达式，超时线程 |
| AsyncSubprocessScheduler | ✅ done | 单循环有界调度，流式 drain，超时 kill，快照 |
| TaskFactory | ✅ done | 生命周期 + op_count 门控快照 + watchdog 重生 |
| CommandParser | ✅ done | 两阶段解析 + registered_executables |
| env_setup | ✅ done | source 捕获 + 进程级缓存 |
| Library | ⚠️ thin | 仅 symlink + 读 registered，无 install_all/InstalledDep |
| LogLikelihood | ✅ done | sympy lambdify 编译，多项求和 |
| Archiver（archiver/database/handoff/file_ops/io_json） | ✅ done | 线程/进程两版，幂等重启，JSON-rows HDF5 |
| Monitor（dashboard + run_summary） | ✅ done（纯文本） | 只读快照 + 冻结 run_summary，无 TUI |
| Checkpoint（runtime_checkpoint + checkpointed_sampler） | ✅ done | 双条件安全屏障 + SeedSequence + 跨格式拒绝 |
| Samplers（Bridson/Random/Grid/CSV/Seeded） | ✅ done（收窄） | 4 个无状态 + 1 个测试用 seeded |
| Distributor | ✅ done（收窄） | 只分派 4 个方法 |
| Core / client | ✅ done | bootstrap + run + shutdown + CLI |
| config（task/runtime/worker_config） | ✅ done（无 schema） | 规整 + 默认值，非 jsonschema |
| base（paths_tokens） | ✅ done | `&J/`/路径解码函数 |
| io_system | ✅ Portal bridge | `io_portal` → Jarvis-Portal registry；未知 type 硬报错 |
| Module base | ⚠️ 无 ABC | 两后端独立类 |
| benchmark / nuisance / project_tools / utils | ❌ 未建/stub | 见 §3.2 |

---

## 5. 代码质量观察

### 优点

1. **spawn 可序列化贯彻到底**：`Worker.__init__`/`ArchiverProcess.__init__` 只存 picklable
   配置，活对象（Redis 客户端、scheduler、calculator、asyncio loop）全部在子进程 `run()` 里建；
   `RedisQueue.connection_config`/`extract_connection_config`、`CommandParser.from_picklable`
   都是为跨进程而设计。`mp_context.get_spawn_context()` 是唯一的 spawn 入口（不变量 #10）。
2. **两阶段 CommandParser**：静态 token（`&J/`/`${LibDeps}`/`${Scan}`/registered names）在控制端
   一次解析完，Phase 2 在 Worker 只解析 `@SampleID/@Sdir/@PackID`，并对「阶段错配」硬报错——
   设计意图被忠实实现。
3. **可复现性**：`SeededOperaSampler` + `derive_sample_seed` 用主 `SeedSequence` 派生子流，
   使结果与 Worker 数无关；checkpoint 原子写（temp + `os.replace`）、跨格式拒绝（V1/throughput-core
   明确报错）、双条件安全屏障（采样器静止 **且** Archiver 已 ack）都到位。
4. **并发安全细节**：calculator 槽位 `acquire/release` 在 `finally` 释放且记录 `held_calc_packs`
   供 watchdog 清扫；同层 observables 合并加 `_observables_lock`；RedisQueue 多键写用 pipeline 事务。
5. **可测试性**：`make_fakeredis_queue` + `EnvCapture.set_runner` 注入点让单测无需真实服务；
   Archiver/监控的只读不变量有专门测试守护。
6. **错误处理总体不吞**：路径/配置/表达式错误大多 `raise` 并带可操作信息（如 `@PackID requires
   pack_id …`、`registered executable … source does not exist`）。

### 关注点

1. **私有 `_*` 方法密集、单文件偏大**：`async_subprocess.py`(679)/`core.py`(675)/`sample.py`(643)
   方法众多，`Jarvis2Core` 30+ 方法、`Worker`/`CalculatorModule` 各 20+，可读性尚可但单元测试
   颗粒度偏粗（多靠集成测试覆盖）。建议对 `_archive_one`、`process_task` 等关键私有路径补针对性单测。
2. **Worker 关键 except 偏宽**：`process_task` 用 `except Exception` 统一转 Failed——对健壮性是对的，
   但会掩盖编程错误（如 KeyError）与真实计算失败。建议日志里区分异常类型，便于排障。
3. **同层并发用线程而非设计中的 scheduler**：`Worker._run_calculator_steps` 用
   `ThreadPoolExecutor` 扇出，再各自向**每 Worker 的** `AsyncSubprocessScheduler` 提交。
   功能正确，但与文档原描述（用 scheduler 直接并发）不同，且线程 + asyncio loop 双层需注意 GIL 与
   句柄压力（scheduler 已有 fd/rss best-effort 快照，可观测）。
4. **watchdog 重入队竞态**：`_handle_worker_failure` 以 `_recovery_lock` + dead-pid 去重，
   `_requeue_in_flight_task` 给任务打 `_retry_count`；但「心跳过期判死」与「Worker 实际仍在收尾」
   之间存在窗口，极端情况下可能重复执行同一 Sample——靠 Archiver 的 uuid 幂等兜底（present
   `SAMPLE/<uuid>/` 短路），属可接受设计，但建议在验收里专门压这条路径。
5. **I/O 后端单一（仅 JSON）**：真实 HEP 工具普遍用 SLHA/HepMC，当前 `io_json` 仅够
   parity-project 规模。这是落地范围的最大功能缺口（见 §7）。
6. **nuisance 仅 stub**：`Sample.combine_nuisance_card` 直接索引 `card["NAttempt"]`/`card["active"]
   ["param"]`，缺字段会 KeyError；由于无人真正驱动 nuisance 卡，目前是「死代码 + 潜在崩溃点」。
7. **`SimpleHDF5Writer` 每条记录开/关一次文件**：`add_record` 每次 `h5py.File(..., "a")` 打开、
   resize、写一行、关闭。批量场景下是明显的 I/O 放大；Archiver 虽有 `batch_size` 聚合 ingest，
   但写盘仍逐行。慢regime 可接受，量大时建议改为持有句柄 + 周期 flush。

---

## 6. 测试覆盖

26 个测试文件、约 210 个用例，覆盖矩阵（按用例数排序的代表项）：

| 测试文件 | 用例 | 覆盖面 |
|----------|------|--------|
| `test_worker_mvp.py` | 17 | opera 端到端、spawn、优雅停机、op_count |
| `test_sample_taskdict.py` | 16 | 序列化往返、不带 logger 上线、懒物化、失败回放 |
| `test_redis_queue.py` | 16 | 任务/结果/自由池/编解码/管线原子性 |
| `test_file_ops.py` | 16 | 删除后端 shutil/rm、不安全路径拒绝 |
| `test_samplers_catalog.py` | 14 | 4 采样器 propose/uuid/选择/分布式提交/状态往返 |
| `test_command_parser.py` | 12 | 两阶段分离、registered、Phase-2 解析 |
| `test_logging_layers.py` / `test_env_setup.py` | 11 / 11 | 两层日志 + 失败回放 / env 捕获缓存 |
| `test_distributed_resume.py` | 8 | 断点续跑、不丢不重、跨格式拒绝 |
| `test_worker_calculator.py` | 8 | calculator parity、token、失败路径 |
| `test_distributed_acceptance.py` | 6 | 慢regime 验收门（D7.1） |
| 其余 | — | layer 并发、clone_shadow、monitor 快照、archiver 交接/parity、dashboard 只读、cli 等 |

**优点**：用 `fakeredis` + 真 spawn 的组合既快又贴近真实；幂等续跑、自由池上限、只读监控等
关键不变量都有专测。

**缺口**：
- **无 SLHA/xSLHA I/O 测试**（因后端未建）；
- **无 benchmark/效率指标测试**（功能未建）；
- **project_tools/utils 零覆盖**（未移植）；
- 关键私有路径（`_archive_one` 各分支、watchdog 判死窗口）多靠集成隐式覆盖，**缺针对性单测**；
- nuisance 路径无测试（且代码仅 stub）。

---

## 7. 风险与缺口（按影响排序）

1. **I/O 后端只有 JSON** —— 直接限制能接入的真实计算器（SLHA/HepMC 工具用不了）。这是把 V2
   从「parity-project demo」推向「生产 HEP 扫描」的**头号阻塞**。
2. **采样能力收窄到 4 个无状态采样器** —— 没有 MCMC/嵌套/梯度，意味着贝叶斯后验采样、证据计算等
   主流 HEP 工作流暂不支持。设计文档承诺的「全采样器复用」**远未兑现**。
3. **nuisance 未实现且 stub 含潜在崩溃点** —— 若有人真填 `Sampling.nuisance`，
   `combine_nuisance_card` 会按 V1 卡格式硬索引，缺字段即 KeyError。建议要么补全、要么把 stub
   改成显式 `NotImplementedError`，避免「看起来支持」。
4. **无 schema 校验** —— 配置错误（拼写、缺字段）多在运行期才暴露，而非启动期 fail-fast。
   对长扫描体验不佳。设计里的 jsonschema 契约 + 启动期表达式校验未落地。
5. **watchdog 判死窗口的重复执行风险** —— 靠 Archiver uuid 幂等兜底，理论安全，但缺专门压测。
6. **`benchmark`/`run_summary` 的效率指标空缺** —— `run_summary` 里 `avg_point_eval_sec`、
   `time_in_external_tools_sec`、`parallelism_efficiency` 等字段大多取不到值（Factory 未采集
   逐点耗时），输出存在但「半空」。

---

## 8. 建议（按优先级）

1. **（已完成）文档与代码对齐** —— 本次已把 `components/` 全部刷新为 as-built，并补齐
   benchmark/nuisance/project_tools/utils 的「未建」横幅、修正 `README.md` 索引与
   `V1_TO_V2_MAP.md` 的 as-built 附注。后续改代码时请同步维护对应组件文档的「Drift」小节。
2. **明确范围声明** —— 在 `README.md`（仓库根）写清 V2 当前**支持范围**：4 个无状态采样器 +
   JSON I/O + opera/calculator/likelihood 工作流；并把「未建」清单（§3.2）作为 roadmap。避免按
   设计文档预期使用而踩空。
3. **优先级排序的功能补全**：① SLHA I/O 后端（解锁真实计算器）；② schema/启动期校验
   （改善 fail-fast）；③ 至少一个 MCMC 采样器（解锁后验采样）。
4. **把 nuisance stub 收口** —— 要么实现，要么 `raise NotImplementedError`，并加一条测试。
5. **补关键私有路径单测** —— `ArchiveProcessor._archive_one` 的各分支、watchdog 判死/重入队、
   `SimpleHDF5Writer` 批量写。
6. **HDF5 写盘优化** —— Archiver 持有 h5py 句柄 + 周期 flush，替代逐行开关文件（量大时再做）。

---

### 附：本次同步刷新的文档清单

- `components/` 下 28 个组件文档全部重写为 as-built（含 [sample](components/sample.md)、
  [redis_queue](components/redis_queue.md)、[worker](components/worker.md)、[core](components/core.md)、
  [factory](components/factory.md)、[samplers_catalog](components/samplers_catalog.md)、
  [datarecorder](components/datarecorder.md)、[checkpoint](components/checkpoint.md) 等）；
- 4 个「未建/stub」文档加显著横幅：[benchmark](components/benchmark.md)、
  [nuisance](components/nuisance.md)、[project_tools](components/project_tools.md)、
  [utils](components/utils.md)；
- 索引 [components/README.md](components/README.md) 与 [V1_TO_V2_MAP.md](components/V1_TO_V2_MAP.md)
  已加 as-built 修正。
