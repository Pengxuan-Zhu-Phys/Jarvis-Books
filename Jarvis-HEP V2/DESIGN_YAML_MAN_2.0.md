# DESIGN — `Jarvis man`：面向 human / coding agent 的 YAML 写法手册中心

**状态**：设计已定稿（§15 五项取舍 2026-08-08 拍板）；WP 已登记进 `V2_DISTRIBUTED_PLAN.md`（D24.1–D24.11）。最小可用切片已落地（2026-08-09）：`Jarvis man` CLI + sampler/calculator/yaml 路径 + distribution 收紧 + 档 C `unstable`。
**日期**：2026-08-08（实现起步 2026-08-09）
**分支**：`jarvis2`
**测量基线**：工作树 `707bde6` + 未提交的 D23.8–D23.12 修复
**目标读者**：实现该 WP 的 coding agent；以及需要判断取舍的维护者
**配套文档**：[`YAML_REFERENCE_2.0.md`](YAML_REFERENCE_2.0.md)（现有散文全集，1650 行）、
[`DESIGN_YAML_VALIDATION_2.0.md`](DESIGN_YAML_VALIDATION_2.0.md)（校验分层）、
[`DESIGN_STRICT_VALIDATION_2.0.md`](DESIGN_STRICT_VALIDATION_2.0.md)、
`Jarvis-HEP-v2/docs/task-card-schema.md`、`Jarvis-HEP-v2/docs/validation-diagnostics.md`

> **本设计文档用中文书写；`Jarvis man` 的运行时输出永远是英文**（§15 决议 2/5，写成不变量 K6）。

---

## 0. 结论摘要

**缺的不是 schema，是把 schema 端出来的那个 CLI 出口。** 但实测下来，只做"出口"是不够的：
把手册接上去的那一刻，会立刻暴露**校验器本身的空洞**——17 个采样方法里 12 个的
`Sampling.Bounds` 在 schema 里**一个键都没声明**，实测 `Jarvis validate` 对
`adapt_windwo: 25` 和 `totally_bogus_key: 7` **零诊断、全绿放行**（§1.5）。
所以本设计的核心不是"写手册"，而是一条硬不变量：

> **K1：`Jarvis man` 只渲染校验器认识的东西。man 能打印的键，`validate` 必须能拒绝它的拼写错误。**

这条不变量把"补文档"变成"补校验"，两件事一次做完，且互为验收：手册有内容 ⇔ 校验有牙齿。
反过来说，任何**只加散文、不加校验**的方案（把 `YAML_REFERENCE_2.0.md` 塞进包里当 man）
都会重演已经发生过三次的漂移（D22.1、D22.3、§7.2 标题，见 §1.6）。

CLI 表面（§3）——`Jarvis man` 是**总的 manual 中心**，回答的是同一个问题：
**"这段 YAML 该怎么写"**：

```
Jarvis man                            # 中心页：有哪些 manual 域
Jarvis man yaml                       # 卡片总览：六个顶层块 + zone + 下一跳
Jarvis man yaml Sampling.Bounds       # 任意 YAML 路径的字段手册
Jarvis man sampler Bridson            # 按方法：Bounds 旋钮 / 默认 / 别名 / 能力
Jarvis man calculator execution       # 按主题：module | execution | modes | pools
Jarvis man operas                     # Operas.Modules[] 该怎么写
Jarvis man tokens                     # &J / @Sdir / @PackID / ${LibDeps:} / ${Scan:}
Jarvis man example bridson            # 吐一张能直接跑的最小卡
Jarvis man --code JV2-MTH-012         # 从诊断码跳回手册（agent 闭环）
… --json                              # 全部子命令的机器可读形态
```

内容来源分四层（§4），**不新增第五个真相源**：L0 结构来自现有 schema catalog（已经是活校验器），
L1 语义词表来自 `contracts/*` 里已有的 `frozenset`/`dict`（提升为可导出的声明），
L2 能力来自 `Distributor` 注册表，L3 散文以**英文 `description` 内联写进 schema JSON**并被 lint 强制。

**`Jarvis man` 与 `Jarvis portal man` / `Jarvis operas info` 是两类文档**（§4.4）：
前者教你**怎么写这段 YAML**，后者解释**那个模块干什么**。man **不把 YAML 写法转交出去**——
实测 14 个 Portal 格式的 YAML 字段 schema（59 个属性、14/14 带 `x-jarvis-example`）
**本来就在 HEP2 自己的 catalog 里**，man 直接渲染即可（§6.2）。

重点两块按要求单独展开：**sampler**（§5）、**calculator**（§6，含 Calculator → Portal-IO → Operas 连接链）。

---

## 1. 现状实测

以下每一条都跑过，不是读 diff 得来的。

### 1.1 已有的三个入口，各自的边界

| 命令 | 覆盖什么 | 不覆盖什么 |
|---|---|---|
| `Jarvis portal man [format]` | Portal-IO 的 8 个格式名（input 7 + output 7 = 14 个方向槽）；每种给 **描述 → observables → YAML 卡片 → 文件切片（执行前/后）→ 注意事项** | 只讲一个 IO 条目内部，不讲它挂在卡片哪里、和 Sampling/Operas 怎么接 |
| `Jarvis operas info NAME` | 单个算子的 `NAME/CATEGORY/MODULE/DESCRIPTION/SIGNATURE/ASYNC` + `--json` | 只讲算子本身，不讲 `Operas.Modules[]` 这个 YAML 块怎么写 |
| `Jarvis validate TASK.yaml` | 已有卡片的诊断（~100 个 `JV2-*` 码），`--json` 带 `suggestion`/`example` | **要求你先有一张卡**；不能回答"我该写什么" |

`Jarvis portal man csv` 的**版式**实测非常好用（描述 / observables / YAML / 文件切片 / notes 五段），
**`Jarvis man` 应当复用这套版式语汇**，而不是另发明一套（但内容归属不同，见 §4.4）。

### 1.2 `jarvishep2/schema/` 不是死文档，它就是活的校验器

一开始我以为 `schema/*.json` 只是给编辑器用的发布产物。实测否定：
[`task_schema.py:68-97`](../../Jarvis-HEP-v2/jarvishep2/task_schema.py:68) 用
`manifest.json` 装配出 `Draft202012Validator`，`Jarvis validate` 的 `JV2-SCH-*` 全部由它产生。

- `manifest.json` 是唯一索引：`schema_files`（39 个文件）、`sampling_methods`（17 个方法 → schema URI）、
  `io.input/output`（14 个方向槽 → schema URI）。**校验器只加载它列出的文件，从不联网取远程 schema。**
- 每个 object 必须带 `x-jarvis-zone`（`closed`/`delegated`/`open`），
  [`schema_catalog_lint_errors()`](../../Jarvis-HEP-v2/jarvishep2/task_schema.py:35) 把缺失当成 CI 失败。
  **这是"作者义务由 lint 强制"的现成先例**，§4.3 的手册作者义务照抄这个模式即可。
