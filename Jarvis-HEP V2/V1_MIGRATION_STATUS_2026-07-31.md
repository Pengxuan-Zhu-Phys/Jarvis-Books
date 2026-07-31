# V1 → V2 迁移状态与开发进度（2026-07-31）

**基线**：`jarvishep2` 分支 `jarvis2` @ `6e308a4`，工作树干净
**方法**：全部数字由脚本对**运行中的代码**实测（Distributor 注册表、CLI、schema、校验器），
不取自文档——`components/V1_TO_V2_MAP.md`（7 月 11 日）早于 D13/D17，已过时。

---

## 1. 开发进度

| 里程碑 | 状态 |
|---|---|
| D0–D7 运行时内核（Redis、Worker、Archiver、监控、断点续跑、验收） | **关闭**，已归档 |
| D9–D12 架构加固 + Calculator/UX 对齐 + 项目工具 + 例子库 | **关闭** |
| D13 采样器（FeedbackSampler、MCMC 家族、nuisance、Dynesty/MultiNest、诊断） | **关闭**（D13.1–D13.11） |
| D16.1 skills 库 v1（9 篇） | **完成** |
| D17 严格卡片校验（词汇表、zone、数值、错误 UX、ASCII-only） | **关闭**（D17.1–D17.8） |

**尚未开工**：

- **D8 Agent Bridge** — 维护者明确冷冻，勿动
- **D14 集群执行**（4 个 WP）— 远程 Worker 池、SLURM、broker 认证/持久化
- **D15 结果复用与分析**（7 个 WP）— warm-start 缓存、`Jarvis2 analyze`、物理级绘图场景
- **D16.2–16.4** — skills 进包 + CLI + 卡片 CI
- **零散**：D13.12（`run_adaptive` 4068 行重构）、D13.15（8 个既有失败测试待分类）、
  D17.8（本轮发现的两个体验回退）

---

## 2. V1 功能迁移清单（实测）

### 2.1 采样器：25 个中 13 个有 V2 实现

V2 的 Distributor 注册了 17 个方法名，但其中有别名（`AM`/`AMMCMC`、`PT`/`PTMCMC`/`PTEnsemble`、
`Ensemble`/`EnsembleMCMC`）。按**能力**去重后分三档：

| 档位 | 方法 | 说明 |
|---|---|---|
| **生产可用**（真实扫描验证过） | `Bridson` `Random` `Grid` `CSV` `AdaptiveBridson` `Dynesty` `MultiNest` | Eggbox / iDM 等示例在跑 |
| **已注册但未验收** | `MCMC` `AM` `AMMCMC` `DRAM` `EnsembleMCMC` `Ensemble` `DEMCMC` `PT` `PTMCMC` `PTEnsemble` | 代码在、校验通过、**用户现在就能启动**，但维护者判定"尚未迁移完成"（2026-07-31）。⚠ 见 §4 |
| **完全没有** | 见下 12 个 | V1 有文件，V2 无任何实现 |

**未迁移的 12 个 V1 采样器**：

| V1 模块 | 类别 |
|---|---|
| `mala` `hmc` `nuts` | 梯度类（Langevin / 哈密顿 / NUTS） |
| `slicemcmc` `ess` | 切片采样 / 椭圆切片 |
| `robustam` | 稳健自适应 Metropolis |
| `dream` `dream_lite` | DREAM 家族 |
| `diver` | 差分进化 |
| `dnn` `rltpmcmc` | 神经网络代理 / RL 调温 |
| `nuisance_sampler` | nuisance 参数采样器（注意：`nuisance_optimize` **已**迁移为 `Module/nuisance.py` + `profile1d.py`） |

这与 `DESIGN_SAMPLERS_2.0.md` 的 Non-goals 一致（RL/DNN/差分进化明确列为later milestone），
不是遗漏。

### 2.2 CLI：3 个 V1 功能没有对应实现

| V1 | 作用 | V2 |
|---|---|---|
| `--benchmark [秒数]` | 吞吐基准模式 | ❌ 无 |
| `--refs` | 打印**引用信息**（内置采样器的参考文献） | ❌ 无 |
| `--skip-library-installation` | 跳过库安装 | ❌ 无 |

`--refs` 值得单独说：物理学家写论文要引用 dynesty / emcee 这类底层采样器，V1 提供了一条命令
直接输出。V2 没有，用户得自己去翻文献。**这是唯一一个面向最终用户产出的缺口**，其余两个是
运维/开发向。

