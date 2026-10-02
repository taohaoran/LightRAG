# 文件型五存储家族（file-backed-stores）

> 本文是 `storage` 域下的叶子子系统文档，也是本域最重的叶子。域级总览见 `../storage.md`。
> 本文展开五个进程内副本存储：NetworkX（图）、NanoVectorDB / Faiss（向量）、JsonKV / JsonDocStatus（KV/文档状态），
> 以及支撑它们的文件指纹围栏、向量空间标记与空向量实现。跨进程原语本身见 `../storage-coordination/`。
>
> 源码基准：`lightrag` main 分支，commit `453dce83`。
> **权威契约**：`docs/design/NetworkXSingleWriterContract.md`、`docs/design/FileBackedSnapshotContract.md`（已读）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| NetworkX 图存储 | 整图驻留进程内存，整图序列化到一个 GraphML 文件发布；唯一声明 `requires_single_writer` | `lightrag/kg/networkx_impl.py:85` |
| Nano 向量存储 | 单 `NanoVectorDB` 实例驻内存，整状态序列化到一个 JSON 文件 | `lightrag/kg/nano_vector_db_impl.py:62` |
| Faiss 向量存储 | `IndexFlatIP` 索引 + `_id_to_meta` 两文件；L2 归一化后内积即余弦 | `lightrag/kg/faiss_impl.py:63` |
| Json KV 存储 | `_data` 是 `Manager().dict()` 共享代理，磁盘文件仅做持久化；延迟提交 | `lightrag/kg/json_kv_impl.py:34` |
| Json 文档状态存储 | 不继承 JsonKV，按相同 shared_storage 原语重实现；调度状态写同步刷盘 | `lightrag/kg/json_doc_status_impl.py:59` |
| 文件指纹通道 | `(st_mtime_ns, st_size)` 作为权威状态通道；`UNREADABLE`/`fence_enabled`/`publication_complete` | `lightrag/kg/file_fingerprint.py:60,67,287` |
| 向量空间标记 | 记录嵌入模型名/维度，切换模型时检测不匹配 | `lightrag/kg/vector_space.py:68,120` |
| 空向量后端 | 全部方法 no-op，用于关闭向量索引的场景 | `lightrag/kg/noop_vector_db_impl.py:12` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `NetworkXStorage` | `networkx_impl.py:85` | `requires_single_writer=True`；`_get_graph`（:513）是唯一取图入口兼重载点；`index_done_callback`（:1481）返回 bool，文件被越过时**拒绝提交** |
| `_commit_gate` | `networkx_impl.py:161` | `asyncio.Event`，序列化期间 clear，防止 worker 线程遍历图时协程改图 |
| `_recovery_reload_pending` | `networkx_impl.py:141` | 进程本地 bool：记录"我欠自己一次重载"，不是第三通道 |
| `NanoVectorDBStorage` | `nano_vector_db_impl.py:62` | 四缓冲：`_pending_upserts`(:174)、`_pending_deletes`(:180)、`_unsaved_deletes`(:184)、`_unsaved_upserts`(:195) |
| `FaissVectorDBStorage` | `faiss_impl.py:63` | 同样四缓冲；`_save_faiss_index` 在 worker 线程跑，只读索引 |
| `JsonKVStorage` | `json_kv_impl.py:34` | `_data` 为共享代理；`storage_updated` 语义相反（True=脏待刷） |
| `JsonDocStatusStorage` | `json_doc_status_impl.py:59` | 重实现同协议；`index_done_callback`(:257) 对调度状态写同步刷盘 |
| `file_fingerprint` 模块 | `file_fingerprint.py` | `sample`(:77)、`adopted`(:108)、`peer_commit_detected`(:216)、`publication_complete`(:287) |

## 3. 关键调用链

**调用链一：NetworkX 写者提交（被越过则拒绝）**
1. 业务经 `upsert_node`（`networkx_impl.py:801`）→ `_get_graph`（:513）在内存改图。
2. 批末 `index_done_callback`（:1481）：在 `_storage_lock` 内先做两通道检测，若文件已被越过程序提交则**返回 False**（拒绝，丢弃本进程未发布 mutation）。
3. 通过则 `write_nx_graph`（:102）经 `atomic_write` 原子写 GraphML，记录指纹。
4. `set_all_update_flags` 翻各进程标志，再复位自己。
5. 序列化在 worker 线程跑期间 `_commit_gate`（:161）clear，挡住协程改图。

**调用链二：向量后端写冲突（重载并重放）**
1. `upsert`（`nano_vector_db_impl.py:455`）只把 `vector=None` 的记录压入 `_pending_upserts`，不调嵌入模型。
2. `index_done_callback` → `_flush_pending_locked`（:531）：先重放 `_unsaved_deletes`/`_unsaved_upserts` redo log（按 `__write_seq__` 裁决新旧），再批量嵌入 pending upsert、批量 delete。
3. 检测到文件被越过时，**重载磁盘快照后重放缓冲**——与 NetworkX 的"拒绝"相反。

