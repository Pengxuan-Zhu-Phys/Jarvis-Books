# DESIGN — 断点续跑（Checkpoint / Resume 重做，V2，D21）

**Status**: implemented and acceptance-verified 2026-08-02 (D21.1–D21.10)
**Scope**: V2-only
**Requirement (maintainer, 2026-08-02)**:

> *"只有写入到 HDF5 中的点才算是真实标记为完成，其余 sample 都需要在 resume 之后重新算。
> sampler 也需要恢复到那些正在计算的点没有提交之前的状态，然后把那些点重新算。我们要恢复：
> (1) sampler，即 generator 的状态；(2) 重新开始计算没有写入到 HDF5 的所有的点。"*

**证据基础**：本文所有现状结论均来自对运行代码的实测（真实扫描 + 真实中断 + 真实 resume +
读取真实 checkpoint 与 HDF5），非阅读 diff。

---

## 1. 维护者定下的两条语义

| # | 语义 | 含义 |
|---|---|---|
| **S1** | **只有落进 HDF5 的点才算完成** | 内存里的 sampler 簿记、Redis 计数、archiver 的在途批次，一律不算数 |
| **S2** | **generator 必须回到"那些在算的点尚未提交之前"的状态** | 不是"接着往下发"，而是回到提交点之前，重新发这些点 |

这两条合起来给出一个很强的正确性目标：**resume 之后的运行轨迹，必须与从未中断过的运行等价**。

## 2. 现状：resume 是坏的（实测）

一次 4000 点扫描，跑到 3132 点时 Ctrl+C（interrupt checkpoint 正常写出），随后 `--resume`：

```
中断时  DB: 3132 行 / 3132 unique
resume 后 DB: 7132 行 / 4000 unique / 3132 个 uuid 各重复 2 次
```

**已完成的 3132 个点被整体重算，并以相同 uuid 追加进同一个 HDF5。**
这不是效率问题——任何基于该库做后验/似然聚合的分析都会把这 3132 个点计两次。

### 2.1 根因链（逐环实测）

| 环 | 实测事实 |
|---|---|
| 1 | `mark_completed()` **生产代码从未调用**（仅两个测试调用）。真实 checkpoint 内容：`submitted=3132, completed=0` |
| 2 | 于是 `repropose_unfinished() = submitted − completed` = **全量重投** |
| 3 | `core.save_runtime_checkpoint` 把 `safe_barrier_confirmed=True` **写死**，从不调用已实现的 `safe_barrier_ready()` |
| 4 | 进程模式 archiver 的 `persistence_state()` **按设计返回 `acked_uuids: []`**（`archiver.py:665`），checkpoint 里没有任何落盘真相 |
| 5 | 写入端不去重，给什么写什么 |

### 2.2 为什么长期没被发现

`tests/test_distributed_resume.py::test_kill_and_resume_completes_without_duplicate_uuids`
**自己调用了 `sampler.mark_completed(uuid)`**（第 333 行）。它验证的是库的契约，
不是生产的接线——生产里那个调用点根本不存在。

### 2.3 影响范围

| 采样器族 | 是否受影响 | 原因 |
|---|---|---|
| `FixedSetSampler` → **Random / Grid / Bridson**、**CSVSampler** | ❌ **全部重算 + 重复写入** | `_completed_uuids` 永远为空 |
| `FeedbackSampler` → **AdaptiveBridson** | ✅ 正常 | 从 `hep:feedback` 通道填 `_completed_uuids`（`feedback_sampler.py:159`） |

旗舰采样器恰好是安全的那一个，这解释了为什么没人踩到。

## 3. 决定整个设计的一条事实：uuid 是索引的纯函数

```python
# Sampling/stateless_batch.py:13
def deterministic_sampler_uuid(*, prefix, seed, sample_index) -> str:
    digest = hashlib.sha256(f"{prefix}:{int(seed)}:{int(sample_index)}".encode()).hexdigest()
```

**`uuid = f(prefix, seed, accepted_index)`**，与 Worker 数、执行顺序、时间全部无关。
并且 HDF5 的每条记录里都带 `uuid` 字段（实测记录键：
`['LogL','LogL_Z','bucket_dir','bucket_id','product_list','uuid','x','y','z']`）。