其余 V1 CLI 功能均已迁移（`--plot` `--convert` `--monitor` `--resume` `--check-modules`
`--skip-draw-flowchart` `-v` `project` `portal`），且 V2 另有 V1 没有的
`validate` / `ps` / `kill` / `operas`。

### 2.3 模块层：已实质覆盖

V1 顶层模块在 V2 中多为改名重构而非缺失：`hdf5writer`→`database`、`plot`→`plot_scene`+
`plot_bridge`、`monitor`→`dashboard`+`monitoring/`、`io_manager`/`observable_io`→`io_portal`、
`moduleManager`/`modulePool`→`worker`+`calculator_pools`、`config`→`task_config`+
`task_validation`、`dataconvert`→`database`+`Jarvis2 convert`。

`Module/` 下：`calculator` `operas` `likelihood` 已迁移；
`nuisance_LogLikelihood`/`nuisance_passCondition` 已合并为 `Module/nuisance.py` + `profile1d.py`。

---

## 3. 本轮新发现：顶层块名打错会被静默吞掉（建议立 WP）

D17 严格校验对**块内部**是封闭的，但**根层**被标成了 `x-jarvis-zone: delegated`
（`additionalProperties: true`）。实测后果：

```
Calculators 拼成 Calculater  ->  放行 ❗ 整个计算器流水线被静默忽略
Operas      拼成 Opera       ->  放行 ❗ 整个 Operas 块被静默忽略
EnvReqs     拼成 EnvReq      ->  放行 ❗ workers 设置失效，按默认值跑
Scan        拼成 Scam        ->  放行 ❗ 输出落到 outputs/default/
```

`Calculater:` 那一行的实际后果是：扫描正常启动、正常结束、产出一整批**没有跑过任何物理程序**
的样本。这正是 D17 立项要消灭的那类"看起来一切正常、结果悄悄不对"的失败。

**为什么会这样**：根层标 `delegated` 很可能是为了不误伤 `LibDeps` 等 V1 顶层块
（已核实：7 份示例卡片在用 `LibDeps`，`command_parser.py:305` 也确实读它，而根 schema 的
`properties` 里没有它）。但按 `DESIGN_STRICT_VALIDATION_2.0.md` §2 的定义，`delegated` 指
**外部独立版本的包拥有该词汇表**——而任务卡的顶层块（Scan/Sampling/Calculators/Operas/
EnvReqs/LibDeps）**全部由 HEP 自己拥有**，没有任何外部组件参与。所以这是 zone 的**误分类**：
正确做法是把 `LibDeps` 等 V1 顶层块**声明进 schema**，然后把根层改回 `closed`。

（`Sampling` 打错反而会被抓到，因为老的 contracts 层要求必须有 `Method`——属于运气，不是设计。）

---

## 4. 两个需要维护者决定的问题

1. **"已注册但未验收"的 10 个 MCMC 家族方法**：代码在、`Jarvis2 validate` 通过、用户现在就能
   `Method: DRAM` 直接开跑。如果这个家族确实还不该给用户用，建议在注册层标成 experimental
   （校验时给 warning，或从 `JV2-MTH-003` 的 "Available:" 列表里摘掉），
   而不是让"校验通过"暗示它已就绪。（此问题在 `SCHEMA_REVIEW_2026-07-31.md` §1 已提过，仍未决。）

2. **`--refs` 要不要补**：这是唯一一个直接影响最终用户产出（论文引用）的 V1 缺口，
   实现成本很低（一份静态引用表 + 一条命令）。

---

## 5. 建议的下一步顺序

1. **根层 zone 误分类**（§3）——十几行的改动，堵住 D17 目前最大的漏网之鱼
2. **D17.8**（注入键幻影报错 + 编码错误压制 schema 错误）——同属校验体验，一起做省一次上下文
3. **D14.1 集群执行**——真正的新能力，也是 plan 里 next pick
4. D13.15（8 个失败测试）建议在开 D14 之前清掉，否则新工作没有干净的基线

---

## 6. 第二轮排查（2026-08-01）：按"用户能写进 YAML 的东西"逐项核对

第一轮是**模块名层面**的对比，会漏掉"模块在、功能是空的"这类——LibDeps 就是这么被漏掉的
（`library.py` 存在，所以看起来已覆盖）。这一轮改为核对用户真正会写的 YAML 键。

### 6.1 `Utils.interpolations_1D` — 能力已迁移，但**迁移路径没告诉用户**

