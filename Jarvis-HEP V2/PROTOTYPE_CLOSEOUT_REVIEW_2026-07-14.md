# Jarvis-HEP V2 原型阶段收尾评审与下一阶段（Calculator / 体验对齐）规划

**评审日期**：2026-07-14
**代码基线**：`Jarvis-HEP-v2` / `jarvis2` 分支，`0a5e85e`（expression 统一 + EnvReqs.V2 已提交；评审时对照 `6b9841e` + 工作树）
**对照基线**：冻结的 `Jarvis-HEP` V1（loguru 日志、flowchart、project 子命令、official library）
**受众**：V2 维护者、执行编码的 Agent（Grok/Codex）
**结论性质**：原型阶段（D0–D10 主线 + Operas 表达式桥）判定为**可以收尾**；下一阶段主题是 **Calculator 的 V1-YAML 对齐**与**用户体验对齐**（日志、flowchart、EnvReqs、project、官方例子库）。

---

## 1. 结论先行

1. **本次改动已提交为 `0a5e85e`。** 核心贡献是 `expression.py`（compile-once 表达式运行时）统一了此前 5 处各自为政的 sympify 调用点（Operas input、LogLikelihood、selection、Portal Dump、AdaptiveLevelSet target），并恢复了 V1 的完整轻量函数面（`inner_func.py`：全套三角/双曲/`Min`/`Max`/`Gauss`/`Normal`/`Heaviside` + `Pi/pi/PI/E/Inf` 常量）。
2. **有 2 个应在提交前或紧随其后修复的实质问题**（§3.1、§3.2）：Operas 误触发扫描（把 calculator 命令行里的 `foo.bar(...)` 误判为 Operas 函数并硬性要求安装 Operas）、以及 registry 快照绕过签名过滤。
3. **Calculator 已有骨架不是从零开始**：Spec/RuntimePreparer/Portal IO/sync execute/clone_shadow/env_setup 都在。真正缺的是 **V1 YAML 形态兼容**（最典型：V1 的 `installation`/`commands` 是**字符串列表**，V2 现在会**静默丢弃**非 mapping 项，§4.1）。
4. 用户点名的五个体验缺口全部属实，已编入 D12 工作包（§5）：core 日志系统、flowchart、EnvReqs 归口、project 支持、Jarvis-Examples 官方例子接口。

---

## 2. 本次改动审阅（现已落地 `0a5e85e`）

### 2.1 值得肯定的设计

| 改动 | 判定 | 说明 |
|---|---|---|
| `expression.py`：`ExpressionContext` + `CompiledExpression` | **正确方向** | 进程本地、带锁、带 LRU 的 compile-once 缓存；`MissingExpressionVariablesError` 携带缺失变量名。namespace 优先级（常量 > Operas namespace 对象 > observable 同名符号）有注释说明意图。 |
| `inner_func.py`：V1 函数面冻结恢复 | **正确方向** | 与 V1 `_Inner_FCs` 对齐；`sinc` 的 NumPy 归一化差异有注释；`Heaviside(0)=1/2` 与 V1 一致。 |
| `operas_functions.py`：启动期发现 + 快照 | **正确方向** | `namespace.function(...)` 的 V1 YAML 表面被保留，HEP 不新增函数注册块；entry-point 只发现一次（按 registry id 去重，含测试复位钩子）。 |
| `operas.py`：预编译 input 表达式 + Sample logger 逐条回放 | **对齐 V1 体验** | `Evaluating <name>: expression -> ... with input -> ... Output -> ...` 的样式和 V1 sample log 一致；NumPy 标量/数组做了可读化。 |
| `task_config.py`：`EnvReqs.V2` + 拒绝顶层 `Runtime` | **符合既定方向** | V1 的 `EnvReqs.Check_default_dependencies.default_yaml_path` 入口被保留，外部 defaults 与任务内 `EnvReqs.V2` 深合并、任务值优先；`worker`/`workers` 单复数兼容且互斥。 |
| `client.py`/`core.py`/`redis_queue.py`：Redis 内部化 | **可接受，但需决策记录** | 删除 `--redis-host/port/db`；空配置不再静默 fakeredis（这曾是 INSTALL.md 附录里的著名坑），改为连接 `127.0.0.1:6379` 并 `ping()` 早失败。与"Redis 设置放 EnvReqs"的最新方向存在张力，见 §4.3。 |
| 采样器接 `ExpressionContext`（selection / ALS target） | **正确方向** | Bridson/Grid/Random/CSV/ALS 全部经 context 传递，Worker 端 context 由 `_init_runtime` 统一构建。 |