- `x-jarvis-example` 已经被诊断渲染器消费（`docs/validation-diagnostics.md`「Sources of guidance」）。

**结论：单一真相源已经存在，且已经是可执行的。** 手册应当是它的一个渲染器，不是它的副本。

### 1.3 但 schema 里几乎没有散文

全量统计（39 个 schema 文件，324 个 schema 节点）：

| 注解 | 出现次数 | 覆盖率 |
|---|---:|---:|
| `x-jarvis-zone` | 68 | lint 强制，达标 |
| `x-jarvis-example` | 21 | 6.5% |
| `description` | 6 | 1.9% |
| `title` | 1 | 0.3% |

也就是说：**schema 能告诉你"有哪些键、什么类型、必填与否"，但基本不能告诉你"这个键是干嘛的"。**
直接拿 schema 生成手册，出来的是一张字段表，不是 man page。散文必须被作者写进来（§4.3）。

### 1.4 17 个采样方法里，12 个的 `Bounds` 在 schema 里是空的

逐文件测量 `manifest.json → sampling_methods` 的 17 个目标：

| 方法 | 文件大小 | `x-jarvis-zone` | `x-jarvis-example` | 声明了 `Bounds` | Bounds 封闭 |
|---|---:|:--:|:--:|:--:|:--:|
| Bridson | 1493 B | ✅ | ✅ | ✅ 5 键 | ✅ |
| Random | 1407 B | ✅ | ✅ | ✅ 4 键 | ✅ |
| Grid | 964 B | ✅ | ✅ | ✅ 2 键 | ✅ |
| CSV | 451 B | ✅ | ✅ | ✅（`$ref` common#/csv，5 键） | ✅ |
| AdaptiveBridson | 1055 B | ✅ | ✅ | ✅（`$ref` common#/adaptiveBridson，**30 键**） | ✅ |
| MCMC / AMMCMC / AM / DRAM | 252–260 B | ❌ | ❌ | ❌ | — |
| EnsembleMCMC / Ensemble / DEMCMC | 260–273 B | ❌ | ❌ | ❌ | — |
| PTMCMC / PT / PTEnsemble | 252–268 B | ❌ | ❌ | ❌ | — |
| **Dynesty / MultiNest** | 262–266 B | ❌ | ❌ | ❌ | — |

那 12 个文件的全部内容就是「`Method` 必须等于这个常量」+ `$ref variables.json`。252 字节，一个旋钮都没有。

**注意 Dynesty / MultiNest 与 MCMC 家族不同**：前两者的旋钮词表**已经存在于代码里且校验已生效**
（`JV2-BND-*`，见 §5.2 档 B），只是没搬进 schema；后十者是真空洞（§1.5）。

### 1.5 后果实测：MCMC 卡片的 Bounds 拼写错误，`validate` 全绿放行

```yaml
Sampling:
  Method: "AMMCMC"
  Bounds:
    num_chains: 4
    num_iterations: 100
    adapt_windwo: 25          # 拼错：应为 adapt_window
    totally_bogus_key: 7      # 纯垃圾键
```

```
$ Jarvis validate mcmc_typo.yaml --json
issues: 0
```

**零诊断。** 原因链清楚：
[`core/sampling.json`](../../Jarvis-HEP-v2/jarvishep2/schema/core/sampling.json) 把 `Bounds` 标成
`x-jarvis-zone: delegated` + `additionalProperties: true`；`ammcmc.json` 不再收窄；
[`contracts/methods.py:291-293`](../../Jarvis-HEP-v2/jarvishep2/contracts/methods.py:291) 对
`_MCMC_METHODS` 只有一个空分支（注释写着 "Already reported above"，实际什么都没报）。

> **本 WP 不修这个洞**（§15 决议 1：MCMC 家族尚未添加完成，先略过）。
> 处理方式见 §5.3：man 对这 10 个方法**显式标注 surface 未定案**，不装作有手册。

### 1.6 唯一完整的散文在另一个仓，不随包发布，且按 WP 手工追平

`YAML_REFERENCE_2.0.md`：1650 行，13 章 + 3 附录，内容质量高。但：

1. 它在 **Jarvis-Books** 仓，不在 `Jarvis-HEP-v2`，**不进 wheel**。装了包的用户看不到。
2. 维护方式是手工追平：`git log` 显示 `a7783a0 docs(d22): document flat Sampling.Mapper` →
   `125b0a9 docs(d22): document Sampling.Mapper as {name, expression} list`，**同一个特性改了两次**。
3. 漂移已发生三次并被逐条抓出：D22.1（docs↔code 不一致）、D22.3（台账仍写"扁平 map"）、
   §7.2 的**小节标题**至今仍写着 "flat name → expression"（正文已对）。
4. D23 刚落地的 Operas 常量：`pdg.` 在该文档出现 4 次，`operas_constants` 出现 **0 次**。

**这不是作者不认真——是"手工散文 + 独立仓库"这个结构必然产生的滞后。**

### 1.7 诊断里的 hint 指向 wheel 用户打不开的路径

实测 `Jarvis validate --json` 的每条 `JV2-SCH-*`：

```json
"hint": "See docs/task-card-schema.md for the strict card interface."
```

`docs/task-card-schema.md` 是**仓库相对路径**。`pip install` 的用户没有这个目录。
这正好是 `Jarvis man` 应当填的位置：hint 应当变成一条**可以直接跑的命令**（§9.2）。

顺带测到一个路径前缀不一致（影响 §3.2 的路径语法）：
schema 类诊断的 `path` 是 `$.Sampling.Bounds`（带 `$.`），
contract 类诊断的 `path` 是 `Sampling.Bounds.MaxAttempt`（不带）。同一份 `--json` 里两种并存。

### 1.8 六张自带示例卡今天就全绿——example 语料是现成的

```
Sampling_Dynesty_Full.yaml         errors=0 warnings=0
Sampling_Dynesty_Simple.yaml       errors=0 warnings=0
Sampling_MultiNest_Full.yaml       errors=0 warnings=0
Sampling_MultiNest_Simple.yaml     errors=0 warnings=0
quickstart_bridson_operas.yaml     errors=0 warnings=0
quickstart_csv_operas.yaml         errors=0 warnings=0
```

它们在 `jarvishep2/project_template/`，**已经进 wheel**（package-data 含 `project_template/**/*`），
已经被 `Jarvis project create` 用作脚手架。
`Jarvis man example` 应当**直接吐这些文件**，不要另写一套示例文本——另写就是又一个漂移源。

### 1.9 Portal 的 YAML 字段 schema 已经在 HEP2 自己的 catalog 里

这是决定 §4.4 边界的关键测量：

| 文件 | 声明属性数 | `x-jarvis-example` |
|---|---:|:--:|
| `io/input/{csv,dat,slha,tsv,wolfram}.json` | 4 each | ✅ |
| `io/input/{json,text}.json` | 5 each | ✅ |
| `io/output/{csv,dat,slha,tsv,wolfram,xslha}.json` | 4 each | ✅ |
| `io/output/json.json` | 5 | ✅ |
| **合计** | **59** | **14 / 14** |

