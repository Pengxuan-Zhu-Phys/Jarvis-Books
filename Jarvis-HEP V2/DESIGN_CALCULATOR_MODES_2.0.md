# DESIGN — Multi-mode Calculators (`Calculators.Modules[].modes`, V2, D20)

**Status**: design proposal 2026-08-01 — **awaiting maintainer decision on §10**; implementation `todo`
**更新 2026-08-01**: 补 §5.1-5.3（触发语义）与 §5.4（主场景为原地重建，新增可选 build 阶段）
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

### 5.1 installation / initialization 的触发语义（维护者提问，2026-08-01）

两个 block **都是可选的**。它们的触发时机完全不同，这是本设计最容易误解的一点，因此单列一节。

| block | 触发频率 | 由谁把关 |
|---|---|---|
| `installation` | **每个 pack 目录一生一次** | 安装 stamp 的指纹 + epoch（D13.11） |
| `initialization` | **每个样本一次**，无条件 | 无（V1 契约就是每次都跑） |

`installation` **既不是"每次都触发"，也不是"按需把 pack 改造成目标模式"**。精确行为如下
（沿用 `RuntimePreparer.prepare()` 现有路径，模式不引入新分支）：

```
Worker 拿到 pack
  └─ ensure_shadow_installed()      ← 每个样本都会"检查"
       ├─ 进程内已装过该 pack？      → 直接返回（同一 Worker 内只查一次）
       ├─ stamp 指纹 + epoch 匹配？  → 直接返回（跨 Worker、跨 run 复用）
       └─ 否则                       → 真正执行 installation 命令，然后写 stamp
  └─ run_initialization()           ← 每个样本无条件执行
```

即：**检查每次都做（读一个 stamp 文件，开销可忽略），命令执行是每个 pack 目录一生一次**，
除非操作者在 `jarvis_install.json` 里置 `reinstall: true`。

### 5.2 为什么不会出现"按需切换模式"——每模式独立 pack 的第二个理由

维护者提出的另一种可能：每次拿到 calculator 时判断它当前属于哪个模式，若不是目标模式，
就用 `installation` 把它改造过去。

**本设计里这种情况不会发生**，因为 `MadGraph.ProcA` 的 `001` 和 `MadGraph.ProcB` 的 `001`
是**两个不同目录**，永远不需要互相转换。

而这恰是**共享 pack 方案的致命伤**，也是 §6.1 结论的第二条独立论据（§6.1 只给了并发那条）：
若模式共用 pack，就**只能**采用按需改造，于是扫描在 ProcA / ProcB 之间交替时，
**每交替一次就要重新生成一次 MadGraph 进程**，最坏每个样本重建一次。stamp 机制救不了这个场景，
因为两个模式的指纹本来就应该不同，"指纹不匹配"在这里是常态而非异常。

一句话：**每模式独立 pack，把"模式切换"这件事从运行时彻底消掉了。**

### 5.3 代价：模块级 installation 会跑「模式数 × pack 数」次

因为每个模式有自己的 `001…N`，模块级（公共）`installation` 会在**每个模式的每个 pack**
里各跑一次。2 个模式 × `make_paraller: 4` = **8 次**。

所以 §8 那条边界要当硬性建议读：**真正昂贵且公共的构建放 LibDeps**（全局装一次），
模块级 `installation` 只留轻量准备（解包、拷贝、软链）。

## 5.4 主场景修正：「改 config 再 make」的原地构建（维护者澄清，2026-08-01）

> *"有一些软件的模式切换是改一下 config 配置之后重新 make。这在早期用 Fortran 构建的软件中很常见。"*

这是 `modes` 的**主场景**，比 §2 里举的 MadGraph 更普遍，而且**证据就在你们自己的卡片里**：

```yaml
# iDM_Vector_Bridson.yaml
- name: MicroOMEGAs_Vector
  path: "&J/calculators/microOMEGAs_vector/@PackID"
  installation:
    - "cp -R ${source}/. ${path}"
    - "cd ${path}/vector-iDM && make clean && make main=main.cpp"

# iDM_Axial_Bridson.yaml
- name: MicroOMEGAs_Axial
  path: "&J/calculators/microOMEGAs/@PackID"
  # 卡片原注释: "Same micrOMEGAs 7 base package as the vector scan;
  #             axial-iDM differs only in its axial DM-current model files."
```

**同一个包、只差模型文件，现在要把整个模块声明复制一份、还分散在两张卡片里。**
`modes` 要消灭的正是这个。

