# DESIGN — 断点续跑（Checkpoint / Resume 重做，V2，D21）

**Status**: design proposal 2026-08-02 — 维护者已定调核心语义（§1），待实现
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
