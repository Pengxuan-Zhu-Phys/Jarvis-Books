# DESIGN — `Sampling.Mapper`（u → 物理参数的显式映射，V2，D22）

**Status**: **已实现**。YAML：`Sampling.Mapper` 为 **`{name, expression}` 列表**（2026-08-05 起；此前短时的扁平 name→expr 对象因对 JSON Schema 不友好已废弃）。
**Scope**: **V2-only**。V1 冻结在 1.7.4，非 bug 不加新功能，本设计永不回移。
**维护者提问（2026-08-04）**：

> 1. 这个 mapper 做成依赖表达式系统怎样？
> 2. 支持隐函数表达，比如 `x = sin(t)`, `y = cos(t)` 的表达的话，该如何做？将 variable
>    设置成 `t` 然后将 `x`, `y` 设置成 `{observable}`，还是支持一个 `v_coords` 系统，
>    `x`, `y` 是 parameter？

**证据基础**：本文全部现状结论来自对运行代码的实测——真实 `Jarvis2 validate` 调用、
真实 mapper 调用、真实表达式编译、2000 次随机等价性对拍、20 万次调用计时——不是读 diff。
测量脚本：`scratchpad/mapper_probe.py`、`scratchpad/mapper_design_probe.py`。

**相关设计**：[`DESIGN_RESUME_2.0.md`](DESIGN_RESUME_2.0.md)（D21 重放不变量，本设计的最强约束
来源）、[`DESIGN_SAMPLERS_2.0.md`](DESIGN_SAMPLERS_2.0.md)、
[`DESIGN_PHYSICS_PLOT_SCENES_2.0.md`](DESIGN_PHYSICS_PLOT_SCENES_2.0.md)（D15.5–7 轴选择）、
[`YAML_REFERENCE_2.0.md`](YAML_REFERENCE_2.0.md) §6/§7/§10。

---

## 0. 结论摘要

| 问题 | 结论 |
|---|---|
| **Q1** mapper 依赖表达式系统？ | **做。**收益明确（零新语法、编译一次缓存、与 `selection`/`LogLikelihood` 同一套词汇），但必须带三条硬约束：**命名空间封闭**（禁止看见 observable / 随机数 / I/O）、**全系统只能有一个实现**、**表达式层叠加在 distribution 层之上而不取代它**。 |
| **Q2** `x=sin(t), y=cos(t)` 怎么做？ | **两个方案都不选。**不把 `x,y` 做成 observable（绘图轴、selection 时序、DATABASE 语义三处都会错），也不引入 `v_coords`（那是 D21 刚拆掉的"第二套坐标真相"）。选**第三条路**：`u` 仍是唯一被持久化的坐标，`Sampling.Mapper` 输出**任意元数的具名物理参数**，`x,y` 进 `Sample.params`。 |
| 新增 YAML | `Sampling.Mapper`：`[{name, expression}, …]` 列表（schema 键固定，用户符号不进 property 名），`Sampling` 下唯一新键。 |
| 必须告警 | 派生维度 > 采样维度 且 Method 是统计型采样器（Dynesty / MultiNest / MCMC 族）时，**装载期 WARNING**：先验活在 `Variables` 空间，样本落在派生空间的零测度流形上。 |
| 必须一并修 | `YAML_REFERENCE §7` 记录了一个**代码已拒绝**的顶层 `Mapper` 块；`worker_config.py:25` 留着读它的**死分支**；`FlatUMapper` 从 YAML **已不可达**。 |

---

## 1. 现状实测

### 1.1 `Mapper` 今天在两个位置都被拒绝

```
$ Jarvis2 validate <card with top-level Mapper>
  [error] JV2-SCH-001  $
          Additional properties are not allowed ('Mapper' was unexpected)
          suggestion: Remove top-level Mapper: it is not a V2 task-card interface.

$ Jarvis2 validate <card with Sampling.Mapper>
  [error] JV2-SCH-001  $.Sampling
          Additional properties are not allowed ('Mapper' was unexpected)
          suggestion: Remove top-level Mapper: it is not a V2 task-card interface.
```

两处都是硬错误。**所以这次是从零定形，没有兼容包袱**——不需要为任何已发布的写法留后门。

> **顺带发现（LOW）**：第二条诊断说的是"Remove **top-level** Mapper"，但键在 `$.Sampling`。
> 根因是 `task_schema.py:159 _root_key_suggestion(key)` 只按**叶子名**查
> `_REMOVED_ROOT_KEYS`，不看路径。`Sampling.Mapper` 合法化之后这条路径的误报自然消失，
> 但同样的错位对嵌套的 `Likelihood` / `Utils` / `project_name` 仍在。见 D22.7。

### 1.2 代码里还留着一个读不到的死分支

```python
# jarvishep2/worker_config.py:24-35
def _default_mapper(cfg: Mapping[str, Any]) -> dict[str, Any]:
    mapper = cfg.get("Mapper")          # ← schema 早已拒绝，永远取不到
    if isinstance(mapper, Mapping):
        return dict(mapper)
    ...
```

