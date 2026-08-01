# D20 Multimode Calculator —— 实现代码审阅（2026-08-02）

**范围**：工作区未提交改动（14 个文件 +995/−111，新增 `jarvishep2/calculator_modes.py` 384 行、
`tests/test_calculator_modes.py` 522 行）
**方法**：不看 diff 下结论——全部结论由**对运行代码的实测**得出（65 张出厂卡片校验扫描、
真实 Worker 进程端到端跑、fakeredis 多线程争用压测、Jarvis-PLOT 实际出图）
**结论**：架构与设计一致且实现质量高；**1 个必须先修的 HIGH（7 张出厂卡片被挡在门外）**，
2 个 MEDIUM（亲和池在争用下退化、selection 跳过仍触发重建），3 个 LOW。

---

## 1. 实现选择：走了 `shared`，不是设计文档里建议的 `per_mode`

设计文档（`DESIGN_CALCULATOR_MODES_2.0.md` §5.6.4）建议 **v1 只做 `per_mode`**，
实现选择了**只做 `shared`**：一个父级 = 一套物理 `@PackID` 池，模式是逻辑步骤，
取用时按需重建；`@Mode` / `${mode_dir}` 两个 token 被显式拒绝（`JV2-MOD-006`）。

**这个选择是对的**，理由在设计 §5.6 里已经写明：multimode 的唯一存在理由是"多种用法在一份安装里
互斥"（Prospino），而互斥场景正是 `shared` 的适用区间。`per_mode` 会给 12 个过程 × 4 槽造 48 份拷贝。

**但设计文档现在与实现不符**，必须同步（见 §6 LOW-3），否则下一个 coding agent 会照着
`@Mode` / `mode_packs` 这些已被拒绝的键去实现。

## 2. 做对了的部分（实测确认，不是读代码推测）

| 设计要求 | 实测证据 |
|---|---|
| 展开是**装载期宏**，运行时零新增概念 | `core.py:341` 在 `raise_if_errors` **之后**展开；校验永远读 `raw_task_card`；普通模块的 `calc:free:<name>` 一行未改 |
| 写前清 stamp（设计 §5.5.2 第 4 条） | `runtime_preparer.py` 先 `os.unlink(install_stamp_path)` 再跑命令；测试断言 `stamp_missing_during_rebuild == [True]`——**这是本次实现里最容易漏而没漏的一条** |
| 公共 base 只在首次/强制/指纹变更时跑 | 端到端测试 `build.log == ["base","a","b","a"]`：base 一次，模式切换只跑模式那段 |
| 多模式共用一个 `jarvis_install.json` 与一个 epoch | `prepare_install_controls` 按 control_path 去重；`reinstall: true` 只推进一次 epoch（0→1，两个模式都拿到 1） |
| 重启后恢复亲和 | `_shared_pack_modes` 从磁盘 stamp 反推 pack→mode，重新 seed 到各模式 free list |
| flowchart 显示逻辑节点而非物理池 | 实际用 Jarvis-PLOT 渲染出图通过：`Prospino.ng` / `Prospino.ns` 两个节点、两行标签不重叠、新增的 `groups` 键未被渲染器拒绝 |
| 端到端可跑 | `test_spawn_worker_switches_one_shared_runtime_and_prefers_warm_mode` 起真实 Worker 进程 + 真实子进程构建 + TCP fakeredis，两个样本都 `Completed` |

新增测试 19/19 通过。

---

## 3. HIGH-1：7 张出厂卡片现在直接被校验挡死

**实测**（同一批卡片，唯一差别是有没有 `validate_calculator_modes`）：

| 卡片 | 改动前 | 改动后 |
|---|---|---|
| `GMFit/bin/GMFit_Random.yaml` | 0 error | **2 error** |
| `GMFit/bin/GMFit_Validation_CSV.yaml` | 0 error | **2 error** |
| `2HDM-PT/bin/Dynesty_LLPPT.yaml` | 0 error | **1 error** |
| `2HDM-PT/bin/Random_LLPPT.yaml` | 0 error | **1 error** |
| `HinoLLP/bin/SPheno_Calculator_Validated_CSV.yaml` | 0 error | **1 error** |
| `HinoLLP/bin/FlexibleSUSY_Calculator_Validated_CSV.yaml` | 0 error | **1 error** |
| `HinoLLP/bin/SUSYHIT_Calculator_Validated_CSV.yaml` | 0 error | **1 error** |