推论：**"哪些索引已落盘"可以在 resume 时被精确重建，不需要任何运行期簿记。**
这让 S1 从"需要维护一套完成台账"降级为"启动时读一列"。

## 4. 三条不变量

### I1 — DATABASE 是完成事实的唯一来源

resume 启动时读 `DATABASE/samples.hdf5` 的 uuid 列，得到 `persisted`。
sampler 内存状态、Redis 计数、archiver ack 集合都只是**缓存**，它们与进程同生共死，不作为真相。

这条直接兑现 S1，并且天然覆盖所有崩溃形态：Ctrl+C、SIGKILL、断电、OOM、机器掉线。
代价是启动时读一列——百万样本量级几百毫秒。

### I2 — restore point 只前进到"已落盘"的边界（滚动安全恢复点）

这是 S2 的工程落地。**不能**指望把 numpy 的随机流"倒带"——它是单向的。
因此改为：**checkpoint 永远不写当前活状态，只写一个滚动的恢复点。**

```
restore_point = (rng_state, index, accepted_index, sampler 私有状态)
                 ↑ 在提出第 B 批之前捕获

只有当第 B 批的全部 uuid 都已落盘，restore_point 才前进到第 B+1 批之前
```

于是 checkpoint 里的 generator 状态**永远处于"在途点尚未提交之前"**，
resume 直接恢复它并重新往下生成，就会**重新提出那些在途点**——正是 S2 要的。

滞后是有界的：最多落后 `_max_inflight` 个样本（默认 `max(batch_size, workers*4)`），
不是落后 30 秒的心跳间隔。

### I3 — 写入端结构性去重

archiver 启动时用 I1 那次读取播种一个 uuid 集合，写入前查一次；已存在则跳过并记一条 debug。
**即使 resume 逻辑再出错，重复记录也不可能产生。** §2 里那 3132 个重复行会被这一条挡死。

## 5. 两类采样器的实现

### 5.1 枚举型（`FixedSetSampler` 家族 + `CSVSampler`）

提议只依赖既往**提议**，不依赖既往**结果**。滚动恢复点按批推进即可：

- `flush_batch()` 之前捕获 `(np.random.get_state(), _index, _accepted_index, ready_queue)` 存为 `pending_restore`
- 收到该批全部 uuid 的落盘确认后，把 `pending_restore` 提升为 `restore_point`
- checkpoint 写 `restore_point` + 该批之后所有**未落盘**批次的批边界（用于精确重放）

resume 后 generator 从 `restore_point` 前向生成，重新产出相同索引 → 相同 uuid → 与未中断运行逐点等价。

### 5.2 反馈自适应型（`FeedbackSampler` → `AdaptiveBridson`）

提议依赖既往**结果**，所以恢复点必须落在**世代屏障**上——而这个屏障它已经有了
（generation barrier）。规则同构，只是粒度更粗：

- 一个世代的全部样本落盘确认后，才把恢复点推进到该世代之后
- resume 回到最后一个**完整落盘**的世代边界，重放该世代之后的全部点

代价是最多重算一个世代；换来的是自适应逻辑的严格可重现。

## 6. Archiver 侧：让落盘确认可跨进程可见

I2 需要"该批是否已落盘"这个信号，而进程模式 archiver 现在返回空 ack 集。改为：

```
写入批次 → HDF5 flush → fsync → SADD hep:archived:<scan> <uuids...>
```

**顺序不可颠倒。** 若在 fsync 之后、SADD 之前崩溃，会出现"盘上有、Redis 没有"，
此时 I1 的落盘对账兜底（那些点会被跳过而非重算）。Redis 只是快路径，不是真相。

同时 `ArchiverProcess.persistence_state()` 改为读这个 Redis 集合，
不再返回 `{"acked_uuids": []}` 的占位。

## 7. resume 流程

```
1. 校验 / 定位 checkpoint            （既有逻辑）
2. persisted ← DATABASE 的 uuid 列    ← I1，唯一真相
3. 恢复 generator 至 restore_point    ← I2，"提交之前"的状态
4. 重建 Redis 池、清空陈旧 task_queue （既有 prepare_resume）
5. archiver 用 persisted 播种去重集   ← I3
6. 前向生成；提交前若 uuid ∈ persisted 则跳过
   （覆盖 checkpoint 与崩溃之间的窗口，以及 fsync/SADD 之间的窗口）
7. 正常跑到 Point number 为止
```