也就是说：**"`execution.input[]` 这条怎么写"这个问题，HEP2 已经有完整答案在手**，
由 [`_validate_selected_io()`](../../Jarvis-HEP-v2/jarvishep2/task_schema.py:297) 用于校验。
man 直接渲染它即可，不需要把用户支到别的命令去（§4.4）。

---

## 2. 目标与非目标

### 2.1 目标

- **G1**：任何人（含 coding agent）在**没有卡片、没有网、没有仓库**的情况下，
  用 `Jarvis man …` 问出"这段 YAML 该写什么"，并**在同一页里得到完整答案**。
- **G2**：手册内容与校验器**同源**，不可能各自漂移（K1）。
- **G3**：coding agent 能拿到稳定的 `--json`，并能从诊断码**单跳**回到手册（§9）。
- **G4**：重点覆盖 **sampler** 与 **calculator**，且把
  Calculator → Portal-IO → Operas 的连接方式讲清楚（§6.4）。
- **G5**：`Jarvis man example` 输出的每张卡都能 `Jarvis validate` 通过（CI 门禁）。

### 2.2 非目标

- **不**把 `YAML_REFERENCE_2.0.md` 整体搬进包里（§1.6：搬进去只是把漂移换个位置）。
  它继续作为**评审用的 as-built 全集**存在；man 成为**用户用的运行时真相**（K7）。
- **不**做 MCMC 家族的 Bounds 声明化与收紧（§15 决议 1）。
- **不**做交互式向导 / TUI。
- **不**做 YAML 自动补全的 LSP（schema 已能喂给编辑器，正交）。
- **不**改任何采样器的数值行为。
- **不**碰 V1（Jarvis-HEP 是冻结产品）。

---

## 3. CLI 表面

### 3.1 `Jarvis man` 是 manual 中心

`Jarvis man` 无参 → 中心页，列出所有 manual 域及一行摘要。
所有域回答的是**同一类问题：这段 YAML 怎么写**。

| 命令 | 内容 |
|---|---|
| `Jarvis man yaml` | 卡片总览：`Scan` / `Sampling` / `Calculators` / `Operas` / `EnvReqs` / `LibDeps` 六个顶层块、各自 zone、一行摘要 |
| `Jarvis man yaml <PATH>` | 任意 YAML 路径的字段手册 |
| `Jarvis man sampler [<METHOD>]` | 无参 → 方法对照表；有参 → 单方法手册（§5） |
| `Jarvis man calculator [<TOPIC>]` | 无参 → 主题清单 + 连接链；有参 → `module`/`execution`/`modes`/`pools`（§6） |
| `Jarvis man operas` | `Operas.Modules[]` 块怎么写（§6.3） |
| `Jarvis man tokens` | `&J` / `@Sdir` / `@PackID` / `@SampleID` / `${LibDeps:}` / `${Scan:}` / `@{ROOT path}`（§7） |
| `Jarvis man example [<NAME>]` | 可直接跑的最小卡（§8） |
| `Jarvis man --code <JV2-XXX-NNN>` | 诊断码 → 所属块的手册（§9.2） |

全部支持 `--json`。

**语法细节（实现时的一个判断，非疑问）**：`yaml` 前缀对其他域**可选且等价**——
`Jarvis man yaml calculator execution` == `Jarvis man calculator execution`。
理由：保留最初提案的书写习惯，同时让中心页保持扁平可 grep。
域名（`yaml`/`sampler`/`calculator`/`operas`/`tokens`/`example`）与 YAML 路径不会撞名
（路径以大写块名开头或带 `$.`），先按域名匹配，否则按路径解析。

**不属于 `Jarvis man` 的**：`Jarvis portal man` 与 `Jarvis operas info` 保持原样、原位置。
它们解释的是**模块行为**，不是 YAML 写法（§4.4）。中心页可以提一句"想了解模块本身看那两条"，
但 man 自己的页面必须**自足**。

**命名核对**：`_PACK_MANIFEST_FLAG = "--man"`（`client.py:61`）是 `project pack` 的旗标，
与新增的 `man` 子命令不冲突（实测无同名子命令）。

### 3.2 路径语法

必须**同时**接受 §1.7 实测到的两种前缀，以及数组的两种写法：

| 输入 | 归一化为 |
|---|---|
| `Sampling.Bounds` | `$.Sampling.Bounds` |
| `$.Sampling.Bounds` | 同上 |
| `$.Calculators.Modules[0].execution.input[0]` | `$.Calculators.Modules[].execution.input[]`（下标抹平为"该数组的元素"） |
| `Calculators.Modules[].execution` | 同上 |
| `sampling.bounds` | 大小写不敏感匹配 → 命中后打印**规范拼写**并提示 |

未命中时给 `difflib` 建议——与 `task_schema.py:208` 现有的 did-you-mean 行为一致，用户体验统一。

### 3.3 人读输出的版式

照抄 `Jarvis portal man <format>` 的分段风格，字段名对齐卡片语境。**输出永远是英文**（K6）：

```
╭─ Sampling.Bounds · Bridson ──────────────────────────────╮
│ Method-specific sampler knobs. Closed: unknown keys are  │
│ rejected (JV2-SCH-001).                                  │
╰──────────────────────────────────────────────────────────╯
╭─ Keys ───────────────────────────────────────────────────╮
│ Radius        number >0     required   Poisson-disk …    │
│ MaxAttempt    integer ≥1    required   k in Bridson's …  │
│ MaxWorker     integer       optional   —                 │
│ Seed / seed   integer       optional   default 0         │
╰──────────────────────────────────────────────────────────╯
╭─ YAML ───────────────────────────────────────────────────╮
│ Bounds:                                                  │
│   Radius: 0.1                                            │
│   MaxAttempt: 30                                         │
╰──────────────────────────────────────────────────────────╯
╭─ Diagnostics that fire here ─────────────────────────────╮
│ JV2-MTH-010  Radius / MaxAttempt missing                 │
│ JV2-MTH-011  Radius not a positive number                │
│ JV2-MTH-013  MaxAttempt not an integer ≥ 1               │
╰──────────────────────────────────────────────────────────╯
╭─ See also ───────────────────────────────────────────────╮
│ Jarvis man yaml Sampling.Variables                       │
│ Jarvis man example bridson                               │
╰──────────────────────────────────────────────────────────╯
```

「Diagnostics that fire here」是**新东西**，也是 agent 最需要的一段：它把
"这个块" 和 "会报什么码" 绑在一起，反向就是 `--code` 查询（§9.2）。

### 3.4 与 D23 的 ASCII 约束

`task_validation._ascii_diagnostic` 会把非 ASCII 转义成 `\uXXXX`（D23.14 记录的既有缺陷）。
man 输出虽是英文，但会用 `≥` `→` `—` 等排版字符，**渲染路径不得复用那条转义**。
实现时 man 走独立的 Rich 渲染，不经过 `format_line`。
若 D23.14 先修好，这条自动满足，但**不要依赖它**。