实际生效的只有后面三条自动推导：`Method: CSV` → `{"type": "none"}`；有 `Sampling.Variables`
→ `{"type": "distribution", ...}`；否则 → `{"type": "identity", "keys": ["x","y"]}`。

### 1.3 `FlatUMapper` 从 YAML 已不可达

`build_mapper` 的兜底分支返回 `FlatUMapper`，但 `_default_mapper` 只会产出
`none` / `distribution` / `identity` 三种 type，而唯一能显式写出 `flat` 的入口（顶层
`Mapper`）已被 schema 拒绝。实测 `grep '"mapper"'`：整个仓库只有 `tests/` 里注入
`{"type": "flat", ...}`。**生产路径上是死代码。**

### 1.4 u→x 今天有**两套独立实现**（本设计最重要的现状事实）

| | 位置 | 何时跑 | 用途 |
|---|---|---|---|
| Worker 侧 | `mapper.py:35 DistributionUMapper.map` | 每个 Sample 被 Worker 取走后 | 写进 `Sample.params` / `observables` → DATABASE |
| 控制侧 | `Sampling/sampling_utils.py:59 map_u_to_physical` | 每次 `propose_next()` | 喂给 `evaluate_selection` 做接受/拒绝 |

两者逻辑逐字相同（都调 `Variable.map_standard_random_to_distribution`），实测对同一个
`u` 给出 bit 相同的结果：

```
worker : {'t': 2.32477856359, 'm': 173.78008287493753}
control: {'t': 2.32477856359, 'm': 173.78008287493753}
identical: True
```

控制侧的调用点有 11 处，分布在 6 个 sampler（`randoms.py` ×2、`grid.py`、`bridson.py`、
`adaptive_bridson.py` ×5、`mcmc_sampler.py`、`dynesty_sampler.py`）。

> **这是 `Sampling.Mapper` 设计的第一约束**：任何只改一侧的 mapper，都会让
> `selection` 看到的物理点和 Worker 实际计算的物理点不是同一组。

### 1.5 `selection` 排在 uuid 分配之前，所以 mapper 在 D21 的重放链上

```python
# jarvishep2/Sampling/bridson.py:195-206
physical = map_row_to_physical(row, self.vars)        # ← 控制侧 mapper
if self._selectionexp and not evaluate_selection(...):
    continue                                          # ← 拒绝，不占 accepted_index
accepted_index = self._accepted_index
self._accepted_index += 1
sample.uuid = self._uuid_for_accepted_index(accepted_index)   # uuid = f(index)
```

而 D21 的续跑重放正是走这条路径：

```python
# jarvishep2/Sampling/fixed_set_sampler.py:204-212
while self._accepted_index < target:
    if self.propose_next() is None:      # ← 重放时重新跑 mapper + selection
        break
```

**推论（本设计最强的约束）**：`u → 物理参数` 只要不是纯函数，续跑重放时 `selection` 的
接受/拒绝就可能与首跑不同 → `accepted_index` 整体错位 → **uuid 与坐标脱钩**。因为 DATABASE
去重按 uuid、uuid 又只是 index 的函数，结果是"uuid 相同但物理点不同"的静默污染——比 D21
修掉的重复写盘更难查。

### 1.6 现有 mapper 的三条隐含契约

1. **一一对应**：`|params| == |Variables|`，名字来自 `Variables[].name`。
2. **按位置**：`u[i]` 恒定映射到第 `i` 个变量（构造顺序就是稳定的 u-index ↔ 变量绑定）。
3. **多余的 u 被静默丢弃**：实测 `|u|=3` 喂给 2 变量的 mapper 返回 2 个 key，第三维无声消失
   （`mapper.py:37` 只检查 `len(coords) < len(self._variables)`）。

第 3 条在引入 Mapper 后是个隐患：它会掩盖"采样维度和声明维度不一致"这类错误。本设计把它
收紧为等长检查（见 §7 M5）。

### 1.7 下游对"谁是物理参数"的依赖（实测）

| 下游 | 读什么 | 后果 |
|---|---|---|
| 绘图轴 | `plot_scene.py:81 _variable_names_from_config` → `Sampling.Variables[].name` | 只声明 `t` 时，自动出图会拿 `t` 当 x 轴 |
| DATABASE 列 | `Sample.observables`（`bind_params` 用 mapper 结果播种） | mapper 输出自动成为归档列 |
| 采样维度 | `core.py:686` / 各 sampler `load_variables(config)` → `ndim = len(variables)` | 派生参数**不**影响 `ndim` |
| check-modules | `testing/check_modules.py:136` 用变量名当 u 坐标列名 | 派生参数不参与 u 列 |

---

## 2. 回答问题 1：mapper 做成依赖表达式系统

### 2.1 结论：做

理由三条，都不是审美问题：