第 6 步是让"心跳滞后"无害的关键：因为 uuid 由索引决定，重放必然产生相同 uuid，
已落盘的直接跳过，未落盘的自然重算。

## 8. 边界情况

1. **失败样本**。物理性失败的点永远不会出现在 HDF5 里，若不处理，resume 会**无限重算它**。
   要求：失败样本必须以带 `status` 的记录落盘（需先核实现状——worker 的 `finally` 分支
   看起来会走归档，但本次实测 `failed=0`，未取得实证）。这是 D21.6。
2. **checkpoint 与崩溃之间的窗口**。由 §7 第 6 步吸收，无需额外机制。
3. **HDF5 陈旧写标志**。实测：SIGKILL 之后该文件**连只读都打不开**
   （`file is already open for write ... may use h5clear`）。resume 前必须检测并 `h5clear`，
   否则第 2 步就失败。
4. **`safe_barrier_confirmed` 写死为真**。改为真调 `safe_barrier_ready(...)`；
   心跳 checkpoint 如实记 `False`。resume 对未确认的 checkpoint 一律走对账（反正总是对账）。
5. **`mark_completed()` 可以删除**。它是这套设计里不再需要的中间概念——完成与否由盘上说了算。

## 9. 进程生命周期：resume 现在根本起不来（前置阻塞）

实测：SIGKILL 控制进程后，`Jarvis-Redis` + 2 个 Worker + Archiver + 2 个 FileOperation
**全部存活无人收尸**，随后三条路全堵：

| 命令 | 实测结果 |
|---|---|
| `Jarvis2 kill R1 -y` | *Refusing unverified runtime: control PID does not match metadata.* |
| `Jarvis2 ps R1` | 同上 |
| `Jarvis2 run … --resume` | *Redis port 6379 is already in use*（孤儿 Redis 占着） |
| `Jarvis2 monitor` | 照常列为"运行中" |

**`kill` 恰恰在唯一需要它的场景下拒绝工作**——控制 PID 对不上 metadata，正是"它已经死了"的定义。
最小修法：

- 控制进程写一个**租约**（runtime.json + Redis TTL）；Worker 已经在往 Redis 写心跳，
  只需反过来读租约，过期即自杀
- **`Jarvis2 kill` 的判断反转**：控制 PID 不存在 ⇒ 这是 stale runtime ⇒ **允许**清理
- **`--resume` 遇到 stale runtime 接管或清理**，而不是报端口占用

另有一条独立观察：机器上累积了 **13 个 ppid=1 的孤儿 `Jarvis2-FileOperation`**，
最老 **2 天 13 小时**，早于本次会话——说明泄漏不只发生在崩溃路径。

## 10. 验收（必须是端到端行为，不是单元桩）

| # | 验收 |
|---|---|
| A1 | 4000 点扫描，中途 Ctrl+C，`--resume` → **DB `unique_uuids == rows`**，且总数 = 4000 |
| A2 | 同上但用 `kill -9` 控制进程 → 清理后可 resume，结果同 A1 |
| A3 | resume 后**重算量 ≤ `max_inflight` + 未落盘批次**，不是全量（记录实际重算数） |
| A4 | 未中断运行 vs 中断+resume：**同 seed 下逐点结果集完全一致**（uuid 集合与数值） |
| A5 | 人为损坏/删除 checkpoint → resume 靠 DATABASE 对账仍能正确续跑（I1 自愈性） |
| A6 | AdaptiveBridson：中断+resume 后世代序列与未中断运行一致 |
| A7 | 现有那个自己调 `mark_completed` 的测试被替换为 A1 形态的端到端断言 |

## 11. 工作包