---

## 4. 内容来源分层（不新增真相源）

```
L3  prose        schema JSON 内联的英文 description / x-jarvis-example  ──► lint 强制
L2  capability   Distributor 注册表：stateless / resume
L1  semantics    contracts/* 的词表（提升为可导出声明）
L0  structure    schema catalog（manifest → 39 个 schema 文件）── 已经是活校验器
```

### 4.1 L0 结构：直接读校验器用的那套

复用 `task_schema._schema_catalog()`（已有 `lru_cache`）。man 渲染器解析出：
键名、`type`、`required`、`enum`/`const`、`additionalProperties`、`x-jarvis-zone`、
数值边界（`minimum` 等）、`$ref` 展开。

**零新增数据。** 这一层今天就能跑，且天然不会与 `validate` 不一致。

### 4.2 L1 语义：把 contracts 里的私有词表提升为声明

现在这些词表是模块私有常量，man 拿不到（也不该 `import` 私有名）：

| 词表 | 位置 | 内容 | schema 里有吗 |
|---|---|---|---|
| `_PARAMS_ALLOWED` / `_PARAMS_REQUIRED` | [`contracts/variables.py:28-52`](../../Jarvis-HEP-v2/jarvishep2/contracts/variables.py:28) | 10 种分布各自的参数名（Flat: min/max；Normal: mean/stddev；…）+ 通用 `length`/`num` | ❌ `distribution.parameters` 是 `delegated`/`additionalProperties: true` |
| `NESTED_CONSTRUCTOR_USER_KEYS` | [`Sampling/dynesty_sampler.py:49-75`](../../Jarvis-HEP-v2/jarvishep2/Sampling/dynesty_sampler.py:49) | 23 个 dynesty 构造器键 | ❌ |
| `_BOUNDS_META_ALLOWED` | [`contracts/nested.py:18-37`](../../Jarvis-HEP-v2/jarvishep2/contracts/nested.py:18) | `nlive`/`n_live`/`rseed`/`seed`/`Seed`/`dlogz`/`dlogz_init`/`run_nested`/`sampler`/`constructor` | ❌ |

做法：凡是能塞进 JSON Schema 的（键名、类型、默认、下界）就塞进 schema，让 L0 直接吃到（§5.2 档 B）；
塞不进的（跨字段关系、别名归一）留在 contracts 但**导出为公共符号**。

**分布参数表（`_PARAMS_ALLOWED`）优先级最高**：它服务于 `Sampling.Variables[].distribution`，
是每一张卡都要写的东西，却在 schema 里完全空白（`parameters` 是 `additionalProperties: true`）。
把 10 种分布的参数写进 `common.json#/$defs/distribution`，man 与校验同时受益。

### 4.3 L3 散文：英文，内联进 schema JSON，由 lint 强制

**§15 决议 2**：散文载体是 JSON 内联 `description`，**不**做外部 `.md` 侧车。

理由：
- 输出既然是英文单语（决议 5），"JSON 里写多行中文很痛苦"这条反对意见不成立；
- 少一层间接 = 少一个失配面（外部文件路径、lint 反向检查、package-data glob 全部不需要）；
- **package-data 现有的 `schema/*.json` + `schema/**/*.json` 已经覆盖**，
  不必扩 glob，也就没有"开发树里手册正常、wheel 里全空"那个坑。

lint 扩展（`schema_catalog_lint_errors()`，照抄 zone lint 的模式）：

1. 每个 `x-jarvis-zone: closed` 的对象**必须**有非空 `description`；
2. 每个 `properties` 下的叶子键**必须**有 `description`（数值/字符串别名可共用一句）；
3. 每个 `manifest.sampling_methods` 条目**必须**能渲染出非空手册；
4. **所有 `description` / `title` / `x-jarvis-example` 必须是纯 ASCII**（K6 的机器化形态）。

第 4 条是 §15 决议 2/5 的执行手段：中文没法悄悄混进 man 输出，因为 lint 会在 CI 里拦下。

`docs/validation-diagnostics.md` 的「Authoring rules」已有"新增格式必须带 `x-jarvis-example`
和正负诊断测试"这条先例——手册作者义务是它的自然延伸。

### 4.4 与 `portal man` / `operas info` 的关系：两类文档，不是上下游

**§15 决议 4** 修正了初稿的定位。初稿把 Portal/Operas 当作 man 的"下一跳"，让 man 在边界上
只打印命令。这是错的：

| 问题 | 谁回答 |
|---|---|
| "`execution.input[]` 这条 YAML 该写哪些键？" | **`Jarvis man`** |
| "JSON 这个格式在物理上怎么读写文件、observables 怎么出来的？" | `Jarvis portal man JSON` |
| "`Operas.Modules[]` 该写哪些键？`input`/`output` 什么形状？" | **`Jarvis man operas`** |
| "`math.add` 这个算子签名是什么、干什么用的？" | `Jarvis operas info math.add` |

**man 页面必须自足**：读完 `Jarvis man calculator execution` 就能写出正确的 `input[]`，
不需要再跑第二条命令。§1.9 证明这做得到——14 个格式的 YAML 字段 schema（59 属性、
14/14 带 example）**本来就在 HEP2 自己的 catalog 里**，是 `_validate_selected_io()` 的校验依据。

保留的**唯一**运行时依赖是 `io_portal.available_io_formats(direction)`：
**格式清单**必须取自运行时（装了新版 Portal 就该立刻看到新格式），
这与 `task_schema.py:302-307` 已经写明的原则一致——
"Portal owns supported format names… 不能因为 Jarvis 还没打包详细 schema 就拒绝一个新格式"。

由此产生一个**必须显式呈现**的状态：若 Portal 报告了一个 HEP2 catalog 里没有 schema 的新格式，
man 要说清"该格式可用，但本版 Jarvis 没有它的字段手册"，并给出 `Jarvis portal man <FMT>`
作为**补充**（不是替代）。这是诚实呈现，不是转交。

---

## 5. 重点一：sampler man

### 5.1 单方法手册应当渲染什么

`Jarvis man sampler <METHOD>` 的段落：

| 段 | 数据来源 | 层 |
|---|---|---|
| 一行摘要 + 适用场景 | `description` | L3 |
| **能力**：stateless? resume 支持程度? | `Distributor` 注册表（`stateless=`, `resume=`，[`distributor.py:190-270`](../../Jarvis-HEP-v2/jarvishep2/distributor.py:190)） | L2 |
| 是否需要 `Sampling.Variables` | `contracts/methods.py:_METHODS_NEED_VARIABLES` | L1 |
| `Variables[].distribution` 对本方法的额外要求 | `require_length`（Bridson）/ `require_num`（Grid），`contracts/methods.py:104-105` | L1 |
| **Bounds 旋钮表**：名/类型/必填/默认/别名/下界/一行说明 | schema | L0+L3 |
| 最小可跑 YAML | `x-jarvis-example` | L3 |
| 会在这里触发的诊断码 | 码 → 块的索引（§9.2） | L1 |
| See also | 静态映射 | L3 |