### 5.4.1 这不改变 §6.1 的结论——每模式独立 pack 依然正确

原地构建意味着**一个目录在同一时刻只能是一个模式**，这反而让"共享 pack"更不可行。三种方案的编译次数：

| 方案 | micrOMEGAs 编译次数（2 模式 × `make_paraller: 4`） |
|---|---|
| 共享 pack + 按需改造 | **每次模式交替一次**——最坏每样本一次，成千上万次 |
| 每模式独立 pack（§3 原设计） | **8 次**（首次），之后 stamp 复用 |
| 每模式独立 pack + 全局构建（§5.4.2） | **2 次**（每模式一次），之后 pack 只拷贝 |

### 5.4.2 补充 `build` 阶段：昂贵的模式构建做一次，pack 只拷贝

对"config + make"这类软件，**昂贵的部分恰恰是模式专属的**（公共部分只是 `cp -R`）。
若照 §5 原样把 `make` 放进模式的 `installation`，就会 **模式数 × pack 数** 次编译。

因此为模式增加一个**可选的 `build` 阶段**：**全局每模式一次**，产物再被各 pack 拷贝。

```yaml
- name: micrOMEGAs
  clone_shadow: true
  path: "&J/calculators/micrOMEGAs/@Mode/@PackID"
  source: "&J/src/micromegas"
  modes:
    - name: Vector
      build:                                   # ← 全局一次，每模式一份构建产物
        path: "&J/builds/micrOMEGAs.Vector"
        commands:
          - "cp -R ${source}/. ${build_dir}"
          - "cd ${build_dir}/vector-iDM && make clean && make main=main.cpp"
      installation:                            # ← 每 pack 一次，只拷贝产物（廉价）
        - "cp -R ${build_dir}/. ${path}"
      execution:
        commands: ["./vector-iDM/main"]
        output: [{name: omega_vec, path: "@Sdir/out.json", type: JSON}]
    - name: Axial
      build:
        path: "&J/builds/micrOMEGAs.Axial"
        commands:
          - "cp -R ${source}/. ${build_dir}"
          - "cd ${build_dir}/axial-iDM && make clean && make main=main.cpp"
      installation:
        - "cp -R ${build_dir}/. ${path}"
      execution:
        commands: ["./axial-iDM/main"]
        output: [{name: omega_ax, path: "@Sdir/out.json", type: JSON}]
```

**三段式生命周期**（与现有两段式向下兼容，`build` 不写就退化成今天的行为）：

| 阶段 | 频率 | 放什么 | 由谁执行 |
|---|---|---|---|
| `build` | **全局每模式一次** | 改 config + `make`（重） | 控制进程 preflight |
| `installation` | 每 pack 一次 | 拷贝构建产物（轻） | Worker 首次拿到该 pack |
| `initialization` | 每样本一次 | 清理输出、写输入卡 | Worker 每个样本 |

**实现上零新机器**：`build` 阶段的执行器、stamp/指纹、`reinstall` 控制、per-module 日志
与 **LibDeps（D18）完全同构**——同样是"控制进程 preflight 里全局装一次"。直接复用其安装引擎，
`build.path` 下同样写 `.jarvis_install_stamp.json`，同样受 `jarvis_install.json` 的
`reinstall` 开关管辖。

**新 token `${build_dir}`**：展开为该模式的 `build.path`，供 `installation` 引用产物。

### 5.4.3 备选：不加 `build`，直接用 LibDeps

同样的效果今天就能表达——把每个模式的构建声明成一个 LibDeps 模块，
`installation` 里 `cp -R ${LibDeps:micrOMEGAs_Vector}/. ${path}`。**零新增代码**。

代价是模式定义被拆到两个顶层块、名字要人工保持同步。`modes` 存在的意义本就是
"把一个包的多种用法收拢在一处"，所以**推荐 `build`**；若维护者倾向零新键，
则本节降级为一条文档约定（在 skill 里写清这个配方即可）。

## 5.5 更正 §6.1：Prospino 类软件下，「共享池 + 取用时重建」才是对的

> *"有个程序包叫 Prospino，算 SUSY 过程的截面。它可以算很多过程，但每个过程都需要改一下
> make 的 config 文件之后重新 make 才能切换过去。重新 make 比较快，但每次切换都要重新 make。"*
> —— 维护者，2026-08-01