错误全部是 `JV2-MOD-005: unknown calculator module or mode dependency`，
经 `raise_if_errors` 后 **exit 2，扫描根本起不来**。

**根因**：`validate_calculator_modes` 的已知名字集合只来自 `Calculators.Modules`，
而 `required_modules` 在真实卡片里**合法地跨块引用**三类名字：

| 被误判的名字 | 它其实是什么 | 出现次数 |
|---|---|---|
| `LoopTools` / `HiggsSignals` | **LibDeps.Modules**（D18 刚实现的共享依赖） | 4 |
| `BuildBSMPTInput` | **Operas.Modules** | 2 |
| `Parameters` | V2 **保留伪模块**——`flowchart.py:511/572/660` 自己就在生成 `required_modules: ["Parameters"]` | 3 |

第三类尤其说明问题：**V2 自己产生的名字被 V2 自己的新校验判为非法**。
（本次审阅渲染出的流程图里，`Parameters` 就是左侧那个节点。）

`resolve_module_layers` 对未知依赖名是**容忍**的（当作排序提示），所以这些卡片一直跑得好好的。

**修法（二选一，建议前者）**：
1. `JV2-MOD-005` **只对带点号的名字**触发，且仅当 `Parent` 确实是已声明的多模式 calculator、
   而 `mode` 不在其模式表里时报错。裸名一律放行——裸名本来就可能指向 LibDeps / Operas。
2. 或把已知集合扩成 `Calculators ∪ LibDeps.Modules ∪ Operas.Modules ∪ {"Parameters"}`。

**验收**：65 张出厂卡片的 `JV2-MOD-*` 错误数回到 0；同时保留
"`Prospino.typo` 报错并给 did-you-mean" 的能力（该用例已有测试）。

---

## 4. MEDIUM-2：亲和池在争用下退化成朴素 shared（实测）

设计 §5.6.3 承诺"稳态下每个 pack 沉淀到一个模式，重建只在同时需要的模式数超过池容量时发生"。
**实测不成立**。用真实的 `acquire_shared_calc` / `release_shared_calc` +
真实的 `Worker._run_shared_mode_group` 贪心选择（只把 calculator 执行换成 sleep），
多线程模拟 Worker 争用：

| 场景 | 重建率 | 对照 |
|---|---|---|
| 3 模式 / **3 pack** / 3 Worker | **63–67%** | 朴素 shared 的理论值就是 67% —— **亲和收益为零** |
| 3 模式 / 4 pack / 3 Worker | 21% | |
| 3 模式 / 5 pack / 3 Worker | 8% | |
| 3 模式 / **6 pack** / 3 Worker | **0%** | |
| 3 模式 / 3 pack / **1 Worker** | **0%** | 无争用时算法本身是对的 |
| 8 模式 / 4 pack / 4 Worker（Prospino 形态） | **86%** | 朴素值 88% |

**两条独立原因**：

1. **借用没有任何等待**。`acquire_shared_calc` 的第一轮非阻塞扫描里，
   目标模式空了就**立刻**借别的模式的 pack 去重建，而不是等一下自己那个热 pack 回来。
2. **阻塞回退把偏好抹平了**。全忙时走 `_blpop_many([目标, unassigned, 其他模式…])`，
   BLPOP 在阻塞期间是**谁先被 push 谁赢**，键的顺序只在调用瞬间有意义。
   于是恰恰在最需要亲和的满载时刻，偏好完全失效。

**修法 A（推荐，已实测）**：借用前先对**目标模式自己的 free list** 做一段有界等待。
实测（重建成本按 4× 单次运行时间建模）：

| 3 模式 / 3 pack / 3 Worker | 重建率 | 墙钟 |
|---|---|---|
| 现实现（立即借用） | 67% | 4.74 s |
| 等待 ≤ 1× 运行时长 | 57% | 5.17 s |
| **等待 ≤ 3× 运行时长** | **0%** | **1.85 s（快 2.6×）** |

**修法 B（必须做，无论 A 做不做）**：把池的下限写进文档并在装载期给出提示。
实测出的规律很干净：**`Pools[parent] ≳ 模式数 + Worker 数` 时重建率归零**；
`= 模式数` 时退化到朴素值。目前 `docs/task-card-schema.md` 对此只字未提，
而默认池大小取 `make_paraller`，用户很容易配成"正好等于模式数"这个最差点。

