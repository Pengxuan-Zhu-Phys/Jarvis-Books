# DESIGN — Multi-mode Calculators (`Calculators.Modules[].modes`, V2, D20)

**Status**: design proposal 2026-08-01 — **awaiting maintainer decision on §6**; implementation `todo`
**Date**: 2026-08-01
**Requirement (maintainer)**: *"一个软件包原则上有多种用法，即在一次扫描任务中需要调用两次，
但这两次的初始化和执行命令不一样，也许还需要重新构建。所以我没想清楚 YAML 怎么设计。"*

---

## 1. 现状

`Calculators.Modules[].modes` 在 V2 schema 里被声明为 `string | [string]`，**零代码消费**
（`Module/calculator.py` 与 `worker.py` 中出现次数均为 0）。V1 也只在
`Module/calculator.py:18-19` 设了个布尔标志后从不使用——**是没做完的设计，不是废弃功能**
（维护者澄清，2026-08-01）。

注意：现有的 `string | [string]` 形态**表达不了**需求——它只能列出模式名，装不下"每个模式
各自的 initialization / execution / 构建"。所以这个键无论如何都要重新定义，不存在
"保持向后兼容"的包袱（0 张真实卡片在用）。

## 2. 需求拆解

一个软件包，在一次扫描中被用作 N 种不同用途。以真实场景为准：

| 场景 | 每模式不同的部分 | 是否需要各自构建 |
|---|---|---|
| **MadGraph**：生成 ProcA / ProcB | 进程卡、生成出的目录、运行命令 | **是**——每个进程是一次独立的代码生成 |
| **FlexibleSUSY**（你们 HinoLLP 在用）：不同模型 | 模型定义、编译产物 | **是** |
| **micrOMEGAs**：relic density / direct detection | 调用的例程、输出量 | 否——同一份二进制，不同调用 |
| **SPheno**：不同输入卡 | 输入文件、输出量 | 否 |

所以设计必须同时支持"要重新构建"和"共用一份构建"两类，而**不能**要求用户为后者付出前者的代价。

## 3. 核心设计：`modes` 是**声明层的展开**，不是运行时的新概念

> **一个模式 = 一个继承了公共声明的兄弟 calculator 模块。**
> `modes` 在配置装载期展开成 N 个模块，运行时**不新增任何概念**。

这是本设计的关键判断。V2 运行时**已经**具备模式所需的一切：

| 模式需要 | 运行时已有的机制 |
|---|---|
| 各自的执行命令 / 输入输出 | 模块的 `execution` |
| 各自的依赖顺序 | `required_modules` → `workflow.resolve_module_layers` |
| 各自的并发槽位 | `calc:free:<name>` 池 + PackID |
| 各自的构建 | `installation` + `jarvis_install.json` 控制（D13.11） |
| 各自的跳过条件 | 模块级 `selection`（D12.1 已交付） |
| 各自的超时 | 模块的 `timeout` |

因此**唯一真正新增的东西是"共享声明"**，那是配置展开的职责。展开之后，
`MadGraph.ProcA` 就是一个普普通通的模块，池、pack、层、flowchart、日志、DATABASE 一律照旧。

**这条判断决定了后续每一个细节**，也是我建议采纳它的理由：任何把模式做成运行时特殊分支的方案，
都要在 Worker、池管理、层推导、flowchart 五个地方各加一条 if。

## 4. YAML 形态

### 4.1 最小可用

```yaml
Calculators:
  Modules:
    - name: micrOMEGAs
      clone_shadow: true
      path: "&J/calculators/micrOMEGAs/@PackID"
      source: "&J/src/micrOMEGAs"
      installation:                        # 公共构建：在每个模式的 pack 里各跑一次
        - "cp -r ${source}/* ${path}"
        - "make -j4"
      modes:
        - name: Relic                      # → 模块 micrOMEGAs.Relic
          execution:
            commands: ["./main relic.in"]
            output:
              - {name: relic_out, path: "@Sdir/relic.json", type: JSON}
        - name: DirectDet                  # → 模块 micrOMEGAs.DirectDet
          execution:
            commands: ["./main dd.in"]
            output:
              - {name: dd_out, path: "@Sdir/dd.json", type: JSON}
```

### 4.2 每模式各自构建（MadGraph / FlexibleSUSY）

```yaml
    - name: MadGraph
      clone_shadow: true
      path: "&J/calculators/MadGraph/@Mode/@PackID"    # @Mode 使各模式目录分离
      source: "&J/src/MG5"
      installation:                        # 公共部分
        - "cp -r ${source}/* ${path}"
      modes:
        - name: ProcA
          installation:                    # 模式专属构建，在公共部分之后执行
            - "./bin/mg5_aMC ${mode_dir}/procA.dat"
          initialization: ["rm -f ${path}/Events/*"]
          execution:
            commands: ["./ProcA/bin/generate_events -f"]
            output: [{name: xsec_A, path: "@Sdir/A.dat", type: DAT}]
        - name: ProcB
          installation:
            - "./bin/mg5_aMC ${mode_dir}/procB.dat"
          execution:
            commands: ["./ProcB/bin/generate_events -f"]
            output: [{name: xsec_B, path: "@Sdir/B.dat", type: DAT}]
```

### 4.3 模式之间的依赖

模式是普通模块，直接用点号寻址：

```yaml
        - name: DirectDet
          required_modules: ["micrOMEGAs.Relic"]     # 先算 relic，再算 DD
```

## 5. 展开规则（实现契约）

1. **命名**：模式模块名 = `<模块名>.<模式名>`。模块名本就是自由字符串且直接用作
   Redis 池键（`calc:free:<name>`），点号无需转义。