| WP | 内容 | 依赖 | 优先级 |
|---|---|---|---|
| **D21.1** | I1 落盘对账 + I3 写入端去重 | — | **高**（单独即可消灭重复写入） |
| **D21.2** | Archiver ack 上报 Redis（flush→fsync→SADD）；`persistence_state()` 不再返回空 | — | 高 |
| **D21.3** | I2 滚动恢复点（枚举型采样器） | D21.2 | 高 |
| **D21.4** | 反馈型采样器的世代屏障恢复点 | D21.3 | 中 |
| **D21.5** | `safe_barrier_confirmed` 改为实算；删除 `mark_completed` 死接口 | D21.2 | 中 |
| **D21.6** | 失败样本的落盘语义（避免 resume 死循环） | D21.1 | 中 |
| **D21.7** | 端到端回归测试 A1–A7 | D21.1–D21.4 | 高 |
| **D21.8** | 生命周期：control lease + Worker 自杀、`kill` 判断反转、stale runtime 接管、`h5clear` | — | **高**（不修则 resume 起不来） |

**回滚**：不带 `--resume` 的运行路径完全不受影响；D21.1/D21.3 之前的行为即今天的行为。

## 12. 实现与验收记录（2026-08-02）

D21.1–D21.8 已按本文三条不变量实现：DATABASE 的 HDF5 UUID 列是完成事实的
唯一来源；checkpoint 保存滚动安全恢复点；Archiver 在写入端按 UUID 去重。落盘确认顺序为
HDF5 commit/flush → fsync → Redis `SADD hep:archived:<scan>`。失败样本同样写入带
`status` 的最小记录，避免恢复时无限重算。

生命周期通路也已闭合：控制进程使用 Redis TTL lease，Worker/Archiver 在租约失配时退出；
FileOperation 被 init 收养后会自行退出；控制 PID 已死时 `ps`/`kill` 接受该 runtime 为 stale；
`--resume` 会清理同名孤儿进程并接管租约；若 SIGKILL 留下 HDF5 writer consistency flag，
恢复前会以 `h5clear` 清理后再读 UUID。

行为验收：

- **A1/A3/A4**：真实 4000 点扫描，SIGINT 后续跑；最终 4000 行 / 4000 unique UUID，
  与未中断同 seed 基线逐 UUID 数值一致，重投量保持在 in-flight 上界内。
- **A2/A5**：SIGKILL 控制进程并删除 checkpoint；仅靠 DATABASE 对账可恢复，最终
  200 行 / 200 unique UUID；stale Worker/Archiver/FileOperation 与 HDF5 标志均被处理。
- **A6**：AdaptiveBridson 在世代内 SIGINT 后恢复，最终 `levelset.json` 与未中断基线
  完全一致，DATABASE 行数等于 unique submitted 数；最终独立复跑 `1 passed in 42.63s`。
- **A7**：删除测试中的人工 `mark_completed()` 接线，改为真实控制进程、Worker、
  Archiver、Redis、HDF5 的端到端断言。
- D21 相关焦点回归：89 passed；恢复/Redis/中断补充回归：39 passed；Ruff 与
  `git diff --check` 通过。
- 全量回归：**749 passed, 58 subtests passed, 5 known failures**。失败均属于 D13.15
  已登记基线（1 个 Redis collision fixture、4 个 legacy `--plot` parser tests）；
  开工前另一个临界 scaling 失败本次通过，因此没有 D21 新增失败。

---

# 第二版：把 checkpoint 挪出热路径（D21.9–D21.12，2026-08-02）

## 13. 新增的硬约束（维护者定调）

> *"checkpoint 一定不能影响运行时的性能。sample 的吞吐量和并发量不能受到影响。
> 断点续跑时可以多跑一些点，因为有心跳的原因，其实也不会浪费多少算力。"*

**C1 — checkpoint 在热路径上的开销必须是 O(1) 且可忽略。** 吞吐与并发不受影响。
**C2 — 允许 resume 重算一部分点。** 上界是"一个心跳周期的墙钟工作量"，可接受。

C2 是 C1 的**授权**：一旦接受重算，恢复点就不必紧贴在途窗口，第一版为了"最小重算"而付出的
每批代价就没有必要了。

## 14. 为什么第一版违反了 C1（实测）

| | 实测 |
|---|---|
| 改动前基线（20000 点、同卡同机） | **145 samples/s**，wall 137 s |
| 第一版（N≈25k 处） | **12 samples/s**，且随 N 继续劣化 |
| 控制进程 RSS | **1.6–4.2 GB 锯齿**（2 变量玩具扫描） |
| checkpoint 文件 | 20k 样本时 **2.26 MB**，线性增长，每 30 s 重写 |

病灶在 `fixed_set_sampler.py:80`：