**「能力」这一段是 man 独有的价值**：它回答"这个方法能不能 `--resume`"。实测注册表（17 条）：

```
stateless=True : Bridson, CSV, Grid, Random
stateless=False: AM, AMMCMC, AdaptiveBridson, DEMCMC, DRAM, Dynesty, Ensemble,
                 EnsembleMCMC, MCMC, MultiNest, PT, PTEnsemble, PTMCMC
resume         : implemented（17/17）
```

这个区分直接决定用户能不能中断重跑，今天散落在 `distributor.py` 的注册调用里，
任何文档都没有系统列出——值得印在手册第一屏。

### 5.2 补齐工作量分级

| 档 | 方法 | 现状 | 本 WP 要做的事 |
|---|---|---|---|
| **A：可直接渲染** | Bridson, Random, Grid, CSV, AdaptiveBridson | Bounds 已封闭、已有 example | 补英文 `description`；AdaptiveBridson 的 **30 个键**需逐条一行说明（最大的散文工作量） |
| **B：词表已在代码里，需搬进 schema** | Dynesty, MultiNest | 23 + 10 个键在 `dynesty_sampler.py` / `contracts/nested.py`，**校验已生效**（`JV2-BND-*`） | 把词表落成 schema properties + `description`；man 即刻有内容 |
| **C：surface 未定案** | MCMC, AMMCMC, AM, DRAM, EnsembleMCMC, Ensemble, DEMCMC, PTMCMC, PT, PTEnsemble | §1.5 的洞 | **不做**（§15 决议 1）；man 按 §5.3 标注 |

档 B 的注意点：`NESTED_CONSTRUCTOR_USER_KEYS` 目前是**硬编码的 frozenset**
（核对过 `dynesty_sampler.py:49-75` 是字面量，不是从 dynesty 签名反射）。
所以搬进 schema 不会引入"随第三方版本变动"的风险，是纯粹的位置搬迁。
**但要留一条测试断言两边集合相等**，防止将来只改一边。档 B 不改变校验强度（本来就在拒未知键）。

### 5.3 档 C：man 如何呈现"未定案"的方法

这 10 个方法今天的真实状态是：**能跑，但 `Bounds` 没有任何声明，拼错静默忽略**（§1.5 实测）。
man 面对它们有三种选择，只有一种是诚实的：

| 选项 | 后果 | 采纳 |
|---|---|---|
| 打印空的 Keys 表 | 用户以为"没有旋钮" | ❌ |
| 从 `mcmc_sampler.py` 反射出旋钮列进去 | 违反 K1：man 印了 `validate` 不认识的键，拼错仍然静默 | ❌ |
| **显式标注 surface 未定案** | 诚实，且不造成假安全感 | ✅ |

具体输出（英文）：

```
╭─ Sampling.Bounds · AMMCMC ───────────────────────────────╮
│ STATUS: not finalised. The MCMC family's Bounds surface   │
│ is not declared in this release: unknown keys are         │
│ accepted silently and are NOT validated. Knob names and   │
│ defaults may change without notice.                       │
╰───────────────────────────────────────────────────────────╯
```

同时在 `Jarvis man sampler`（无参表）里给这 10 行一个 `unstable` 标记。
**这条状态本身也要有数据来源**，不能硬编码在渲染器里：建议在这 10 个 method schema 上加
`"x-jarvis-status": "unstable"`（并让 lint 允许 `unstable` 的对象豁免 §4.3 第 1 条），
将来 MCMC 定案时删掉这个标记即可，man 自动转成正常渲染。

> **登记但不修**：§1.5 的静默放行仍是一个真实缺陷，只是被判定为"MCMC 未完成"的一部分。
> 建议单独记一行（不在 D24 内），等 MCMC 补完时连同 Bounds 声明一起做。

### 5.4 `Jarvis man sampler`（无参）

一张对照表，让用户先选方法。列：方法 / stateless / resume / 需要 Variables / 必填 Bounds 键 /
稳定性 / 一行适用场景。**全部由 L0–L2 生成，零手写**，因此永远不会漏掉新注册的方法。

---

## 6. 重点二：calculator man

### 6.1 主题划分

`Calculators` 的结构比 `Sampling` 深（`calculators.json` 有 44 个节点、11 个 zone），
一页塞不下。按 `<TOPIC>` 切：

| TOPIC | 覆盖 | 主要来源 |
|---|---|---|
| （无参） | 顶层五键 `Modules`/`Pools`/`make_paraller`/`Archiver`/`Cleanup`/`path` + **§6.4 连接链** | L0 + L3 |
| `module` | `Modules[]` 的 16 个字段：`name`（不含 `.`）、`path`、`source`、`clone_shadow`、`required_modules`、`installation`、`initialization`、`timeout`、`symlink_name`、`env_setup`、`selection`、`make_paraller`、`execution`、`modes` | L0 + L3 |
| `execution` | `path` / `commands` / `input[]` / `output[]`——**含每种格式的字段表**（§6.2） | L0 + L3 |
| `modes` | 多模式展开语义、`@PackID` 共享物理包、`modes` 与 `execution` 互斥 | L0 + `calculator_modes.py` |
| `pools` | `Calculators.Pools` 名→槽位数；`JV2-MOD-007/008` | L0 + L1 |

`module` 与 `mode` 的互斥关系值得单独一段：schema 里是
`module = moduleFields ∧ (if required[modes] then not required[execution])`，
`mode = moduleFields ∧ required[execution] ∧ not required[modes]`。
这个 `allOf/if/then/not` 组合**用户读不懂**，但一句话能说清：
**"A Module either declares `execution` directly, or declares `modes` (each carrying its own
`execution`) — never both."** 这正是 L3 散文存在的理由。

### 6.2 `execution.input[]` / `output[]`：man 自己讲完

按 §4.4，man 不把写法转交出去。分三段渲染：

1. **信封**：条目是列表；`name` 是这条 IO 的标识、`path` 是文件、`type` 是格式；
   `output[].name` 进 observables 命名空间（§6.4 规则 1）。来源：`calculators.json` + L3。
2. **格式清单**：`type` 的合法取值**取自运行时** `available_io_formats(direction)`
   （实测 input 7 / output 7，去重 8 个格式名）。
3. **该格式的字段表**：渲染 `manifest.io[direction][TYPE]` 指向的 schema
   （§1.9：59 个属性、14/14 带 `x-jarvis-example`）。
   `Jarvis man calculator execution --type JSON` 直接给出 JSON input 的 5 个字段 + 示例。

只有一种情况需要提第二条命令：**Portal 报了一个 catalog 里没有 schema 的新格式**——
此时明说"字段手册本版缺失"，并给 `Jarvis portal man <FMT>` 作为补充（§4.4 末段）。

### 6.3 `Jarvis man operas`