1. **零新语法**。物理学家已经在 `Sampling.selection`、`Sampling.LogLikelihood`、
   `Operas.Modules[].input.expression`、`AdaptiveBridson.target_expression` 里写同一套
   sympy 表达式（38 个 V1 轻量函数 + 5 个常量）。mapper 用它，学习成本为零。
2. **机制现成**。`ExpressionContext` 按 `(表达式文本, 排序后的符号集)` 缓存
   `CompiledExpression`，sympify + lambdify 每种表达式每进程只付一次。
3. **成本可忽略**（实测，§9）：两条表达式 **0.70 µs/sample**，比现有的 distribution 映射
   （0.92 µs/sample）还便宜，而一个物理点通常是秒级。

### 2.2 硬约束一：mapper 的命名空间必须**封闭**

mapper 表达式**可以**看见：

- `Sampling.Variables[].name`（被采样的变量）
- 在它之前定义的派生参数（DAG 序，§4.3）
- `EXPRESSION_CONSTANTS`：`Pi` / `pi` / `PI` / `E` / `Inf`
- `NUMERIC_MODULES` 的 38 个纯数学函数

mapper 表达式**绝不能**看见：

- 任何 **observable**（calculator / Operas 的输出）
- `uuid` / `sample_index` / worker id / 墙钟时间 / 任何 RNG / 任何 I/O

这不是洁癖，是 §1.5 的直接后果：`u → params` 必须是**纯函数**，否则 D21 续跑静默产出与
未中断运行不同的物理。另外从结构上说，observable 在 mapper 之后才存在
（`worker.py:811 bind_params` 早于 `_run_layer`），想引用也引用不到——但用户会**写得出来**，
所以必须在装载期挡住而不是运行期报 `MissingExpressionVariablesError`。

### 2.3 封闭性可以**静态强制**，机制已经存在（实测）

`ExpressionContext.compile` 返回的 `CompiledExpression.variable_names` 就是解析后的
`free_symbols`。未声明的符号会原样留在里面：

```
expr 'sin(t) + q'  declared symbols=('t',)  ->  free: ('q', 't')     ← q 被抓到
expr 'sin(t)*PI'   declared symbols=('t',)  ->  free: ('t',)         ← 常量被替换掉
expr 'sin(t)+cos(t)' declared symbols=('t',) -> free: ('t',)         ← 函数不算自由符号
```

所以封闭性检查就是一句：

```
free_symbols(expr)  ⊆  {采样变量}  ∪  {已定义的派生参数}
```

**装载期完成，零运行时开销，且是完全的**（不存在漏网的动态引用，因为 sympy 解析是静态的）。

### 2.4 硬约束二：全系统只能有**一个** mapper 实现

§1.4 已经有两套；再加一层表达式而只改一侧，等于把"两处必须保持一致"升级成"两处必须
保持一致，而且其中一处会跑用户任意表达式"。本设计要求把控制侧和 Worker 侧收敛到同一个
`MapperPipeline`（§6.1）——这是**先决条件**，不是可选的清理。

### 2.5 硬约束三：表达式层**叠加**在 distribution 层之上，不取代它

```
u ──(distribution: Flat/Log/Normal/Beta/…)──> 采样变量 ──(expression)──> 派生物理参数
   └──────────── 先验语义（逆 CDF）─────────┘   └───── 重参数化（无先验语义）─────┘
```

`Flat` / `Log` / `Normal` 承载的是**先验**（nested / MCMC 的 prior transform 直接依赖它）；
表达式承载的是**重参数化**。把两者合并成"随便写个表达式"会丢掉先验语义，对做证据计算的
用户是灾难。所以 `Variables[].distribution` **保持原样不动**，`Mapper` 只在其后追加一层。

### 2.6 v1 **不**接入 Operas 函数

Operas 函数（`interp1.*`、`dmddxe.*` 等）在 `selection` 里是可用的，但 mapper v1 **不**开放：

- mapper 在**控制进程**也要跑（§1.4），而控制进程目前只在 `selection` 用到 Operas 时才
  懒加载（`sampling_utils.py:36-47`）；把 mapper 挂上去等于给每次 proposal 增加一个
  Operas 加载面。
- 更重要的是**确定性**：Operas 函数可能从磁盘读插值表，首跑和续跑之间表变了，就违反 §2.2。
  Operas 输出走 observable 通道时这个风险是可接受的（结果落 DATABASE，可核对）；走 mapper
  时它会污染 uuid ↔ 坐标的绑定。

需要 Opera 计算量的用户用 `Operas` 块（产出 observable）。放开 Operas-in-mapper 是 v2 的
可加性扩展（不变量 1：附加键），需要先给 Opera 函数一个"纯函数"声明位。

---

## 3. 回答问题 2：`x = sin(t), y = cos(t)`

### 3.1 三个候选

| | 方案 | 形态 |
|---|---|---|
| **A** | `Variables: [t]`，`x,y` 做成 **observable** | 写个 Opera 算 sin/cos，输出到 observables |
| **B** | `v_coords` 体系 | 在 `u` 之外再引入一层坐标数组，`x,y` 是它的分量 |
| **C** | **具名派生参数**（本设计选择） | `u` 仍是唯一被持久化的坐标，mapper 输出任意元数的具名 param |