**先承认错误**：§6.1 断言"每模式独立 pack 不是权衡，现有并发模型已把答案定死"，
这个论证是**过度推广**的。

并发论据只能排除**"所有模式共用同一个 pack 目录"**，
排除不了**"共用一个 pack 池、每个 pack 在取用时被重建成所需模式"**：

```
Prospino 池: [001, 002, 003, 004]
  样本 S 同层需要 ng 与 ns 两个模式：
    ng → 取到 001 → 001 当前是 ns？重建成 ng（几秒）→ 运行
    ns → 取到 002 → 002 当前是 ns？直接运行
  两者在不同目录里并发，无冲突。
```

维护者最初那句"每次拿到 calculator 时判断一下 mode，不是就用 installation 改过去"，
描述的正是这个方案——**对 Prospino 这类软件它是正确的**，我不该一口否掉。

### 5.5.1 两种策略各有其适用区间

| | `per_mode`（§3 原设计） | `shared`（本节） |
|---|---|---|
| 池与目录 | 每模式一个池，`模式数 × pack 数` 个目录 | **一个池，`pack 数` 个目录** |
| 构建次数 | 每目录一次，之后永不重建 | 每次 pack **换模式**时重建一次 |
| 适用 | 构建**贵**、模式**少**（MadGraph、micrOMEGAs：分钟级编译、2–3 个模式） | 构建**便宜**、模式**多**（Prospino：秒级重编、十余个过程） |
| 代价形态 | 磁盘 ∝ 模式数 | 时间 ∝ 模式切换次数 |

Prospino 若用 `per_mode`：12 个过程 × 4 槽 = **48 份拷贝**，磁盘吃紧且首轮 48 次编译；
用 `shared`：**4 份拷贝**，代价是切换时几秒重编——显然后者对。

MadGraph 若用 `shared`：每次 ProcA/ProcB 交替都重新生成进程，最坏每样本一次——灾难。

**所以这是一个真实的、有物理含义的权衡，必须让卡片来声明**：

```yaml
- name: Prospino
  mode_packs: shared          # per_mode（默认） | shared
```

**默认取 `per_mode`**，理由是两种误配的后果不对称：`per_mode` 用错只是多占磁盘、
且是一次性成本；`shared` 用错是**扫描全程持续重建**。让代价可预测的那个当默认。

### 5.5.2 `shared` 的实现语义

1. **池**：`calc:free:<Module>`（不含模式名），pack 目录 `…/<Module>/@PackID`。
2. **stamp 增加 `mode` 字段**：记录该 pack 当前构建成了哪个模式。
3. **取用时**：`ensure_shadow_installed` 增加一步判断——
   `stamp.mode != 需要的模式` **或** 指纹/epoch 不匹配 → 执行该模式的 `installation`
   （即那次重新 make），成功后更新 stamp。
4. **写前清 stamp（关键）**：重建**开始前**先删除 stamp，成功后再写。
   否则重建中途崩溃会留下"stamp 说是 A、目录其实是半个 B"的状态，
   下次取用 A 时会跳过重建、直接跑一个坏掉的构建产物。这是标准的 write-ahead 顺序，
   必须写进实现契约。
5. **模式转换在持槽期间进行**，天然互斥，无需额外加锁。

### 5.5.3 `shared` 的硬性约束：池大小 ≥ 单样本内并发模式数

同层模式在**一个样本内并发**，所以该样本会**同时**持有多个 pack。
若 `Pools[<Module>]` 小于单个样本内并发使用的模式数，第二次 acquire 会一直等到超时，
样本以 `timed out acquiring calculator slot` 失败——**每个样本都会失败**。

因此装载期必须校验：`mode_packs: shared` 时，
`Pools[<Module>]`（或 `make_paraller`）**≥ 任一层内该模块被并发使用的模式数**，
否则报错（建议 `JV2-MOD-006`），并在消息里给出应设的最小值。

这是 `shared` 相比 `per_mode` 多出来的唯一一个坑，但它是可静态计算、可提前拦截的。

## 5.6 收窄适用范围：multimode 只为「互斥构建」而存在（维护者观察，2026-08-01）

> *"iDM 中的 micrOMEGAs 那种情况也存在，但它好像无需使用 multimode 就可以实现。"*

**这个观察是对的，而且它把 multimode 的存在理由收窄成了一句话。** 核对 iDM 卡片：