`Operas.Modules[]`（`core/operas.json`）：`name` + `operator`（必填）、`call_mode`（`call`/`acall`）、
`selection`、`required_modules`、`kwargs`（delegated）、`timeout`/`timeout_sec`、
`input[]`/`output[]`（`common#/$defs/operasIO` = `{name, entry?, expression?}`）。

man 把**这个块怎么写**讲完整。`operator` 那一栏的取值是"某个已注册算子的全名"——
man 说明取值形态与解析规则，**具体某个算子干什么**才是 `Jarvis operas info <NAME>` 的事（§4.4）。
表达式里能用哪些名字（含 D23 的 `pdg.mZ` 无括号常量）同理：man 讲**语法与作用域**
（Variables、前序 Mapper 名、Operas 函数与常量），具体常量数值与单位问 `jopera`。

> **已知冲突：D22.8（未修）**——schema 拒绝 `Operas.Modules[].input: ["x"]`，
> 而 `operas.py:319-321` 实现了这种简写。man 若照 schema 渲染，会告诉用户一种代码支持、
> schema 不支持的写法不存在。**建议 D22.8 先修，或 man 在该处显式标注该分歧**——
> 不要让手册去掩盖一个已登记的缺陷。

### 6.4 连接链：Calculator → Portal-IO → Operas

这一段不属于任何单一 schema 节点，是**跨块的数据流**，由 L3 散文承载，
放在 `Jarvis man calculator`（无参）首屏和 `Jarvis man yaml` 总览各一份。

```
Sampling.Variables[].name        <- sampler-produced free parameters (u space)
        |
        +- Sampling.Mapper[]     <- optional pure reparameterisation (D22)
        |                           may reference: Variables, earlier Mapper names,
        |                           Operas functions and constants (D23.9)
        v
   parameter namespace { x, y, mz, ... }
        |
        +--------------------------------------------+
        v                                            v
Calculators.Modules[].execution.input[]      Operas.Modules[].input[]
   type: JSON/SLHA/...  (Portal writes files)   {name, expression}  (in-process)
   entries reference parameter names             expression references parameter names
        |                                            |
   (external program runs)                      (Python operator runs)
        |                                            |
        v                                            v
Calculators...execution.output[]             Operas.Modules[].output[]
   {name, column/row/block ...}                 {name, entry}
        |                                            |
        +----------------------+---------------------+
                               v
                      observables namespace
                               |
                               v
             Sampling.LogLikelihood[].expression
             (a term named LogL IS the total; otherwise terms are summed)
                               |
                               v
                      Sampling.selection
```

三条容易踩的规则，man 必须明写（都来自现有代码，非新增）：

1. **名字是唯一的粘合剂**。Calculator 的 `output[].name` 与 Operas 的 `output[].name`
   进的是同一个 observables 命名空间；重名即互相覆盖。
2. **顺序由 `required_modules` 决定**，不是书写顺序。Operas 模块要用 Calculator 的产物，
   必须 `required_modules: [那个 Calculator 名]`。
3. **`selection` 的求值时机**：`Sampling.selection` 在**接受索引 / uuid 分配之前**执行
   （`bridson.py:195-206`）——D21/D22 的 resume 不变量。
   写进手册能避免用户把有副作用的表达式塞进 `selection`。

### 6.5 令牌

`&J`、`@PackID`、`@Sdir`、`@SampleID` 几乎只在 Calculators 块里用，所以
`Jarvis man calculator` 末尾给一行 `Jarvis man tokens`（§7）。

---

## 7. `Jarvis man tokens`

实测到的完整令牌集，两阶段解析（`command_parser.py`）：

| 令牌 | 展开为 | 阶段 | 实现 |
|---|---|---|---|
| `&J` / `&J/…` | 项目根绝对路径 | 静态（控制端） | [`base.py:37-51`](../../Jarvis-HEP-v2/jarvishep2/base.py:37) |
| `${LibDeps:NAME}` | 该 LibDeps 条目的值或路径；**未注册即 `KeyError` 硬失败** | 静态 | `command_parser.py:317-323` |
| `${Scan:name}` | `Scan.name` | 静态 | `command_parser.py:309-315` |
| `${Scan:root\|output\|outputs}` | `<project>/outputs/<scan>` | 静态 | 同上 |
| `@{ROOT path}` | `EnvReqs.CERN_ROOT` 解析出的路径；未配置则报可读错误 | 静态 | `command_parser.py:325-331` |
| `@PackID` | 该 Calculator 的物理包槽位 id | 运行时（Worker） | `calculator_pools.py:64-69` |
| `@Sdir` | 本 sample 的工作目录 | 运行时 | `SAMPLE_TOKENS`, `sample.py:695` |
| `@SampleID` | 本 sample 的 uuid | 运行时 | `SAMPLE_TOKENS` |

手册必须讲清的三件事：

1. **`&J` 是静态的、`@Sdir` 是运行时的**——把 `@Sdir` 写进 `installation` 命令没有意义
   （安装发生在任何 sample 之前）。`worker_config.py:52` 和 `runtime_config.py:479` 都在
   探测 `@Sdir` 的存在与否来决定调度形态，说明这个区分有实际后果。
2. **多模式 Calculator 的 `path` 必须含 `@PackID`**，否则 `JV2-MOD-006`
   （`calculator_modes.py:262-268`，错误文案已写好："Use a path such as runtime/Prospino/@PackID"）。
3. **`&J/…/src/card/…` 是被显式拒绝的历史路径**（`base.py:43-47`），
   手册应直接给替代写法 `&J/deps/…`。

> 顺带记录一个小瑕疵（不在本 WP 范围，但手册不该掩盖）：
> `_replace_scan_token` 对无法识别的 token 名**静默回落**到 output root
> （`command_parser.py:315`），所以 `${Scan:typoo}` 不报错。
> man 只应列出 `name` / `root` / `output` / `outputs` 四个受支持的值；
> 若要修成"未知名报错"，另开 WP。

---

## 8. `Jarvis man example`

### 8.1 语料 = 已随包发布的模板卡（§1.8）

| `<NAME>` | 文件 | 实测 |
|---|---|---|
| `bridson` | `project_template/bin/quickstart_bridson_operas.yaml` | 0 error 0 warning |
| `csv` | `project_template/bin/quickstart_csv_operas.yaml` | 0/0 |
| `dynesty` / `dynesty-full` | `project_template/bin/sampling/Sampling_Dynesty_{Simple,Full}.yaml` | 0/0 |
| `multinest` / `multinest-full` | `project_template/bin/sampling/Sampling_MultiNest_{Simple,Full}.yaml` | 0/0 |

**缺口**：没有任何一张模板卡带 `Calculators` 块，而 `Jarvis man example calculator` 是明确需求。
需**新增一张最小 Calculator 示例卡**，且必须真的能跑（否则又是一份会腐烂的散文）。
建议 `project_template/bin/quickstart_calculator.yaml`：
一个 shell 计算器（`echo` 写一个 JSON）+ JSON input/output + 一个 Operas 模块消费它，
这样它同时是 §6.4 连接链的可执行示例。**这是本 WP 里唯一需要新写 YAML 的地方。**