```python
self._pending_restore_batches.append(
    ({sample.uuid for sample in batch}, deepcopy(self.export_runtime_state()))
)
```

`export_runtime_state()` 每次物化三个 **O(N)** 容器（`u_by_uuid` 全量 `tolist()`、
`uuid_by_accepted_index`、`submitted_uuids`），`batch_size: 2` ⇒ 每 2 个样本一次 O(N) 拷贝
⇒ **总体 O(N²)**；`_pending_restore_batches` 只在心跳排空，峰值同时驻留约一千份 O(N) 快照。

另有一处同类问题：控制进程每次心跳用 `SMEMBERS` 读整个 `hep:archived:<scan>`——**每 30 s 一次 O(N)**。

**这条路径对所有运行生效，不只是 resume。** 而 resume 唯一有价值的场景就是长扫描，
恰恰是 O(N²) 崩溃的地方。

## 15. 第二版机制：心跳快照 + 索引水位

### 15.1 三个部件

1. **快照只含 generator 本身**（~3 KB 定长）：
   `(rng_state | Generator, index, accepted_index, ready_queue)`。
   `u_by_uuid` / `uuid_by_accepted_index` / `submitted_uuids` **全部移除**——见 §16。

2. **快照由心跳触发、由 sampler 在批边界执行。**
   心跳线程只置一个布尔标志；sampler 在下一个 `flush_batch` 边界看到标志后自己做 O(1) 捕获。
   **热路径的固定开销 = 每批一次布尔判断**，没有锁竞争、没有拷贝。

3. **索引水位（contiguous persisted prefix）`P`**：满足"索引 `[0, P)` 全部已落盘"的最大 `P`。
   增量维护：每收到一批新的落盘确认就尝试推进 `P`，乱序到达的索引暂存在一个
   **以在途窗口为界**（而非以 N 为界）的小集合里。

### 15.2 恢复点的选取规则

```
保存的恢复点 = 最新的那个满足 index_j ≤ P 的心跳快照 S_j
```

保留最近 K 个心跳快照（K 取个位数，每个 3 KB）即可，选最新的合格者。

**正确性论证**：`S_j` 的条件保证索引 `[0, index_j)` 全部在盘上。resume 时从 `index_j`
前向重放，`≥ index_j` 的索引全部被重新产出——已落盘的由写入端去重跳过，未落盘的重算。
于是**没有任何索引会被遗漏**：小于 `index_j` 的在盘上，大于等于的被重放覆盖。∎

这正是"心跳快照"必须配水位的原因：**裸的心跳快照是不安全的**——心跳时刻的 generator
已经越过在途点，直接用它恢复会让那些点永远不被重新提出（S2 被打破，点会静默丢失）。
水位条件把这个洞堵死，且代价是 O(1)。

### 15.3 重算量

上界 = 从 `S_j` 到崩溃之间提交的点，即**一个心跳周期的墙钟工作量**（外加落盘滞后）。

- 每样本 0.007 s 的玩具扫描：30 s ≈ 4000 点，但它们很便宜
- 每样本 60 s 的真实计算：30 s ≈ 每 Worker 一个点

两种情形下"浪费"都约等于**半个到一个心跳周期的墙钟时间**，与 N 无关——正是 C2 接受的那个量级。
因此 `CHECKPOINT_HEARTBEAT_SEC` 应当**可配置**（贵样本的用户可以调小），不再是写死的 30。

## 16. 连带删除

`u_by_uuid`、`uuid_by_accepted_index`、`submitted_uuids`、`repropose_unfinished()`
存在的唯一理由是"按 uuid 把在途点重新投一遍"。改为"恢复 + 重放"后它们全部多余：

| 结构 | 为什么可以删 |
|---|---|
| `uuid_by_accepted_index` | `uuid = sha256(prefix:seed:index)` 是**纯函数**，重算即可（§3） |
| `u_by_uuid` | 重放会用同一个 RNG 状态重新产出同样的 u_coords |
| `submitted_uuids` | 完成事实在 DATABASE；提交事实由 `index` 表达 |
| `repropose_unfinished()` | 重放本身就覆盖了在途点 |

跟 `mark_completed()` 一样——**少一套机制，不是多一套**。

### 16.1 pickle 而非 dill；不要 pickle sampler 对象