### 3.2 为什么不选 A（`x,y` 当 observable）

A **今天就能跑**——但分类是错的，而且有三处实测后果：

1. **绘图轴会挑错**。`plot_scene.py:120 resolve_plot_axis_keys` 的优先级是
   `levelset.variable_names` → `Sampling.Variables` 顺序 → 归档列。只声明 `t` 时，
   自动生成的散点/levelset 会拿 `t` 当 x 轴——那个没有物理意义的参数化坐标。用户每次
   都得手改 plot YAML，这正是 D15.5 要消灭的东西。
2. **`selection` 用不上**。`selection` 在**控制进程**、在 Opera 跑之前求值（§1.5）。
   observable 那时还不存在，所以 `selection: "x**2 + y**2 > 0.5"` 写不出来。
3. **DATABASE 语义错位**。`params` 与 `observables` 是 `Sample` 上两个不同的字典
   （`sample.py:125,127`）。把物理参数塞进 observable，等于对外宣称"这个扫描只有一个
   物理参数 `t`"，后续 `Jarvis2 analyze`（D15.2）的 corner 图、边缘化、best-fit 全都
   会照这个错误的参数集来做。

### 3.3 为什么不选 B（`v_coords`）

因为 `v_coords` 是**第二套需要被持久化并与 `u` 对账的坐标真相**——正是 D21 花一整轮
拆掉的东西（`completed_uuids` / `acked_uuids` 都是这么错的）。现在整个续跑机制建立在

> **`u` 是唯一坐标，其余全是派生**

之上：checkpoint 只存 generator（~3 KB），DATABASE 只存结果，重放靠 `u` 的确定性再生。
引入 `v_coords` 立刻要回答一串新问题：它进不进 checkpoint？崩溃时它和 `u` 对不上怎么办？
重放该信谁？——每一个都是 D21 已经付过学费的问题。

**`v_coords` 提供的能力，C 方案用零新状态就给了**：用户想要的是"`x,y` 是参数"，不是
"多一个坐标数组"。

### 3.4 选 C：`u` 仍是唯一坐标，mapper 输出任意元数的具名参数

```yaml
Sampling:
  Method: Bridson
  Variables:                      # 采样空间：1 维，u ∈ [0,1]^1
    - name: t
      distribution:
        type: Flat
        parameters: {min: 0.0, max: 6.283185307, length: 1.0}
  Bounds:
    Radius: 0.01
    MaxAttempt: 30
  Mapper:                         # 纯函数：采样变量 → 物理参数（列表，固定键）
    - name: x
      expression: "cos(t)"
    - name: y
      expression: "sin(t)"
  selection: "y > 0"              # 现在写得出来了
```

得到的性质：

- `t`、`x`、`y` **全部**进 `Sample.params` 和 DATABASE（采样变量也保留，见 §4.4）；
- `selection` 在控制侧看得见 `x,y`（同一个 pipeline，§6.2）；
- 绘图轴取得到 `x,y`（§4.7）；
- 采样器只知道 1 维的 `t`——Bridson 的 `length`、dynesty 的 `ndim`、Grid 的 `num`
  全部不受影响；
- checkpoint 一个字节都不变：仍然只有 generator 状态。

### 3.5 先验与 Jacobian：必须告警，不能静默

**采样一条曲线 ≠ 采样一个区域。**先验活在 `t` 上，不在 `(x,y)` 上。

- 对 **Random / Grid / Bridson / AdaptiveBridson**（覆盖式扫描）：完全没问题，用户要的
  就是"沿这条曲线扫"。
- 对 **Dynesty / MultiNest / MCMC 族**（统计推断）：如果用户以为自己在 `(x,y)` 平面上放了
  先验，结论是错的——缺的是 Jacobian。维度扩张时更极端：样本落在派生空间的**零测度流形**上。

这不能禁止（有人确实要在流形上做推断，并自己在 `LogLikelihood` 里补 Jacobian），但**不能
静默**。装载期规则：

| 条件 | 行为 |
|---|---|
| `#Mapper 键 > #Variables` 且 Method ∈ {Dynesty, MultiNest, MCMC, AMMCMC, AM, DRAM, EnsembleMCMC, Ensemble, DEMCMC, PT*} | **WARNING `JV2-MAP-050`**：样本落在 `#Variables` 维流形上；先验定义在 `Variables` 空间；若要在派生空间做推断需自行提供 Jacobian |
| `#Mapper 键 ≤ #Variables` 且存在非线性 Mapper 表达式且 Method 为统计型 | **WARNING `JV2-MAP-051`**：这是重参数化视图；先验仍定义在 `Variables` 空间 |
| Method 为覆盖式扫描（Bridson/Random/Grid/AdaptiveBridson/CSV） | 静默通过 |