### 8.2 门禁

- `Jarvis man example` 列出的每一张，CI 里必须 `Jarvis validate` 通过（今天六张已全绿，加测试即可锁住）。
- Calculator 示例卡额外要求 `Jarvis check`（check-modules 冒烟）通过。
- 示例文件**只有一份**：`man example` 读的就是 `project create` 用的那份，不允许出现第二份副本。

---

## 9. 面向 coding agent 的契约

### 9.1 `--json` 形态

稳定键，风格对齐现有 `validate --json`：

```json
{
  "path": "$.Sampling.Bounds",
  "context": {"method": "Bridson"},
  "zone": "closed",
  "status": "stable",
  "summary": "Method-specific sampler knobs.",
  "keys": [
    {"name": "Radius", "type": "number", "required": true, "minimum": 0,
     "exclusive": true, "default": null, "aliases": [],
     "description": "Poisson-disk minimum separation in unit-cube coordinates."},
    {"name": "Seed", "type": "integer", "required": false, "default": 0,
     "aliases": ["seed"], "description": "..."}
  ],
  "examples": ["Bounds:\n  Radius: 0.1\n  MaxAttempt: 30"],
  "diagnostics": ["JV2-MTH-010", "JV2-MTH-011", "JV2-MTH-012", "JV2-MTH-013"],
  "see_also": ["Jarvis man yaml Sampling.Variables", "Jarvis man example bridson"],
  "further_reading": null
}
```

- `status`：`"stable"` | `"unstable"`（§5.3 的 MCMC 家族）。
  **agent 看到 `unstable` 就知道这里的键名可能变、且拼错不会被拒。**
- `further_reading`：**不是转交**，是补充。非空形如
  `{"command": "Jarvis portal man JSON", "topic": "what the JSON adapter does at runtime"}`。
  YAML 写法本身永远在 `keys[]` 里给全（§4.4）。

### 9.2 诊断码 → 手册的闭环

这是 agent 最关心的一环，也是修掉 §1.7 那条死链的地方。

1. 建立 `code → path` 索引。`issue()` 已带 `code` 和 `path`，
   在 contracts 里为每个码登记它所属的**规范块路径**（一次性静态表，用测试断言全覆盖）。
2. `Jarvis man --code JV2-MTH-012` → 打印 `$.Sampling.Bounds`（Bridson 上下文）的手册。
3. **把 `hint` 从 `"See docs/task-card-schema.md …"` 改成
   `"Run: Jarvis man yaml $.Sampling.Bounds"`**——一条可执行命令，wheel 用户和 agent 都能用。

于是 agent 的循环变成：

```
Jarvis validate card.yaml --json
  -> issues[i].code / .path
  -> Jarvis man yaml <path> --json     (or --code <code>)
  -> 按 keys[] / examples[] 改卡
  -> 重跑 validate
```

**全程不需要仓库、不需要网、不需要读 1650 行文档。**

### 9.3 稳定性承诺

`--json` 顶层键（`path`/`context`/`zone`/`status`/`summary`/`keys`/`examples`/`diagnostics`/
`see_also`/`further_reading`）与 `keys[]` 的键名进入公共接口；只增不改不删。
`keys[]` 顺序 = schema 声明顺序，稳定可 diff。

---

## 10. 落地实现点

| # | 文件 | 改动 |
|---|---|---|
| 1 | `jarvishep2/man.py`（新） | 渲染器：域/路径解析、L0–L2 提取、Rich 版式、`--json` 序列化 |
| 2 | `jarvishep2/client.py` | 新增 `man` 子命令；加进 `_SUBCOMMANDS`（:37）、`_COMMAND_HELP_PANELS`（:76，建议归 "Scan workflow"）、`_COMMAND_HELP_ORDER`（:90，建议 `15`，紧挨 `validate: 10`）。**注意 `--man` 是 `project pack` 的旗标（:61），不冲突** |
| 3 | `jarvishep2/task_schema.py` | 导出只读 catalog 访问器供 man 用（不改 `validate_task_card_schema` 现有签名）；扩 `schema_catalog_lint_errors()` 加 §4.3 四条 |
| 4 | `jarvishep2/schema/**/*.json` | 内联英文 `description`（档 A 全部 + 档 B 词表 + `distribution` 的 10 组参数）；档 C 的 10 个方法加 `x-jarvis-status: unstable` |
| 5 | `jarvishep2/contracts/{variables,nested,methods}.py` | 把 §4.2 私有词表提升为公共导出；登记 `code → path` 表（§9.2） |
| 6 | `jarvishep2/task_validation.py` | `JV2-SCH-*` 的 `hint` 换成可执行命令（§9.2 第 3 条） |
| 7 | `jarvishep2/project_template/bin/quickstart_calculator.yaml`（新） | §8.1 的缺口 |
| 8 | `tests/test_man.py`（新） | §12 验收 |
| 9 | `docs/`、`YAML_REFERENCE_2.0.md` | 交叉引用；`YAML_REFERENCE` 头部注明"运行时真相以 `Jarvis man` 为准" |

**`pyproject.toml` 不需要改**：散文内联在 JSON 里，现有 package-data 已覆盖（§4.3）。

**不得触碰**：任何采样器的数值路径、`build_sympy_dicts` 的公共签名（D23 的封闭契约）、
`mcmc_sampler.py`（§15 决议 1）、V1（`Jarvis-HEP`）任何文件。

---

## 11. 不变量

- **K1**：man 只渲染校验器认识的东西。**新增一个 man 键 ⇒ 必须同时让 `validate` 能拒绝它的拼写错误。**
  校验暂时管不到的 surface（档 C）**标注为 `unstable`**，不假装有手册（§5.3）。
- **K2**：不新增第五个真相源。散文内联在 schema 并被 lint 绑定；结构/词表/能力全部读现有对象。
- **K3**：**man 页面自足**。YAML 写法在 man 里讲完；`portal man` / `operas info` 是"模块行为"的
  补充读物，不是 YAML 写法的答案所在（§4.4）。
- **K4**：`man example` 的语料就是 `project create` 的语料，**唯一一份**。
- **K5**：`--json` 的公共键只增不改不删（§9.3）。
- **K6**：**man 的所有输出永远是英文纯 ASCII 散文**；由 lint 第 4 条在 CI 强制，
  且渲染不经过 `_ascii_diagnostic`（§3.4）。
- **K7**：`YAML_REFERENCE_2.0.md` 与 man 角色分离——前者是评审用 as-built 全集（含 Appendix A 已知坑），
  后者是用户用的运行时真相。**两者冲突时以 man（即校验器）为准**，并按 D22.1 的做法把差异登记成 WP。

---

## 12. 验收

