# 存储协调与跨进程原语（storage-coordination）

> 本文是 `storage` 域下的叶子子系统文档。域级总览见 `../storage.md`。
> 本文只展开**跨进程共享状态、命名空间锁、更新标志围栏、pipeline_status 与写序号**等协调原语；
> 这些原语如何被各后端消费（两通道围栏、写门）见 `../file-backed-stores/` 与 `../graph-operations/`。
>
> 源码基准：`lightrag` main 分支，commit `453dce83`。
> 说明：facts.md 提到的 `namespace.py` 在本版本中不存在——命名空间拼接逻辑已内聚在
> `shared_storage.py:get_final_namespace`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 共享数据初始化 | 单进程用线程锁 + 本地 dict；多进程用 `multiprocessing.Manager()` 的共享 dict/Value 代理 | `lightrag/kg/shared_storage.py:1524`（`initialize_share_data`） |
| 命名空间拼接 | `workspace:namespace` 统一前缀，实现工作区隔离 | `lightrag/kg/shared_storage.py:209`（`get_final_namespace`） |
| 双模式统一锁 | 同一把锁同时支持 `async with` 与同步 `with`，跨进程时用 Manager 锁 | `lightrag/kg/shared_storage.py:247`（`UnifiedLock`） |
| 键化锁表 | 按 `(namespace, key)` 维度加细粒度锁，含死持有者检测与租约回收 | `lightrag/kg/shared_storage.py:936`（`KeyedUnifiedLock`）、`:753`（`KeyedHolderTable`） |
| 命名空间锁门面 | 返回可重复使用的 `NamespaceLock`，每次用开新上下文防重入 | `lightrag/kg/shared_storage.py:3971`（`get_namespace_lock`） |
| 全局初始化旗 | 每个 `(namespace, workspace)` 仅一个进程返回 `True` 负责从文件加载 | `lightrag/kg/shared_storage.py:3812`（`try_initialize_namespace`） |
| 共享数据引用 | 返回某命名空间的 `Manager().dict()` 代理引用（跨进程立即可见） | `lightrag/kg/shared_storage.py:3840`（`get_namespace_data`） |
| 更新标志围栏 | `set_all_update_flags` 把每个进程的 `storage_updated` 置 True；`clear_all_update_flags` 复位 | `lightrag/kg/shared_storage.py:3736` / `:3755` |
| pipeline_status 共享态 | 每工作区一个共享 dict，承载 busy/destructive_busy/scanning 等调度状态 | `lightrag/kg/shared_storage.py:1653`（`initialize_pipeline_status`） |
| 工作区摄入邮箱 | pipeline 唯一唤醒通道 | `lightrag/kg/shared_storage.py:1788`（`get_pipeline_ingress`） |
| 写序号令牌 | 每次 upsert 盖纳秒单调令牌 `__write_seq__`，供 redo log 跨进程比较新旧 | `lightrag/kg/write_seq.py`（`next_write_seq`、`row_is_strictly_newer`） |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `UnifiedLock` | `shared_storage.py:247` | 双模式锁：`__aenter__/__aexit__` 走 asyncio，`__enter__/__exit__` 走同步；多进程时经 executor 持有 Manager 锁 |
| `KeyedUnifiedLock` | `shared_storage.py:936` | 键化锁工厂，`__call__` 返回 `_KeyedLockContext`；用 `KeyedHolderTable` 记录持有者，死进程持有的锁可被回收 |
| `KeyedHolderTable` | `shared_storage.py:753` | 跨进程持有者表（Manager 代理 `_HolderTableProxy`），`try_acquire`/`release`/`holders_snapshot`；`_holder_dead`（:713）借 `/proc` starttime 判定持锁进程是否真死 |
| `NamespaceLock` | （由 `get_namespace_lock` 返回） | 可重复使用的上下文包装，`async with lock:` 每次新建底层锁上下文，避免非重入锁死锁 |
| `_LightRAGManager` | `shared_storage.py:860` | 注册了 `KeyedHolderTable` 的 `SyncManager` 子类，使持有者表可跨进程共享 |
| `try_acquire_global_slot` | `shared_storage.py:4429` | 跨 worker 全局并发槽位（如 `llm:extract`、`embedding`），带租约与等待者队列 |
| `next_write_seq` | `write_seq.py` | `max(time.time_ns(), _last_seq+1)`，进程内严格递增；`threading.Lock` 保护 |
| `row_is_strictly_newer` | `write_seq.py` | 两行都有令牌则令牌决定；一方缺失回退整秒 `__created_at__`；平局判 False（回放继续） |

## 3. 关键调用链

**调用链一：多进程共享态建立**
1. Gunicorn preload 或 FastAPI lifespan 调 `initialize_share_data(workers=n)`（`shared_storage.py:1524`）。
2. `workers==1` 走单进程：用 `threading.Lock` 与普通 `dict`。
3. `workers>1` 走多进程：创建 `_LightRAGManager`，`_shared_dicts`/`_update_flags`/`_init_flags` 均为 Manager 代理。
4. 每个存储后端 `initialize()` 时调 `try_initialize_namespace`（`shared_storage.py:3812`）——全局 init 锁保证恰好一个进程从 JSON/GraphML 文件加载初始数据，其余进程经 `get_namespace_data` 看到同一代理。