"跑得很正常、结论悄悄不对"是这个项目一路在消灭的东西，这里不能开口子。

---

## 4. YAML 设计

### 4.1 语法

`Sampling.Mapper` 是 `Sampling` 下的**列表**，每项是固定键的映射
`{name, expression}`（`additionalProperties: false`）。**不要**写成
`x: "cos(t)"` 这种“用户符号当 property 名”的自由对象——那对 JSON Schema 不友好，
会把结构字段与用户输入混在同一层。

```yaml
Sampling:
  Mapper:
    - name: <参数名>
      expression: <表达式字符串>
```

示例：

```yaml
  Mapper:
    - name: x
      expression: "cos(t)"
    - name: y
      expression: "sin(t)"
```

- 列表顺序仅用于绘图轴偏好（§4.7）；求值顺序由 DAG 决定（§4.3），与书写顺序无关。
- `name` / `expression` 均为非空字符串；表达式语法同 `selection` / `LogLikelihood`。
- 内部实现（`MapperSpec.derive_order` 等）可仍用 derive 前缀，那是代码字段名，
  **不出现在 YAML 表面**。
- `Mapper` 缺省时行为与无 Mapper 时完全一致（自动推导 distribution mapper）——
  **纯附加**。
- 旧的扁平 map 形态（`Mapper: {x: "cos(t)"}`）**拒绝**（`JV2-MAP-001` / schema 类型错误）。

**长写法留给以后**（不变量 1，在条目上附加可选键）：

```yaml
  Mapper:
    - name: y
      expression: "sin(t)"
      description: "unit-circle y"   # v2 扩展
      latex: "$y$"                   # 与 D15.7 Variables[].latex 对齐
```

### 4.2 命名空间与可见性

| 组 | 内容 | 来源 |
|---|---|---|
| 采样变量 | `Sampling.Variables[].name` | 始终可见 |
| 已定义派生参数 | DAG 序中排在本表达式之前的 Mapper 名 | §4.3 |
| 常量 | `Pi` `pi` `PI` `E` `Inf` | `inner_func.EXPRESSION_CONSTANTS` |
| 函数 | 38 个 V1 轻量函数 | `inner_func.NUMERIC_MODULES` |
| **不可见** | observable、Operas 函数、`uuid`、`sample_index`、`status`、RNG、I/O | §2.2 / §2.6 |

### 4.3 求值顺序：DAG

派生参数之间**允许互相引用**：

```yaml
    Mapper:
      - name: z
        expression: "x + y"   # 允许，即使写在 x,y 之前
      - name: x
        expression: "cos(t)"
      - name: y
        expression: "sin(t)"
```

装载期对 Mapper 名做依赖图 + 拓扑排序，得到一个固定的求值序列，存进 pipeline spec
（因此控制侧和 Worker 侧的求值顺序必然一致）。存在环 → 硬错误 `JV2-MAP-004`，诊断里
打印出环上的名字。

### 4.4 命名冲突规则

| 冲突 | 行为 | 码 |
|---|---|---|
| Mapper 名 == 某个 `Variables[].name` | **error**（禁止遮蔽采样变量；要改名就改 `Variables`） | `JV2-MAP-003` |
| Mapper 名 == 保留常量（`Pi`/`E`/…） | **error** | `JV2-MAP-005` |
| Mapper 名 ∈ {`uuid`, `sample_index`, `status`, `LogL`} | **error**（DATABASE 保留列） | `JV2-MAP-006` |
| Mapper 名与某个 `Operas.Modules[].output[].name` 相同 | **warning**：Opera 输出稍后经 `merge_observables` 覆盖同名 observable，DATABASE 里存的是 Opera 的值，而 `Sample.params` 里仍是 mapper 的值——两者会不一致 | `JV2-MAP-052` |
| Mapper 名与某个 calculator dump 变量名相同 | 同上（warning） | `JV2-MAP-052` |

**采样变量始终保留在输出里**（`t` 一定会进 params 和 DATABASE）。不提供"丢弃采样变量"的
开关：保留一列的成本是零，丢掉之后不可逆，而且续跑排障和复现都要它。

### 4.5 与各 Method 的关系

| Method | `Sampling.Mapper` | 说明 |
|---|---|---|
| Bridson / Random / Grid / AdaptiveBridson | **支持** | 覆盖式扫描，无先验语义问题 |
| Dynesty / MultiNest / MCMC 族 | **支持 + 告警** | §3.5 |
| **CSV** | **v1 拒绝**（`JV2-MAP-010`） | CSV 卡的参数直接来自列（`build_mapper` 返回 `None`，Worker 走 `adopt_params(opera_params)`），命名空间是列名而非 `Variables`，接上去等于第二条代码路径。理由写进诊断，作为 v2 扩展 |
| `mode: check_modules` | **支持** | u 坐标列仍按 `Variables` 名给（`check_modules.py:136`），派生参数由 pipeline 补齐；列入验收 P5 |