### 2.2 测试与验证

- 新增 `tests/test_expression.py`（V1 轻量函数兼容矩阵）、`tests/test_operas_functions.py`（动态发现 + V1 YAML 兼容）、`tests/test_task_config_compat.py`（EnvReqs 合并/校验）。
- 新增 `tests/parity_project/bridson_operas_v1.yaml`：V1 形态（无 Runtime 块）的 Eggbox/Operas 冒烟。
- 全量测试结果见 §6。

---

## 3. 提交前应修复的问题（按严重度排序）

### 3.1 Operas 误触发：正则扫描范围过宽（中高）

`operas_functions.expression_uses_operas_function` 用 `\b\w+(\.\w+)+\s*\(` 在**整个配置子树**上递归匹配：

- `Sampling/sampler.py` 的 `set_config` 扫 `self.config` 全量；
- `worker.py` 的 `_init_runtime` 扫 `opera_modules + calculator_modules + likelihood_expressions` 全量——**包含 calculator 的 `cmd` 字符串**。

后果：calculator 命令里的任何点号调用（如 `python3 -c "numpy.savetxt(...)"`、`awk 'BEGIN{...}' f.txt` 中的 `xxx.yyy(`、shell 函数）都会令 `build_operas_expression_context(required=True)` 抛 `ImportError`，即使任务根本不用 Operas 表达式。**修复建议**：只扫描真正的表达式字段（`Operas.Modules[].input[].expression`、`LogLikelihood[].expression`、`selection`、Portal Dump `variables[].expression`），不要扫 `cmd`/`path`/`installation` 等命令与路径字段。

> **用户决议（2026-07-14）**：Jarvis-Operas 升级为 V2 **核心依赖**（从 `[operas]` extra 移入 `pyproject.toml` 主依赖），装 V2 即自动安装。最坏后果（ImportError 起不来）随之消失，本项严重度降为**顺手修**。扫描收窄仍保留在 D12.0：误判会触发不必要的 entry-point 发现（启动开销），且"任务是否使用 Operas 表达式"的判断本身应当准确。同步改动：README 插件表、INSTALL.md 安装说明中 Operas 不再是 extra。

### 3.2 registry 快照绕过 `registry.call` 的调用协议（中）

`_snapshot_operas_registry_operator` 直接取 `declaration.numpy_impl` 并在 `execute()` 里把 `observables`、`logger` 和全部 input observables 一并作为 kwargs 传入。对 `**kwargs` 风格的 helper 没问题；对严格签名的算子会 `TypeError`（此前 `registry.call` 会做签名过滤）。README 已把"签名过滤"列为 follow-up——建议在 D11.4 里与严格 `call_mode` 校验一并落地：快照时用 `inspect.signature` 冻结可接受参数集，调用时过滤。

> **用户决议（2026-07-14）**：确认给 Opera 调用加一层**参数解析层**（argument-resolution layer），设计如下：
> 1. **快照期**（每 Worker 一次，在 `_snapshot_operas_registry_operator` 内）：`inspect.signature(numpy_impl)` 解析并冻结参数集，区分三类——具名参数 / 是否声明 `logger`、`observables` / 是否有 `**kwargs`；
> 2. **调用期**：有 `**kwargs` → 保持现行为全量传入；否则只传函数声明的参数名（从 input observables 取值），`logger`/`observables` 仅在函数显式声明时传入；
> 3. **错误升级**：函数必需参数在 observables 中缺失时，抛出人话错误（"算子 `X` 需要参数 `z`，当前样本仅有 `x, y`"），替代裸 `TypeError`。
> 性能不受影响（signature 仅预加载时解析一次）。归入 D11.4 与严格 `call_mode` 校验一并实现。

### 3.3 模块级全局表达式上下文（低中）

`io_portal.py`（`_IO_EXPRESSIONS`/`_IO_OPERAS_LOADED`）与 `sampling_utils.py`（`_SELECTION_*`）持有模块级可变全局，无锁、上下文可能在运行中途被整体替换（丢弃已编译缓存），测试需依赖 `reset_operas_discovery_for_tests`。Worker 路径已经显式传 context，是正确样板；建议将两处 fallback 全局收敛为"仅进程内惰性默认 + 文档注明"，或者由调用方一律显式传入。