**15/65 张真实卡片**用了这个块。V1 的语义是：从 CSV（如 `Xenon1T2019SD_p.csv`，
氙实验自旋相关质子截面上限）或内联 x/y 数组构造一维插值，支持 `logX`/`logY`/`kind`，
然后把它**注册进表达式命名空间**（`core.py:351` → `self._funcs`，和 Operas 函数同一个字典），
于是 `XenonSD2019(mchi)` 可以直接写在 LogLikelihood 或 selection 里。对暗物质唯象来说，
"把发表的排除曲线当函数用"是日常操作。

V2 当前的回应是一句硬拒绝（`task_schema.py:28`）：

```
"Utils": "Remove top-level Utils: it is not supported by V2."
```

**但能力其实在的**——已经迁进 Jarvis-Operas，而且形态更好：

- `interp1.interp1_xy_flat` / `interp1_xlog_yflat` / `interp1_xflat_ylog` / `interp1_xy_log`
  —— 正好覆盖 V1 的 `logX`/`logY` 四种组合
- `dmddxe.*` —— 一整套已注册的直接探测实验限制（CDEX、CDMS、DAMIC、DarkSide50、
  `AllLimitsHM2024`/`AllLimitsLM2024` …），用户不必再自己找 CSV
- `add_interpolation_namespace(manifest_path, namespace, namespace_manifest, …)`
  —— 用户仍可注册**自己的**曲线数据

所以这不是功能缺失，是**迁移路径缺失**：用户被告知"删掉这个块"，却没被告知能力去了哪里。
一个手上有 15 张这种卡片的用户，看到这条消息只会以为 V2 砍掉了这个功能。

**建议**（WP **D18.6**）：把这条消息改成指路 —— *"`Utils.interpolations_1D` 已迁移到
Jarvis-Operas：内置实验限制见 `Jarvis2 operas list | grep dmddxe`；自有曲线用
`interp1.*` 或 `add_interpolation_namespace` 注册"*，并在 YAML_REFERENCE + 一篇 skill
里写清对照写法。

### 6.2 变量分布：10 种里缺 5 种

| | 类型 |
|---|---|
| V2 支持 | `Flat` `Log` `Normal` `Log-Normal` `Logit` |
| **V2 缺** | `Binomial` `Poisson` `Beta` `Exponential` `Gamma` |

V2 的 `variables.py` 用显式白名单硬拒绝（`JV2-VAR-002`），所以 V1 卡片里写 `Poisson`
会直接失败。**缓解因素**：65 张真实卡片里 **0 张**用到这 5 种，所以这是潜在的 V1 卡片
兼容缺口，不影响任何现有工作流。V1 的实现是 `variables.py` 里几行 numpy 调用，
补齐成本很低（WP **D18.7**）。

### 6.3 澄清：`modes` 不是缺口（V1 死代码）

`Calculators.Modules[].modes` 在 V2 schema 里被容忍但零代码消费——看着像"解析了没实现"。
核实后：**V1 自己也只在 `Module/calculator.py:18-19` 设了个布尔标志，之后从不使用**，
且 0 张真实卡片用它。记录在此，免得将来有人把它当缺口去"补齐"。

### 6.4 已确认覆盖（本轮核实，无需再查）

- **模块级 `selection`**（V1 按参数点跳过某个 calculator）——已实现
  （`Module/calculator.py:549`，D12.1 交付）
- `Scan` 子键（`name` / `save_dir` / `sample_directory`）——真实卡片用到的全部已声明
- `required_modules` / `make_paraller` / `clone_shadow` / `Sampling.Nuisance`——均有代码消费
- `Module/parameters.py`——V1 内部类，无对应 YAML 块，对应 V2 的 `Sampling.Variables`

### 6.5 更新后的缺口总表

| 缺口 | 严重度 | WP |
|---|---|---|
| LibDeps 装一次（完全没有安装引擎 + 2 个高频 token KeyError） | 高 | D18.1–18.5 |
| `Utils.interpolations_1D` 迁移路径无指引（15 张卡片） | **中高**（能力在，是指引缺失） | D18.6 |
| 5 种变量分布被硬拒 | 中（0 张卡片在用） | D18.7 |
| 12 个 V1 采样器无实现 | 中（多数是 `DESIGN_SAMPLERS_2.0.md` 明列的 non-goal） | 待定 |
| `--refs`（引用信息，写论文用） | 中低 | 待定 |
| `--benchmark` / `--skip-library-installation` | 低（后者随 D18.4 一起） | D18.4 |