**调用链三：JsonKV/JsonDocStatus（共享内存，无重载）**
1. `upsert`（`json_kv_impl.py:314`）在锁内直接改 `self._data`（共享 dict 代理，跨进程立即可见），并 `set_all_update_flags(True)`。
2. `index_done_callback`（`json_kv_impl.py:134`）：若 `storage_updated.value` 为 True，快照整个 `_data` 经 `write_json` 原子落盘，再 `clear_all_update_flags`。
3. `JsonDocStatusStorage` 的调度状态写（`upsert`:368 / `update_doc_status_fields`:962）**同步** `await index_done_callback()`——因为它是管线恢复锚点。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `working_dir` | 文件落 `working_dir/[workspace/]<文件>`；GraphML / vdb json / kv_store json | 各 impl `__post_init__` |
| `cosine_better_than_threshold` | 向量检索阈值（PG/向量后端共享） | LightRAG 构造参数 |
| 单/多进程 | 多进程才启用文件指纹通道（`fence_enabled`，`file_fingerprint.py:67`）；单进程跳过 | `file_fingerprint.py:67` |

## 5. 错误与重试语义

- **NetworkX 拒绝提交必须 loud**：`index_done_callback` 返回 False 被三类调用方转成失败——`utils_graph._commit_graph_or_raise` 抛错、`LightRAG._flush_storages` 包成 `IndexFlushError`、`_persist_graph_updates` 抛 `_declined_commit_error`。失败后文档走 FAILED 重处理自愈。
- **脏图兜底**：`_graph_dirty` 记录未发布 mutation；若重载替换了脏图则置 `_dirty_discard_pending`，下次提交以 `GraphMutationsDiscardedError` 拒绝（`networkx_impl.py:8` 导入），不允许静默成功。
- **Faiss 跨文件非原子**：`.index` 与 `.meta.json` 两次 `atomic_write` 独立；崩溃中间态靠 `_load_faiss_index` 容忍（meta>index 静默丢行，index>meta 告警不自动修）。
- **redo log 按行指纹匹配**：`_unsaved_deletes` 存被删行的整行指纹，回放只删仍匹配的行，避免误删他进程同名 id 的新行。

## 6. 并发细节

- **三不变量**：每工作区单写者；最终一致即可；mutation 在单线程事件循环上同步不可抢占（故方法不在调用 `graph.xxx`/`client.xxx` 时持 `_storage_lock`）。
- **两通道围栏**：文件指纹（权威、状态）OR `storage_updated`（加速、事件）。盲点不重叠：指纹怕同刻同尺寸两次提交，标志怕 Manager RPC 漏通知。
- **NetworkX vs 向量的写冲突分叉**：图 store **拒绝** commit（payload 是 accumulate-over-read，重放会丢他进程累积的证据）；向量 store **重载并重放**（缓冲存整行按 id，last-writer-wins 即契约）。
- **JsonKV/JsonDocStatus 锁范围更大**：对每次 `self._data` 访问（含读）持锁，因 `Manager().dict()` 不跨线程自由；读侧 `deepcopy` 返回，不外泄代理引用。
- **`storage_updated` 语义相反**：NetworkX/Nano/Faiss 中 True="磁盘有更新需重载"；JsonKV/JsonDocStatus 中 True="有脏数据待刷盘"。

## 7. 系统边界

**In-Scope**：`lightrag/kg/` 下 networkx_impl、nano_vector_db_impl、faiss_impl、json_kv_impl、json_doc_status_impl、file_fingerprint、vector_space、noop_vector_db。

**Out-of-Scope（不在本仓库源码内）**
- `networkx` / `faiss` 第三方库本身（图库/索引算法）。
- 外部服务端后端（PG/Mongo/Neo4j/Redis/Milvus/Qdrant/Memgraph/OpenSearch）——生产部署用它们，本家族仅用于小规模测试验证。
- admin 写门 `_admin_write_gate` 如何消费 `requires_single_writer`：见 `../graph-operations/`。

## 8. 与相邻子系统交互

- **上游**：pipeline 摄取与 purge 流经 `busy` 预约串行化这些写；admin 图写经 `_admin_write_gate`（NetworkX 专属）。
- **下游**：`file_fingerprint` 被三个重载型存储共用；`write_seq` 的 `__write_seq__` 被 Nano/Faiss redo log 比较消费；`set_all_update_flags` 等原语来自 `../storage-coordination/`。

## 9. 语言专项适配口径

按 `language-python.md`：本叶子是纯 Python 框架的**双轨实现 + 生产者-消费者缓冲**。
- **双轨**：同一抽象契约下，文件型纯 Python 兜底（内存+整文件重写）与服务端后端（行级并发）并存。
- **缓冲/redo log**：延迟嵌入（upsert 不调模型，flush 时一次嵌入）、延迟删除（合并热路径上一次 `client.delete` 批量）是典型生产者-消费者批量。
- **隐藏状态机**：NetworkX 的 `_recovery_reload_pending` 与脏图标记是"循环内隐藏状态机"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 文件型家族分层与两通道围栏 | `file-backed-stores-architecture.html` | architecture | standard |
| 向量后端写冲突重载重放数据流 | `file-backed-stores-replay-dataflow.html` | dataflow | showcase |
| NetworkX 提交主路径生命周期 | `file-backed-stores-commit-lifecycle.html` | lifecycle | showcase |

架构图落 standard：组件分两区、连线较多，一次渲染通过 standard（无致命错误）。dataflow 一次通过 showcase。lifecycle 经历两轮：首轮 schema 已演进为 `lanes`/`col`（替代旧 `phase`），按保留泳道 `main`/`terminal` 重排后，分支失败态反复触发 `clean-flow/edge-through-node` 穿越与标签重叠，按 cheatsheet 拆为无分支线性主路径后通过 showcase；拒绝提交分支语义写入本 MD 第 3/5 节。

JSON IR 源文件位于 `json/` 目录。