### 3.4 错误信息降级（低）

`evaluate_selection` 把 `MissingExpressionVariablesError` 吞进笼统的 `BoolConversionError`，丢失缺失变量名；Operas/LogLikelihood 路径都保留了这个信息。建议保留原因链上的缺失变量列表（现有 `raise ... from exc` 已有 `__cause__`，但用户可见文案里没有）。

### 3.5 小项

- `ExpressionContext.evaluate(symbols=None)` 用 value dict 的键集作为缓存键的一部分，键集不稳定时缓存会膨胀；调用方基本都显式传 `symbols`，可在 docstring 里写明约定。
- `worker.py` 给 Operas 步骤绑定的 module 标签（`Sample@{uuid} (Operas:{step})`）与 `OperasModule._sample_logger` 的 fallback 标签（`Sample (Operas:{name})`）拼法不一致，日志 grep 时会出现两种前缀。
- `base.infer_project_root` 现在优先读环境变量 task root——行为变化本身合理（Worker spawn 需要），但要在 YAML_REFERENCE 的路径解析一节注明优先级：env > 向上找锚点。

---

## 4. 下一阶段主题一：Calculator 的 V1-YAML 对齐

现状：`Module/calculator.py` + `calculator_spec.py` + `runtime_preparer.py` 已具备 Spec 解析、clone_shadow 安装、`@SampleID/@Sdir/@PackID` token、env_setup 捕获、Portal IO 读写、超时 deadline。**缺的不是执行机制，是 V1 YAML 形态与用户可感知细节。**

### 4.1 YAML 形态差异（对照 `Jarvis-Examples/Eggbox/bin/Example_Bridson_process.yaml`）

| V1 形态 | V2 现状 | 影响 |
|---|---|---|
| `installation`/`initialization`/`execution.commands` 是**字符串列表**（`- "cp -r ${source}/* ${path}"`） | `CalculatorSpec.from_config` 只收 `isinstance(item, Mapping)` 的项，**字符串被静默丢弃** | **V1 卡片装载后安装/执行命令为空**，且无任何报错。这是 Calculator 阶段第一优先级：字符串项应规范化为 `{cmd: <str>, cwd: <execution.path>}`，或在解析期显式报错。 |
| `${source}`/`${path}` YAML 内变量插值 | 无 | V1 卡片的安装命令无法解析；建议在 Spec 解析期做替换（值来自同模块的 `source`/`path` 字段）。 |
| 模块级 `selection`（按参数点跳过模块） | 无 | 工作流语义缺失；`workflow.py` 的执行计划需支持条件步。 |
| `required_modules` 参与 layer 推导 | 已支持（D2.2） | 已对齐。 |
| `Calculators.make_paraller` / 顶层 `path` | 未消费 | 至少要接受并忽略 + 警告，不能报 unknown key。 |
| `modes` 字段（V1 多模式模块） | 无 | 可推迟，但解析期要容忍。 |
| SLHA/xSLHA/File IO 类型 | Portal 侧未暴露（D11.3） | 真实 HEP 卡片（NMSSM 等）依赖 SLHA；无它就谈不上"用户无感"。 |

### 4.2 验收标准（建议）

以 Jarvis-Examples 的 `Eggbox/bin/Example_Bridson_process.yaml`（process 计算器版，而非 Operas 版）**原样跑通**为 Calculator 阶段的 exit criterion：不改用户 YAML 一个字（`EnvReqs.V2` 允许作为新增块出现），DATABASE 与 V1 golden 对齐，sample log 逐条样式对齐（§5.1）。

### 4.3 EnvReqs 归口（用户新决策，2026-07-14）

方向：**工厂、Worker、Redis 等运行设置全部放 YAML 的 `EnvReqs` 下**。现状与张力：

- 已有：`EnvReqs.V2.{workers, batch_size}`，白名单校验，其余键报错。
- 刚做的 Redis 内部化把 broker 定为"不可配置的实现细节"（`INTERNAL_REDIS_CONFIG` 硬编码 `127.0.0.1:6379`）。
- **建议决议**：保留"默认零配置、本地 Redis"体验，同时开 `EnvReqs.V2.redis.{host, port, db}` 作为**可选**覆盖（多机部署、非默认端口场景需要它）；`factory`（watchdog 开关、monitor 频率）与 `worker`（force_serial_layers、sample_artifacts）分组同样收进 `EnvReqs.V2`。白名单从 `{workers, batch_size}` 扩为分组结构，未知键仍硬报错。
- V1 的 `EnvReqs.Python`/`EnvReqs.CERN_ROOT` 依赖检查（V1 `config.py`）是否复刻，留待 project 支持一起决定；解析期先容忍这些键（V1 卡片里存在）。

