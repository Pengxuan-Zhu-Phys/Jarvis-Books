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