2. **继承**：模式**继承**模块的 `source` / `clone_shadow` / `env_setup` / `timeout` /
   `make_paraller` / `selection` / `required_modules`；模式内同名键**覆盖**之。
3. **命令拼接**：`installation` 与 `initialization` 为**模块级在前、模式级在后**拼接
   （公共构建 → 专属构建）。`execution` **不拼接**，必须由模式提供（模块级 `execution`
   在有 `modes` 时非法——见 §7 校验）。
4. **新 token `@Mode`**：展开为模式名，供 `path` 模板分目录；`${mode_dir}` 展开为该模式的
   运行目录，与现有 `${path}`/`${source}` 同族。
5. **池与 pack 按模式独立**——见 §6.1，这不是可选项。
6. **`required_modules` 中的裸模块名**（如 `["MadGraph"]`）展开为"该模块的全部模式"，
   便于"等这个包的所有用途都算完"。

## 6. 三个曾经想不清楚的问题，及答案

### 6.1 模式与 PackID 槽位的关系 → **必须每模式独立，无需设开关**

这个问题看似要在"每模式一套 pack"和"共享一套 pack"之间权衡，**但现有并发模型已经把答案定死了**：

`workflow.resolve_module_layers` 会把**互不依赖的模块排进同一层**，而同层 calculator 在一个
样本内**并发执行**（D2.2）。所以两个独立的模式会同时运行——它们**不可能共用一个 pack 目录**，
否则就是两个进程同时写同一个工作目录，正是 clone_shadow 要防的事。

结论：**每个模式有自己的池（`calc:free:MadGraph.ProcA`）和自己的 `001…N` pack 目录**。
不提供 `shared/per_mode` 开关——那个开关的"共享"档在并发下是错的。

想省磁盘的用户用现有手段即可：`Calculators.Pools: {MadGraph.ProcA: 1}` 限制槽位数。

### 6.2 `required_modules` 如何引用模式 → **点号寻址，裸名 = 全部模式**

`"MadGraph.ProcA"` 指定单个模式；`"MadGraph"` 表示"该模块的所有模式"（展开为全部模式名）。
后者覆盖"等这个包全部用途算完"的常见意图，且让**老卡片的裸名引用在加 modes 后语义仍然正确**。

### 6.3 `execution.input/output` 是否按模式声明 → **必须按模式，且要防重名**

各模式产出不同的物理量，`input`/`output` 只能在模式内声明。

**但这里有个必须堵上的坑**：两个模式若声明了**同名的 output**（如都叫 `omega_h2`），
它们会先后写进同一个 `sample.observables`，**后者静默覆盖前者**——又是一个"跑得很正常、
结果悄悄不对"的失败。

处置：**装载期校验**——同一模块下任意两个模式声明了相同的 output `name` 即报错，
提示改名或用 `@Mode` 前缀。不做自动改名（自动加前缀会改变 DATABASE 列名，
且让用户的表达式对不上）。这与 D17 "宁可装载期报错，不要运行期静默错" 的原则一致。

## 7. 校验规则（D17 zone: `closed`）

| 规则 | 错误码（建议） |
|---|---|
| `modes` 非空数组，元素含 `name` | `JV2-MOD-001` |
| 有 `modes` 时，模块级不得出现 `execution` | `JV2-MOD-002` |
| 同模块内模式名重复 | `JV2-MOD-003` |
| 同模块内两个模式的 output `name` 冲突（§6.3） | `JV2-MOD-004` |
| `required_modules` 引用了不存在的 `模块.模式` | `JV2-MOD-005`（带 did-you-mean） |
| 模式名含非 ASCII / 点号 | 复用 `JV2-ENC-001` / 新增名称字符约束 |

## 8. 与既有能力的边界

- **公共且昂贵的构建应放 LibDeps，而不是模块级 `installation`。** 因为每个模式有自己的 pack，
  模块级 `installation` 会在**每个模式的每个 pack 里各跑一次**。真正"全局装一次"的东西
  （ROOT、Delphes）属于 LibDeps（D18 已实现）。文档要把这条写清楚，否则用户会在
  `installation` 里放重型编译然后抱怨慢。
- **不扩展到 `Operas.Modules`**：Operas 算子是纯函数调用，多用途直接写成多个模块即可，
  没有共享构建的问题。

## 9. 工作包（建议）

| WP | 内容 | 验收 |
|---|---|---|
| D20.1 | 配置展开器 + schema（`modes` 重定义，zone `closed`） | 一张双模式卡片展开为两个模块；`Jarvis2 validate` 覆盖 §7 全部规则 |
| D20.2 | `@Mode` / `${mode_dir}` token + 每模式 pack 路径 | 两模式的 pack 目录互不重叠；并发跑不互踩 |
| D20.3 | `required_modules` 点号寻址 + 裸名展开 | 依赖序正确；跨模式依赖进入正确的层 |
| D20.4 | 文档 + skill（`multi-mode-calculator.md`，卡片实测通过） | D16 规则 |

**回滚**：不写 `modes` 的卡片走今天的路径，一字不变。

## 10. 需要维护者拍板的两点

1. **寻址分隔符用点号还是冒号**：本设计用 `MadGraph.ProcA`。若担心与 Operas 的
   `namespace.function` 视觉混淆，可改 `MadGraph:ProcA`。两者实现代价相同，纯偏好问题。
2. **裸名 `required_modules: ["MadGraph"]` 的语义**：本设计定为"全部模式"。
   另一种选择是"报错，必须写明模式"——更严格但会让老卡片在加 modes 后失效。
   建议维持"全部模式"，理由见 §6.2。