**边界（同样实测）**：模式数 > pack 数时（8 模式 4 pack），等待策略把重建率从 88% 压到 58%，
**但墙钟反而变慢**（7.82 s → 8.92 s）——因为 8 个模式根本塞不进 4 个 pack。
所以文档要直说：**pack 数少于模式数时，抖动不可避免**，这是 shared 的结构性代价，
不是可以靠调度绕过的。

---

## 5. MEDIUM-3：被 `selection` 跳过的模式，仍然会触发一次完整重建

调用链（`worker.py:302-321` → `Module/calculator.py:565-567`）：

```
_run_calculator_step
  ├─ acquire_shared_calc(...)          ← 占住一个物理 pack
  ├─ module.prepare_runtime(...)       ← ensure_shadow_installed → 可能整套 make 重建
  └─ module.execute(...)               ← 到这里才 _should_run()，selection 假则直接 return {}
```

`prepare_runtime` 全程不看 `selection`。对**普通 calculator** 这只是每个 pack 一生浪费一次安装；
对 **shared 模式**是**每次取用都可能浪费一次重建**，而且更糟的是：
重建成功后 pack 的亲和标签被写成了一个**根本没运行过的模式**，直接污染其他 Worker 的命中率。

考虑到"按参数点决定算不算某个过程"正是 multimode 最自然的搭配用法，这个组合会经常被踩到。

**修法**：把 `_should_run` 提到 acquire 之前——在 `_run_calculator_step` 开头判断，
或在 `_run_shared_mode_group` 组装 `pending` 时就把跳过的模式滤掉（后者更好，
连贪心排序都不用把它算进去）。

---

## 6. LOW

**LOW-1：`Utils` 诊断措辞被回退。** `cbb629f`（上一个提交）刚把
*"Remove top-level Utils: it is not supported by V2."* 这句去掉——因为 D18.6 定的调子是
"弃用指引"而非"V2 缺功能"。工作区把这句加回来了。看起来是合并时的误带，建议还原成 `cbb629f` 的措辞。

**LOW-2：裸名自引用会把自己也展开进去。** 模式 `P.ns` 写 `required_modules: ["P"]`，
展开后得到 `['P.ng', 'P.ns']`——**依赖里含自己**。实测 `resolve_module_layers` 容忍自环
（`P.ns` 落到 layer 1，行为符合用户本意），所以今天无害；但展开时应把自身名字滤掉，
免得将来引入环检测后变成假阳性。（顺带记录：真正的兄弟模式互环今天被静默压平到同一层，
这是 `resolve_module_layers` 的既有行为，普通模块也一样，**不是 D20 引入的**。）

**LOW-3：设计文档与实现已经不一致。** `DESIGN_CALCULATOR_MODES_2.0.md` 仍在讲
"v1 只做 `per_mode`"、`@Mode` token、`mode_packs: per_mode | shared` 开关，
而实现是 shared-only 且**显式拒绝** `@Mode`。文档需要按实现重写（§3–§5 大改），
并把本文实测到的池大小规律写进 §5.6。否则下一个 agent 会照着废弃的 spec 干活。

---

## 7. 建议的工作包

| WP | 内容 | 优先级 | 验收 |
|---|---|---|---|
| **D20.5** | `JV2-MOD-005` 只对"父级已知的点号名"触发（或扩展已知集合到 LibDeps/Operas/`Parameters`） | **高（阻塞发布）** | 65 张出厂卡片 `JV2-MOD-*` 错误数 = 0；`Prospino.typo` 仍报错并给 did-you-mean |
| D20.6 | 借用前对目标模式做有界等待；池下限写进文档 + 装载期提示 | 中 | 3 模式/3 pack/3 Worker 的重建率从 ~65% 降到 ~0%；文档写明 `Pools ≳ 模式数 + Worker 数`，以及"pack < 模式数时抖动不可避免" |
| D20.7 | `selection` 判定提前到 acquire 之前 | 中 | 被跳过的模式不占 pack、不触发重建、不改写亲和标签 |
| D20.8 | 还原 `Utils` 措辞；展开时滤掉自引用；按实现重写 D20 设计文档 | 低 | 文档与代码一致 |

---

## 8. 一句话总结

**主体实现是扎实的**——展开边界、写前清 stamp、共享 epoch、亲和恢复、真实端到端测试都到位，
这些恰恰是最容易做错的地方。**发布前必须修的只有一个**：新校验把 `required_modules`
的跨块引用（LibDeps / Operas / `Parameters`）当成了非法，7 张出厂卡片因此起不来。
另外两个 MEDIUM 不影响正确性，只影响 multimode 的性能承诺能不能兑现——而那正是这个功能的全部意义。