```yaml
# 两个"模式"其实是同一个包的两个子目录，彼此可共存
installation: ["cp -R ${source}/. ${path}", "cd ${path}/vector-iDM && make ..."]
installation: ["cp -R ${source}/. ${path}", "cd axial-iDM && make ..."]
```

`vector-iDM/` 与 `axial-iDM/` 是并列子目录，**一份安装里可以同时build 好两个**，
选哪个只是"跑哪个二进制"——那就是普通的 `execution.commands` 差异，
**声明两个普通模块即可，完全不需要 modes**。

### 5.6.1 判据

| 包 | 多种用法能否共存于一份安装 | 需要 multimode 吗 |
|---|---|---|
| micrOMEGAs（vector / axial 子目录） | ✅ 可共存 | ❌ **不需要**——两个普通模块 |
| MadGraph（ProcA / ProcB 各自 output 目录） | ✅ 可共存 | ❌ **不需要** |
| **Prospino**（`final_state_in` 是同一构建目标的编译期配置） | ❌ **互斥** | ✅ **必须** |

> **multimode 的唯一存在理由：多种用法在一份安装里互斥——即"要用 B 就必须先把 A 拆掉"。**

这条判据要写进 YAML_REFERENCE 与 skill 的开头。否则用户会对本可以用两个普通模块表达的场景
滥用 modes，白白引入池、pack、重建这些复杂度。

**因此 §2 的场景表需要更正**：MadGraph 与 micrOMEGAs 都**不是** multimode 的适用场景，
它们只是"共享 source 的两个模块"。真正的适用场景是 Prospino 这类编译期互斥的软件。

### 5.6.2 收窄之后，`per_mode` 与 `shared` 的取舍变了

既然只剩互斥构建这一类，两种策略的对比要重算：

| | `per_mode` | `shared`（朴素轮换） |
|---|---|---|
| 目录数 | 模式数 × pack 数 | pack 数 |
| **重建次数** | **首轮各一次，之后为零** | **取决于命中率** |

朴素 `shared` 的危险在于**命中率**：若取用是从一个扁平 free list 里 FIFO 取，
拿到的 pack 恰好已是所需模式的概率约为 `1/模式数`。声明 4 个过程时，
**约 75% 的取用会触发一次重建**。即使单次只要几秒，一万个样本 × 每样本数次取用，
累计就是数小时纯编译——而 `per_mode` 是零。

所以"重建很快"**不足以**让朴素 `shared` 成立，真正决定性的是**命中率**。

### 5.6.3 建议方案：带模式亲和的池（affinity pool）

把"一个扁平池"换成**每模式一个 free list + 借用规则**：

```
acquire(Prospino, mode=ng):
  1. calc:free:Prospino:ng 非空  → 直接取，零重建          ← 稳态下走这条
  2. 否则从最长的其他模式 list 借一个 → 重建成 ng → 归还时进 ng 的 list
```

稳态性质：**每个 pack 会自然沉淀到一个模式上**，重建只在"同时需要的模式数超过池容量"时发生。
于是同时拿到两者的好处——磁盘 ∝ **pack 数**（不是模式数 × pack 数），重建 ≈ **仅预热阶段**。

代价是 `redis_queue` 侧要从"一个 list"变成"每模式一个 list + 借用"，复杂度中等；
借用时的竞态最坏只导致一次多余重建，无正确性风险。

### 5.6.4 分阶段建议

鉴于适用面已收窄到 Prospino 这一类，建议：

- **v1 只做 `per_mode`**（零重建、实现最简、行为可预测），先把 modes 的展开、寻址、
  校验这些主体跑通；
- 待真实 Prospino 卡片跑起来、磁盘确实成为问题时，再加 §5.6.3 的亲和池作为 `mode_packs: shared`。

这样 §5.5 的 `mode_packs` 开关**保留在设计里但 v1 只实现一档**，避免过早引入亲和池的复杂度。

## 6. 三个曾经想不清楚的问题，及答案

### 6.1 模式与 PackID 槽位的关系 → ~~必须每模式独立~~ **已被 §5.5 更正**

> ⚠ **本节结论过度推广，见 §5.5**：并发论据只排除了「所有模式共用同一个 pack 目录」，
> 排除不了「共用一个池、取用时把 pack 重建成所需模式」。对 Prospino 这类「改 config 重新
> make、重建很快、过程很多」的软件，`shared` 池才是对的。最终设计为
> `mode_packs: per_mode | shared`，默认 `per_mode`。以下原文保留以记录推理过程。

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