### 4.6 与 `selection` / `LogLikelihood` / `Operas` 的关系

```
u ─┬─> MapperPipeline ─> {t, x, y}  ─┬─> selection（控制进程，决定接受/拒绝 + uuid）
   │                                 └─> Sample.params / observables 播种（Worker）
   └─> （持久化的唯一坐标；checkpoint 只有 generator）
                                          │
                    Calculators / Operas ──┴──> merge_observables ──> LogLikelihood ──> DATABASE
```

- `selection` 的可见符号集从"`Variables` 名"扩展为"`Variables` 名 ∪ Mapper 名"。
- `LogLikelihood` 本来就在 observables 上求值，而 `bind_params` 用 mapper 结果播种
  observables（`sample.py:255-256`），所以 Mapper 出来的参数**自动**对 LogLikelihood 可见，
  无需额外改动。
- `Operas` 不变。

### 4.7 绘图轴选择

`plot_scene.resolve_plot_axis_keys` 的优先级插入一档：

```
levelset.variable_names
  → Sampling.Mapper 列表书写顺序             ← 新增
  → Sampling.Variables 顺序
  → 归档数值列
```

理由：用户既然显式声明了派生物理参数，那就是他要看的物理轴；`t` 是参数化用的内部坐标。
（这是一个判断，维护者可以翻转；D15.7 的 `Scan.plot` 块落地后会给显式控制，届时这条
启发式退为兜底。）

### 4.8 Key Index 增量（`YAML_REFERENCE §3.2`）

| Path | § | Required | Default |
|---|---|---|---|
| `Sampling.Mapper[].name` / `expression` | 6.14 | no | — |

---

## 5. 装载期诊断（新前缀 `JV2-MAP`）

实测：现有前缀为 `ARC BND DEAD ENC ENV LOAD MOD MTH OPR SCH VAR YAML`，`MAP` 未被占用。

| 码 | 级别 | 触发 | suggestion |
|---|---|---|---|
| `JV2-MAP-001` | error | `Mapper` 不是列表 / 条目缺 `name`/`expression` / 误用扁平 map | 给出列表形状：`Mapper: [{name: x, expression: "cos(t)"}]` |
| `JV2-MAP-002` | error | 表达式引用了不在命名空间里的符号（§2.3 子集检查） | 列出未知符号 + 当前可用符号集 |
| `JV2-MAP-003` | error | Mapper 名与 `Variables[].name` 冲突 | 点名冲突的名字 |
| `JV2-MAP-004` | error | Mapper 表达式依赖成环 | 打印环 |
| `JV2-MAP-005` | error | Mapper 名是保留常量 | 列出保留名 |
| `JV2-MAP-006` | error | Mapper 名是 DATABASE 保留列 | 列出保留列 |
| `JV2-MAP-007` | error | 表达式 sympify / lambdify 失败 | 转述 sympy 报错 + 表达式文本 |
| `JV2-MAP-010` | error | `Method: CSV` 同时给了 `Sampling.Mapper` | 说明 v1 不支持及原因 |
| `JV2-MAP-050` | **warning** | 维度扩张 + 统计型采样器 | §3.5 |
| `JV2-MAP-051` | **warning** | 非线性重参数化 + 统计型采样器 | §3.5 |
| `JV2-MAP-052` | **warning** | Mapper 名与 Opera output / calculator dump 名冲突 | 说明覆盖顺序 |

全部在 `Jarvis2 validate` 里就能给出——**不需要连 Redis、不需要起进程**。

---

## 6. 实现设计

### 6.1 单一实现点：`MapperPipeline`

`jarvishep2/mapper.py` 新增（并让现有三个类退居内部实现）：

```python
@dataclass(frozen=True)
class MapperSpec:                       # 纯数据，picklable，无闭包
    variables: tuple[VariableSpec, ...]         # 采样变量（名 + 分布 + 参数）
    derive_order: tuple[str, ...]               # 拓扑排序后的求值序列
    derive_exprs: Mapping[str, str]             # 名 -> 表达式文本
    derive_symbols: Mapping[str, tuple[str, ...]]   # 名 -> 该表达式声明的符号集

class MapperPipeline:
    @classmethod
    def from_config(cls, config) -> "MapperPipeline": ...   # 读 Sampling，做全部 §5 校验
    @classmethod
    def from_spec(cls, spec: MapperSpec) -> "MapperPipeline": ...   # 进程内重建（编译在此发生）

    def map(self, u_coords: np.ndarray) -> dict[str, float]: ...
    @property
    def output_names(self) -> tuple[str, ...]: ...   # 采样变量序 + derive 拓扑序
    @property
    def spec(self) -> MapperSpec: ...
```

`map()` 两步：先按现有 distribution 逻辑得到采样变量值，再按 `derive_order` 依次
`CompiledExpression.evaluate`，结果并入同一个 dict。表达式编译走进程本地
`ExpressionContext`，因此每种表达式每进程只编译一次。

### 6.2 控制侧接入