payload 全是 dict / int / ndarray / RNG 元组，**原生可 pickle**；`np.random.Generator` 本身
也可 pickle。dill 的价值在 lambda/闭包/动态类，这里一个都没有——**不引入 dill**。
现有的 `atomic_pickle_dump`（`.tmp` + `os.replace`）保留。

**不要直接 pickle 活着的 sampler 对象**：它挂着 `self._logger`（loguru handle）、
`self.redis`（socket）、`self.config`，拖不动；要 pickle 就得写 `__getstate__` 剔除它们，
那本质上就是现在的显式 export，只是换个拼法。

保留显式 export，但补一条防漏测试：**断言 sampler 的每个属性都被显式归类为"保存"或"排除"**，
新增字段未归类即测试失败。这样既不会"保险起见全存"退回 O(N²)，也不会静默漏状态。

## 17. 是否拆成两个文件

可以，但分界是**可变 / 不可变**，不是 sampler / factory：

| 文件 | 内容 | 写入时机 |
|---|---|---|
| `run.json` | seed、config hash、variable signature、scan 名、格式版本 | 运行开始写一次 |
| `state.pkl` | generator 状态（~3 KB） | 每次恢复点推进 |

payload 缩到 3 KB 之后这只是整洁性问题，**不是修复**，可选。

**明确不做：用 factory 的 pickle 记录"哪些点落盘了"。** 那会重新引入第二个真相来源，
而崩溃点必然落在"写了盘"和"更新簿记"之间，两者**一定会不一致**——`completed_uuids` /
`acked_uuids` 之前就是这么错的。核对过：factory 本身没有需要 pickle 的状态——
SAMPLE 的 bucket 编号由 `init_sample_buckets()` **扫目录**得出，calculator 池由配置重建，
记录里本来就带 `bucket_dir` / `bucket_id`。

最终形态仍然是两份信息的综合，只是第二份是 HDF5 本身而不是 pickle：

```
恢复节点 = generator 恢复点（state.pkl，~3 KB）⊕ 已落盘 UUID（DATABASE，唯一真相）
```

## 18. 落盘确认的增量获取

控制进程必须能以 **O(新增)** 而非 O(N) 获知新落盘的样本，否则心跳自己就成了 O(N)。
两条可选路径，实现者择一：

- **(a) 流式**：Archiver 在 `SADD` 之外再 `RPUSH` 一条增量记录，控制进程每心跳只 drain 增量；
- **(b) 记录里加 `sample_index`**：Archiver 可直接发布**单调的连续索引水位**，
  控制进程连映射表都不用维护，`P` 直接读。

**(b) 更彻底**——它让水位、去重、resume 的索引对账三件事共用同一个字段，代价是记录里多一个整数。

写入端去重集当前是"全部已落盘 UUID"的内存集合（10⁶ 样本约百 MB 量级）。若采用 (b)，
去重可退化为**索引区间判断**，内存降到常数。

## 19. 第二版验收（性能是门禁，不是附注）

| # | 验收 |
|---|---|
| **P1** | 同一张 20000 点卡片，吞吐**回到 ≥ 基线 145 samples/s 的 90%**；与关闭 checkpoint 的对照跑差异 < 5% |
| **P2** | 控制进程 RSS **平坦**（不随 N 增长），全程 < 500 MB |
| **P3** | checkpoint 文件大小**不随 N 增长**（20k 与 200k 样本处相差 < 2×） |
| **P4** | 每批热路径新增开销 = 一次布尔判断；无锁竞争（用 profile 或计数断言） |
| **P5** | 心跳自身为 O(新增)：单次心跳耗时不随 N 增长 |
| A1′ | 中断 + resume 后 DB `rows == unique`（沿用第一版 A1） |
| A3′ | 重算量 ≤ **一个心跳周期**的提交量（不再是 in-flight 上界），并记录实测值 |
| A5′ | 删除 checkpoint 后仍能靠 DATABASE 续跑（沿用第一版 A5） |
| A8 | **不丢点**：`S_j ≤ P` 水位条件的负面测试——人为让某个索引晚落盘，确认恢复点不会跨过它 |

## 20. 第二版工作包