**调用链二：两通道围栏的标志通道（加速通道）**
1. 写者提交后调 `set_all_update_flags(namespace, workspace)`（`shared_storage.py:3736`）。
2. 在 `get_internal_lock()` 保护下，对 `_update_flags[final_namespace][:]` 每个 `flag.value=True`（一次 slice 快照减少 RPC）。
3. 读者在 `_storage_lock` 内先查自己的 `storage_updated.value`；为 True 即触发重载。
4. 写者随后把自己的标志复位 False。

**调用链三：写序号裁决 redo log**
1. 向量后端 `upsert` 时调 `next_write_seq()`（`write_seq.py`）盖 `__write_seq__`。
2. 提交失败后重载磁盘快照，flush 时对每条 redo 日志调 `row_is_strictly_newer(candidate, reference)`（`write_seq.py`）。
3. 两行都带令牌 → 令牌大者为新，新者保留、旧者被回放覆盖；一方缺令牌 → 回退整秒时间戳；平局 → 判 False（回放继续，保留旧行为）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `workers`（Gunicorn/API） | 1=单进程（线程锁+本地 dict）；>1=多进程（Manager 共享） | `initialize_share_data` 形参，`:1525` |
| `global_concurrency_limits` | 按组（`llm:extract`/`embedding`/`rerank`）限跨 worker 并发，初始化后只读 | `shared_storage.py:1526` |
| `workspace` | 经 `get_final_namespace` 拼成 `workspace:namespace` 前缀隔离 | `shared_storage.py:221` |
| `ADMIN_WRITE_LOCK_ACQUIRE_TIMEOUT` 等 | admin 写锁等待上限（被 `_admin_write_gate` 消费） | 见 graph-operations 叶子 |

## 5. 错误与重试语义

- **死持锁回收**：`KeyedHolderTable.try_acquire` 发现持锁进程已死（`_holder_dead` 借 `/proc/<pid>` starttime 判定 pid 复用，`shared_storage.py:713`）时让新写者接管，避免死锁。
- **全局槽位租约**：跨 worker 并发槽带租约（`renew_global_slots`/`reconcile_global_slots`），崩溃进程的槽位由 `reconcile` 回收。
- **标志通道可能丢失**：`set_all_update_flags` 对每个进程一次 Manager RPC，部分发布漏掉某进程是其固有盲点——由文件指纹通道兜底（见 file-backed-stores 叶子）。
- **未初始化即用**：`set_all_update_flags`/`get_namespace_data` 在共享数据未初始化时抛 `ValueError`，fail loud。

## 6. 并发细节

- **锁的分层**：`get_internal_lock()`（:1408）保护共享表结构本身；`get_namespace_lock`（:3971）保护某命名空间数据；`get_storage_keyed_lock`（:1430）是存储后端用的键化锁。
- **非重入**：`NamespaceLock` 不可重入，故 `JsonKVStorage.upsert` 用 `_cooperative_yield` 在键间让出事件循环——兄弟协程阻塞在同一把锁上仍保持阻塞。
- **Manager RPC 优化**：`set_all_update_flags` 先 `[:]` slice 一次拿到 ValueProxy 列表再遍历写 `.value`，避免每次迭代一次 RPC（`shared_storage.py:3747-3752`）。
- **写序号线程安全**：`next_write_seq` 用模块级 `threading.Lock` + 高水位 `_last_seq`，保证同进程内严格递增。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/kg/shared_storage.py`：全部跨进程原语。
- `lightrag/kg/write_seq.py`：写序号令牌。

**Out-of-Scope（不在本仓库源码内）**
- `multiprocessing.Manager` / `SyncManager` 本身是 Python 标准库。
- 各后端如何把这些原语接进自己的重载/提交流程：见 file-backed-stores 叶子。
- pipeline_status 的 busy 语义与入队拒绝规则：pipeline 域叶子。

## 8. 与相邻子系统交互

- **上游**：API 服务（Gunicorn preload / FastAPI lifespan）调 `initialize_share_data`；各存储后端 `initialize()` 调 `try_initialize_namespace`/`get_namespace_data`。
- **下游消费**：文件型五存储用 `set_all_update_flags` + 文件指纹组成两通道围栏（L10）；`pipeline_status` 被 pipeline 域与 admin 写门读写（L15）；`__write_seq__` 被 Nano/Faiss 的 redo log 比较消费。

## 9. 语言专项适配口径

按 `language-python.md`：本叶子是纯 Python 框架的**生产者-消费者 + 上下文管理器**能力缝。
- **多进程双轨**：单进程（线程锁+本地 dict）与多进程（Manager 代理）两套实现，运行时按 `workers` 切换——双轨实现模式。
- **可选依赖懒加载不适用**：这里是标准库 `multiprocessing`，无外部依赖。
- **跨进程一致性**：Manager 代理是唯一跨面；文件是另一同步面；两者互补（盲点不重叠）是核心设计。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 共享存储协调组件拓扑 | `storage-coordination-architecture.html` | architecture | standard |
| 提交者发布与读者检测时序 | `storage-coordination-fence-sequence.html` | sequence | showcase |

架构图落 standard：首轮 `clean-flow/edge-through-node` 报 c6（worker1→持有者表）横穿 sdict 组件，调整 orthogonal 路由仍穿越，最终删除该次要边（租约关系写入本 MD 第 2/6 节）后渲染通过。时序图一次通过 showcase。

JSON IR 源文件位于 `json/` 目录。