`sampling_utils.map_u_to_physical(u, vars)` 的 11 个调用点改为
`self._mapper_pipeline.map(u)`。pipeline 在 sampler `set_config` 时由
`MapperPipeline.from_config(self.config)` 构造一次。

保留 `map_u_to_physical` 作为 pipeline 内部的无 Mapper 表达式快路径即可，但**不再有第二个
对外的 u→x 入口**。

### 6.3 Worker 侧接入

`worker_config._default_mapper` 改为产出 `MapperSpec`（picklable dataclass），
`worker.py:155 build_mapper(...)` 改为 `MapperPipeline.from_spec(...)`。
`Sample.bind_params` 完全不用改（它只要一个 `map(u) -> Mapping`）。

### 6.4 Bridson 的 row 路径可以等价重构（实测 bit-exact）

Bridson 走的是 `map_row_to_physical(row)`（先除 `length` 再映射），与
`map_u_to_physical(row_to_u_coords(row))` 是同一件事。实测 2000 组随机 row、
跨 Flat/Log/Normal 三种分布：

```
max |difference| over 2000 random rows: 0.0
```

**bit-exact**。所以 Bridson 可以安全改成 `pipeline.map(row_to_u_coords(row, vars))`，
不需要给 pipeline 开第二个 row 入口。

### 6.5 picklable / checkpoint

`MapperSpec` 是 frozen dataclass + 纯字符串/数字，**没有 lambdify 出来的可调用对象**
（那些在 `from_spec` 时按进程重建）。因此：

- Worker 配置照旧可 pickle 后 spawn；
- sampler checkpoint 里如果携带 spec，也只是几百字节的文本，不违反 D21 的 payload 瘦身约束
  （§13 C1）；
- 更好的做法是 **checkpoint 里根本不存 spec**——它是卡片的函数，续跑时从卡片重建即可。
  这样 checkpoint 大小完全不变。

---

## 7. 不变量

| # | 不变量 | 强制方式 |
|---|---|---|
| **M1** | `u → params` 是纯函数：同一个 `u` 在任何进程、任何时刻、首跑与续跑，给出同一组 params | §2.3 静态封闭性检查 + §2.6 不接 Operas + 无 RNG/I-O 入口 |
| **M2** | 控制侧与 Worker 侧共用同一个 `MapperPipeline` 实现 | §6.1；测试断言两侧对同一 `u` 输出相同 dict |
| **M3** | `u` 是唯一被持久化的坐标；params 永远可由 `u` + 卡片重建 | 不引入 `v_coords`；checkpoint 不存 params |
| **M4** | `Variables[].distribution` 的先验语义不被 mapper 改变 | mapper 只在 distribution 之后追加；prior transform 路径不动 |
| **M5** | `len(u_coords) == len(Variables)` | 把 `mapper.py:37` 的 `<` 检查收紧为 `!=`（§1.6 第 3 条） |
| **M6** | `Sampling.Mapper` 缺省时行为与今天逐位相同 | 回归测试：现有 65 张示例卡的 params 输出不变 |

---

## 8. 与 D21 resume 的相互作用

| D21 机制 | 本设计的影响 |
|---|---|
| I1 DATABASE uuid 是唯一完成真相 | 不变 |
| I2 restore point 不越过在途工作 | 不变 |
| I3 写侧去重 | 不变 |
| `deterministic_sampler_uuid = f(prefix, seed, index)` | 不变——但 **index 依赖 selection，selection 依赖 mapper**，所以 M1 是 D21 正确性的前置条件（§1.5） |
| `advance_to_persisted_prefix` 本地重放 | 重放期间会跑 mapper（每点 ~1.6 µs），对 `checkpoint_heartbeat_sec` 规模的重放量可忽略 |
| checkpoint payload | **不变**（§6.5） |

**新增回归风险点**：用户在两次运行之间**改了 `Sampling.Mapper` 的表达式**然后 `--resume`。
此时 uuid 仍按旧 index 生成，但物理点变了——静默污染。

**处置**：把 `Mapper` 块的规范化文本哈希写进 checkpoint 的卡片指纹，`--resume` 时不一致
则**拒绝续跑**并说明"mapper 已变更，请另起扫描或 `--force-resume`"。这属于 D21 已有的
卡片指纹机制的自然扩展（D22.5）。

---

## 9. 性能预算（实测）

约束沿用维护者对 D21 的定调：**不能影响运行时吞吐与并发**。

| 项 | 实测 |
|---|---|
| 2 条 mapper 表达式求值 | **0.70 µs / sample** |
| 现有 2 变量 distribution 映射（基线） | **0.92 µs / sample** |
| 合计 | **~1.6 µs / sample** |

对比：一个真实物理点通常是 10⁰–10² 秒量级；即使空跑的 Bridson 提案循环，单点成本也在
数十 µs。**mapper 的表达式层在噪声里。**

编译成本：每种表达式每进程 sympify + lambdify 一次（`ExpressionContext` 缓存命中后为零）。
控制进程 1 次 + 每个 Worker 1 次。