| WP | 内容 | 依赖 | 优先级 |
|---|---|---|---|
| **D21.9** | 移除每批 `deepcopy(export_runtime_state())`；改为**心跳置标志 + 批边界 O(1) 捕获**；引入索引水位 `P` 与"最新合格快照"选取 | — | **高**（吞吐塌陷的直接原因） |
| **D21.10** | payload 瘦身：只存 generator 状态；删除 `u_by_uuid` / `uuid_by_accepted_index` / `submitted_uuids` / `repropose_unfinished()`；补属性归类测试 | D21.9 | 高 |
| **D21.11** | 落盘确认增量化（§18，建议走 (b) 加 `sample_index`）；`CHECKPOINT_HEARTBEAT_SEC` 改为可配置 | D21.9 | 中 |
| **D21.12** | 性能门禁 P1–P5 进回归；A8 负面测试 | D21.9–D21.11 | 高 |

**回滚**：这一版只改 checkpoint 的**取样时机与载荷**，不改三条不变量（I1 落盘为真相、
I2 恢复点不越过在途、I3 写入端去重），A1/A5 的行为断言全部沿用。

## 13. FileOperation 孤儿复核与加固（D21.9）

复核机器上 11 个 `ppid=1` 的历史 `Jarvis2-FileOperation` 后，确认来源是 D21 之前的
永久阻塞循环：Worker 被 SIGKILL 后，子进程仍阻塞于无 timeout 的
`request_queue.get()`。这些进程没有 scan 后缀，也不会进入按 scan 分组的 `ps`/`kill`。

D21 最初加入的空闲 PPID 检查仍有漏洞：主线程忙于 NAS copy/delete、阻塞于 response
queue，或等待 `rm` 子进程时，不会回到空闲检查点。最终实现改为：

1. FileOperation 启动独立 owner-watch 线程，Worker PPID 一旦变化立即退出；
2. FileOperation 成为独立 session/process-group leader，退出/强杀覆盖嵌套的 `rm`；
3. Worker heartbeat 公布 FileOperation PID，Factory watchdog 可按 session leader 安全兜底；
4. client 等待 response 时轮询 child liveness，子进程崩溃不再让 Worker 永久挂住；
5. Worker 初始化包含在 cleanup 边界内，scheduler/file-operation/Redis 分别清理，互不跳过；
6. Factory 优雅退出超时后使用真正 SIGKILL，不再重复发送被 Worker handler 吞掉的 SIGTERM；
7. 无对应 live Control 的已知 Jarvis 进程统一进入固定 `ZP`（Zambie Process）分组，
   可通过 `Jarvis2 ps ZP` 检查、`Jarvis2 kill ZP -y` 一键清理；不再伪装成 `R#` 扫描。

真实验收将 FileOperation 阻塞在 FIFO copy 中后 SIGKILL Worker，忙碌子进程按 owner-watch
退出；最终焦点测试 33 passed，扩展 Worker/FileOperation/resume 回归 62 passed，Ruff 与
`git diff --check` 通过，测试结束后的完整进程表中 FileOperation 数为 0。

## 14. ZP 无归属进程分类（D21.10）

`Jarvis2 ps` 会同时展示两类引用：有 scan id 的运行时继续按名称稳定分配 `R1/R2/...`；
没有对应 live Control 的裸进程或残留 Worker/Archiver/FileOperation/Redis 统一显示为固定
`ZP`；即使残留标题仍带旧 `:<scan>` 后缀，也不能再占用 `R#`。
`Jarvis2 ps ZP` 不依赖 Redis/runtime metadata，适用于控制进程身份已经丢失的历史孤儿；
`Jarvis2 kill ZP -y` 对 ZP 中全部进程执行 TERM→KILL 清理。

发现阶段保持与 `ps ... | grep -E 'Jarvis2|Jarvis-Redis'` 相同的宽匹配思路，真正进入
ZP 和允许 kill 时则只接受 Jarvis 自己会生成的裸标题，避免误杀 `Jarvis2Helper` 一类
仅共享前缀的第三方程序。真实 OS 验收中，裸 FileOperation 进入 ZP，正常 `scan` 保持 R1；
执行 `kill ZP -y` 后只有孤儿进程退出，R1 未受影响。补充验收确认孤立的
`Jarvis2-Archiver:scan` 即使残留 scan 后缀，也进入 ZP 而不是虚假的 R1。