| # | 验收项 | 判据 |
|---|---|---|
| A1 | `Jarvis man` 无参可跑 | 列出全部 manual 域，退出码 0 |
| A2 | 路径两种前缀等价 | `Sampling.Bounds` 与 `$.Sampling.Bounds` 输出逐字节相同 |
| A3 | 数组下标抹平 | `$.Calculators.Modules[0].execution.input[0]` 命中 `…input[]` |
| A4 | `yaml` 前缀可选 | `Jarvis man calculator execution` 与 `Jarvis man yaml calculator execution` 输出相同 |
| A5 | 未命中给建议 | `Jarvis man yaml Samplng` → did-you-mean `Sampling`，退出码非 0 |
| A6 | **17 个方法全部有非空手册** | 遍历 `manifest.sampling_methods`：档 A/B 的 7 个 `keys[]` 非空且带 `description`；档 C 的 10 个 `status == "unstable"` 且带明确警示文案 |
| A7 | 档 B 词表一致 | schema 里 dynesty 构造器键集合 == `NESTED_CONSTRUCTOR_USER_KEYS`（23 个） |
| A8 | 分布参数完整 | `Jarvis man yaml Sampling.Variables` 列出 10 种分布各自的必填参数，与 `_PARAMS_REQUIRED` 逐项相等 |
| A9 | **man 自足** | `Jarvis man calculator execution --type JSON` 直接给出 JSON input 的 5 个字段 + 示例，输出中**不含**"run `Jarvis portal man`"式的转交措辞 |
| A10 | 格式清单来自运行时 | mock `available_io_formats` 增加一个假格式，man 输出立即包含它，并标注"字段手册本版缺失" |
| A11 | example 全绿 | `Jarvis man example` 的每一张 `Jarvis validate` 0 error；calculator 那张 `Jarvis check` 通过 |
| A12 | 码 → 手册闭环 | 每个 `JV2-*` 码都能 `--code` 命中一个块（测试遍历全部码，无孤儿） |
| A13 | hint 可执行 | `validate --json` 里不再出现 `docs/*.md` 字样 |
| A14 | **英文 ASCII 门禁** | lint 断言全部 `description`/`title`/`x-jarvis-example` 为纯 ASCII；man 全域输出扫描无非 ASCII 字母 |
| A15 | wheel 完整 | 从构建出的 wheel 装到干净 venv，`Jarvis man sampler Bridson` 有散文 |
| A16 | 无回归 | `pytest -q` 与基线一致；六张模板卡仍 0/0（本 WP 不收紧任何 Bounds） |
| A17 | 冷启开销 | `Jarvis man yaml <path>` 端到端 < 1.5 s（schema catalog 有 `lru_cache`，不应触发 Operas 冷发现） |

A17 值得单测：man **不应该**为了打印一个 Sampling 字段去做 Operas 冷发现
（实测冷发现 0.86–1.19 s/进程）。除非用户问的正是 Operas 相关路径。

---

## 13. 明确不做

- 不做 MCMC 家族的 Bounds 声明化与收紧（§15 决议 1）；§1.5 的静默放行**保持开放**，
  建议随 MCMC 完成时一并处理。
- 不做 TUI / 交互向导。
- 不把 `YAML_REFERENCE_2.0.md` 整体打包（§2.2）。
- 不做外部 `.md` 散文侧车（§15 决议 2）。
- 不改 `Jarvis portal man` / `Jarvis operas info` 的任何行为（§15 决议 4：两类文档并存）。
- 不修 D22.8（Operas `input: ["x"]` 简写）、不修 D23.14（`_ascii_diagnostic` 过度应用）、
  不修 `${Scan:typo}` 静默回落——三者各有独立 WP；man 只**如实呈现**，不掩盖。
- 不改任何 V1 文件。
- 不引入新第三方依赖（Rich 与 jsonschema 都已在用）。

---

## 14. 建议的 WP 拆分（待登记）

| WP | 标题 | 优先级 | 依赖 |
|---|---|---|---|
| D24.1 | `man.py` 渲染器骨架 + `client.py` `man` 子命令接线 + 域/路径语法（A1–A5） | HIGH | — |
| D24.2 | L0/L2 提取：schema catalog 只读访问器 + Distributor 能力表；`Jarvis man sampler`（无参表）（A6 结构部分） | HIGH | D24.1 |
| D24.3 | L3 散文层：内联英文 `description` + lint 四条（含 ASCII 门禁）+ 档 A 五个方法（A6, A14） | HIGH | D24.1 |
| D24.4 | 档 B：dynesty/multinest 词表搬进 schema + 一致性测试（A7） | MEDIUM | D24.2 |
| D24.5 | 档 C：`x-jarvis-status: unstable` 标记 + 警示渲染（A6 后半） | MEDIUM | D24.2 |
| D24.6 | `distribution` 10 组参数写进 `common.json`（A8） | HIGH | D24.3 |
| D24.7 | `Jarvis man calculator`：五个 TOPIC + §6.4 连接链散文 + `--type` 格式字段表（A9, A10） | HIGH | D24.3 |
| D24.8 | `Jarvis man operas` + `Jarvis man tokens` | MEDIUM | D24.3 |
| D24.9 | `Jarvis man example` + `quickstart_calculator.yaml` + 门禁（A11） | MEDIUM | D24.1 |
| D24.10 | 码 → 手册索引 + `--code` + `hint` 换成可执行命令（A12, A13） | MEDIUM | D24.2 |
| D24.11 | `--json` 契约冻结 + agent 端到端测试 + wheel 门禁（§9.3, A15, A17） | MEDIUM | D24.10 |

**最小可用切片**：D24.1 + D24.2 + D24.3 + D24.6 + D24.7。
到这里用户已经能 `Jarvis man`、`… sampler Bridson`、`… yaml Sampling.Variables`、
`… calculator execution --type JSON`，且散文层的 lint（含英文 ASCII 门禁）已经就位。

---

## 15. 取舍决议（2026-08-08 拍板）

| # | 问题 | 决议 | 影响 |
|---|---|---|---|
| 1 | 收紧 MCMC `Bounds` 报 error 还是 warning？ | **略过**——MCMC 家族尚未添加完成 | 档 C 出本 WP 范围；man 标 `unstable`（§5.3）；§1.5 的洞保持开放并单独记录 |
| 2 | 散文载体：外部 `.md` 侧车 vs JSON 内联？ | **JSON 内联 `description`**，且不允许中文 | §4.3 重写；`pyproject.toml` 不必改；lint 加 ASCII 门禁 |
| 3 | 命令语序 `Jarvis yaml man` vs `Jarvis man yaml`？ | **`Jarvis man yaml`**，`Jarvis man` 是总的 manual 中心 | §3.1 重写为域中心；`yaml` 前缀对其他域可选（A4） |
| 4 | 与 `portal man` / `operas info` 的关系？ | **两类文档**：man 教怎么写 YAML，其他解释模块功能 | §4.4 从"转交下一跳"改为"man 页面自足"（K3）；§1.9 证明数据已在手；`delegates_to` 改名 `further_reading` |
| 5 | man 输出语言？ | **永远英文** | K6；lint 第 4 条 + A14 强制 |