**注意**：当前 `task_config.py` 对 `EnvReqs.V2` 之外的兄弟键（`Python`、`CERN_ROOT` 等）不校验、不消费——这是对的（V1 卡片能通过）；但 `_runtime_defaults_from_envreqs` 读入的外部 Runtime defaults 若含 `workers/batch_size` 之外的键会硬报错，V1 遗留 defaults 文件会踩雷，需要在实现 EnvReqs 扩展时一并放宽。

---

## 5. 下一阶段主题二：用户体验对齐（"用户体验不出来和 V1 有啥不一样"）

### 5.1 Core 日志系统（缺口属实）

- V1：loguru，双 sink（文件 rotation 5 MB + stdout colorize），格式 `\n·•· <cyan>{module}</cyan>\n\t-> <green>MM-DD HH:mm:ss.SSS</green> - [LEVEL] >>>\n{message}`，hdf5-Writer 用 `Ϡ` 前缀，`raw` extra 直通，启动打 logo banner，`--debug` 控制台阈值切换；另有独立 sampler log 与逐 sample log 文件。
- V2：stdlib logging + QueueListener，格式 `%(asctime)s LEVEL name | message` 加 `key=value` 上下文，colorlog 可选。架构（D0.3 两层、队列化、spawn 安全）**保留**，但**渲染层要换成 V1 视觉格式**：实现一个 V1 样式 Formatter（`·•·`/`Ϡ` 前缀、module 高亮、时间样式、raw 直通），文件与控制台共用；logo banner 在 `init_logger` 后打印；`jarvis_{role}_{pid}.log` 命名改回 V1 的 `<scan>/jarvis.log` 布局。Sample log 侧 `BufferedSampleLogger` 已存在，需要按 V1 的 per-sample 文件布局落盘核对一遍格式。

### 5.2 Flowchart（缺口属实）

- V1：`workflow.export_flowchart_semantics()` 产出 `flowchart.json`（语义图），`jarvisplot.render_flowchart` 渲染 `flowchart.png`；失败降级为跳过 + 警告；`--skip-draw-flowchart` 可关。
- V2：`workflow.py` 只有层推导；golden 测试整体 skip。
- 计划：把 V1 的 semantic export 移植到 V2 `workflow.py`（V2 执行计划已有 layer/step 结构，语义节点还需补 input/output 端口与 selection 元数据），复用 `plot_bridge.py` 调 JarvisPLOT 渲染；默认 run 前绘制、`--skip-draw-flowchart` 兼容，落盘路径与 V1 相同（`<scan>/images/flowchart.{json,png}`）。此项同时是 D11 PLOT 闭环的一部分。

### 5.3 Project 支持（缺口属实）

V1 有 `Jarvis project create/pack/browse/fetch/info`（`project_scaffold.py`、`project_packager.py`、`project_template/`、`official_project_library.py`）。V2 一个都没有。计划：整体移植这四个模块（它们几乎不依赖执行内核，属于低风险搬运），命令挂在 D11.2 的一意图 CLI 下（`Jarvis2 project ...`），`jarvis.project.yaml` 布局与 V1 保持一致。

### 5.4 官方例子接口：Jarvis-Examples 目录化（新需求）

需求：**更新例子库时不动 Jarvis-HEP 代码**，模式参照 Jarvis-Portal 更新 IO format 的方式（entry-point registry，HEP 只认接口）。

现状问题:V1 的 catalog JSON（`jarvishep/card/official_project_library.json`）**打包在 Jarvis-HEP 里**，远程 index URL 也指向 Jarvis-HEP 仓库的 raw 文件——加一个例子必须改 HEP 仓库。

**设计（DESIGN_EXAMPLES_CATALOG_2.0，建议新文档）**：