---

## 10. 迁移与清理（必须与本设计一起做）

| # | 项 | 动作 |
|---|---|---|
| 1 | `YAML_REFERENCE §7` 记录了一个**已被拒绝**的顶层 `Mapper` 块（含完整 type 表） | 重写为"顶层 `Mapper` 已移除；mapper 自动推导；`Sampling.Mapper` 为 D22 规划接口" |
| 2 | `YAML_REFERENCE §3.3` 的 `Mapper.type` / `Mapper.keys` / `Mapper.variables` 三行 | 删除；§3.2 增加 `Sampling.Mapper.<name>`（扁平 map） |
| 3 | `worker_config.py:25` 读顶层 `Mapper` 的死分支 | 删除 |
| 4 | `FlatUMapper` 从 YAML 不可达（仅 tests 注入） | 标注为内部/测试专用，或随 `MapperPipeline` 一并收编 |
| 5 | `components/umapper.md` 只讲 Worker 侧，未提控制侧 `map_u_to_physical` | 补上双实现现状，实现后改写为单 pipeline |
| 6 | `_root_key_suggestion` 按叶子名匹配导致 `$.Sampling.Mapper` 报 "top-level" | 改为按路径匹配（D22.7） |

> §7 的文档修正**先于**实现落地——文档现在描述的是一个不存在的接口，这本身就是 bug。

---

## 11. 验收门

| # | 门 | 判据 |
|---|---|---|
| **P1** | 无 Mapper 卡片零回归 | 65 张示例卡 `Jarvis2 validate` 通过率不变；同 seed 下 params 输出逐位相同（M6） |
| **P2** | 双侧一致 | 单测：同一 `u` 经控制侧 pipeline 与 Worker 侧 pipeline 得到相同 dict（M2） |
| **P3** | 封闭性 | `Mapper: [{name: x, expression: "sin(t) + LogL"}]` 在 `Jarvis2 validate` 阶段报 `JV2-MAP-002` 并点名 `LogL`；不连 Redis、不起进程 |
| **P4** | 参数方程端到端 | §3.4 的卡片跑 200 点：DATABASE 含 `t,x,y` 三列；`x²+y²=1` 在 1e-12 内；`selection: "y > 0"` 确实只留半圆；自动出图的轴是 `x,y` 不是 `t` |
| **P5** | 续跑等价 | 该卡片跑到中途 SIGINT → `--resume` → 最终 DATABASE 的 `(uuid → (t,x,y))` 映射与未中断运行逐行相同；无重复 uuid |
| **P6** | 指纹保护 | 改掉 `Mapper` 表达式后 `--resume` 被拒绝并给出可执行的说明（§8） |
| **P7** | 告警 | Dynesty + 1 变量 + 2 个 Mapper 键的卡片在 `validate` 阶段给出 `JV2-MAP-050`，且**不**是 error |
| **P8** | 性能 | 带 4 条 Mapper 表达式的 Bridson 卡片，控制侧提案吞吐相对无 mapper 基线下降 < 2%（同机对拍） |

---

## 12. 开发台账（D22）

见 [`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md) §3 的 D22.1–D22.7 行。摘要：

| WP | 内容 | 依赖 |
|---|---|---|
| D22.1 | **文档先行**：修 §7 / §3.3 的顶层 `Mapper` 矛盾（纯文档，可立即做） | — |
| D22.2 | `MapperPipeline` + `MapperSpec`；收编 Worker 侧与控制侧的两套实现（**无新 YAML**，纯重构 + M2/M5/M6 回归） | D22.1 |
| D22.3 | `Sampling.Mapper` 列表 schema（`{name, expression}`）+ `JV2-MAP-0xx` 装载期诊断（§5） | D22.2 |
| D22.4 | 求值接线：`selection` 符号集扩展、`bind_params` 播种、DATABASE 列 | D22.3 |
| D22.5 | checkpoint 卡片指纹纳入 Mapper 文本哈希；`--resume` 变更拒绝（§8） | D22.3 |
| D22.6 | 绘图轴优先级 + skill/参考文档 + 一张参数方程示例卡 | D22.4 |
| D22.7 | LOW：`_root_key_suggestion` 改为按路径匹配 | — |

---

## 13. 明确不做的事

1. **不做 `v_coords`**（§3.3）。
2. **不做 inverse mapper**。`x → u` 的逆映射没有用例（没有任何组件需要它），而且派生映射
   一般不可逆。
3. **不让 mapper 看见 observable**（§2.2）——这是 D21 正确性的前置条件，不是可配置项。
4. **v1 不接 Operas 函数**（§2.6）。
5. **不改 `Variables[].distribution` 的先验语义**（§2.5）。
6. **不回移 V1**。V1 冻结在 1.7.4，非 bug 不加新功能。
7. **不做"丢弃采样变量"开关**（§4.4）。
8. **CSV 方法 v1 不支持 Mapper**（§4.5），拒绝时给出理由。