1. catalog JSON 移到 **Jarvis-Examples 仓库**（如 `catalog/official_project_library.json`），由例子库自己维护：每项含 `name/category/summary/entrypoint/archive_url/archive_root/min_hep_version/compatibility_notes`。
2. HEP 侧只固化三样东西：默认 index URL（指向 Jarvis-Examples raw main）、catalog **schema 版本号**（向后兼容校验）、下载/解包/入口校验逻辑（V1 `official_project_library.py` 已有 90%，含 `JARVIS_OFFICIAL_LIBRARY_INDEX_URL`/`..._TIMEOUT_SEC` env 覆盖，直接搬）。
3. HEP 内不再打包 catalog 副本；离线 fallback 改为"上次成功拉取的本地缓存"（`~/.jarvis/cache/official_catalog.json`）。
4. `Jarvis2 project browse/fetch/info` 消费该接口;catalog 加 `schema_version` 字段，HEP 对未知 major 版本报"请升级 Jarvis-HEP"。

这样加例子 = 在 Jarvis-Examples 提交 tarball + 改 catalog JSON，HEP 零改动，与 Portal 的"升级插件包即得新格式"体验同构。

### 5.5 缺口小结与既有计划的关系

| 用户点名缺口 | 归属 | 说明 |
|---|---|---|
| core 日志系统 | **D12.2**（新） | 渲染层对齐 V1，架构不动 |
| flowchart | **D12.3**（新，与 D11 PLOT 闭环共用） | semantic export + JarvisPLOT 渲染 |
| EnvReqs 归口 | **D12.4**（新） | 扩展 `EnvReqs.V2` 分组白名单，含可选 redis 覆盖 |
| project 支持 | **D12.5**（新） | 移植 V1 四模块 + CLI 子命令 |
| Jarvis-Examples 接口 | **D12.6**（新） | catalog 移仓 + schema 版本化 |
| Calculator V1 对齐 | **D12.1**（新，本阶段主线） | §4；依赖 D11.3 SLHA 暴露 |

D12 各 WP 已登记进 `V2_DISTRIBUTED_PLAN.md` Progress Ledger。执行顺序（绑定）：
**D12.0（依赖 + 表达式扫描小修）→ D12.2（日志格式定型）→ D12.1（Calculator 对齐 + 验收）**，
其后才是 D12.3 flowchart / D12.4 EnvReqs 扩展 / D12.5–D12.6 project+examples（可与 D11.2 CLI 穿插）。
D12.2 必须先于 D12.1 **验收**：sample-log golden 依赖日志格式；日志不定型 golden 会返工。
D12.1 实现可与 D12.2 部分重叠，但 **Eggbox process 卡片 golden 对齐** 以 D12.2 落地为前提。
Eggbox process 用 JSON IO，可先验收；声明"Calculator 完整对齐"还要等 **D11.3**（SLHA/xSLHA）。

---

## 6. 本次验证记录

- `python3 -m pytest -q` @ `0a5e85e`：**283 passed, 1 skipped, 38 subtests passed**，~4 min 45 s。唯一 skip 仍是 flowchart export 占位（`tests/test_workflow_execution_plan.py`），与 §5.2 结论一致。
- 静态审阅范围：33 个改动文件全量 diff + 3 个新模块全文 + Module/calculator 全文。
- 对照材料：V1 `core.py`（init_logger）、`client.py`（CLI 帮助）、`Module/calculator.py`、`official_project_library.py`、`Jarvis-Examples/Eggbox` 卡片。

---

## 7. 给执行 Agent 的行动清单（浓缩）

1. **提交前**：修 §3.1（表达式扫描收窄到表达式字段）；§3.2 至少加 TODO + 测试标注已知限制。
2. **D12.0**：Operas 升核心依赖 + 表达式扫描仅扫 expression 字段（不扫 calculator `cmd`/paths）。
3. **D12.2 日志**：V1 样式 Formatter + logo banner + 文件布局；保留 QueueListener 架构。**先定型**。
4. **D12.1 Calculator（本阶段主线）**：先修 `CalculatorSpec` 字符串命令静默丢弃（§4.1 第一行），再做
   `${source}/${path}` 插值、模块级 `selection`、`make_paraller`/`modes`/顶层 `path` 容忍；
   验收 = `Jarvis-Examples/Eggbox/bin/Example_Bridson_process.yaml` **一字不改**跑通 + DATABASE/sample-log golden 对齐。
5. **D12.3–D12.6**：按 §5 各节设计执行；D12.6 需要在 Jarvis-Examples 仓库先落 catalog JSON（跨仓库改动，需用户确认后再动 Jarvis-Examples）。
