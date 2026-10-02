# 文档与搜索数据库家族（document-db-storage）

> 本文是 `storage` 域下的叶子子系统文档。域级总览见 `../storage.md`。
> 本文展开 MongoDB（文档库）与 OpenSearch（搜索库）两套家族，每套各实现四类存储。
> 按"目录即实例库"口径：深读 MongoDB 代表，OpenSearch 以共性机制（延迟缓冲+刷新代际）说明并披露缺口。
>
> 源码基准：`lightrag` main 分支，commit `453dce83`。mongo_impl.py 约 5246 行、opensearch_impl.py 约 7352 行。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Mongo KV | 文档库 KV，即时写 | `lightrag/kg/mongo_impl.py:405`（`MongoKVStorage`） |
| Mongo 文档状态 | 调度状态集合 | `lightrag/kg/mongo_impl.py:641`（`MongoDocStatusStorage`） |
| Mongo 图 | 文档库内自连表模拟图 | `lightrag/kg/mongo_impl.py:1765`（`MongoGraphStorage`） |
| Mongo 向量 | 文档库向量字段 | `lightrag/kg/mongo_impl.py:4220`（`MongoVectorDBStorage`） |
| OpenSearch KV | 每进程延迟缓冲 + bulk 提交；刷新代际门控 | `lightrag/kg/opensearch_impl.py:992`（`OpenSearchKVStorage`） |
| OpenSearch 文档状态 | 写时 `refresh="wait_for"` 强可见 | `lightrag/kg/opensearch_impl.py:1906`（`OpenSearchDocStatusStorage`） |
| OpenSearch 图 | 搜索索引模拟图 | `lightrag/kg/opensearch_impl.py:3679`（`OpenSearchGraphStorage`） |
| OpenSearch 向量 | 延迟缓冲向量 | `lightrag/kg/opensearch_impl.py:6218`（`OpenSearchVectorDBStorage`） |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `ClientManager`（mongo） | `mongo_impl.py:358` | 连接客户端生命周期 |
| `ClientManager`（opensearch） | `opensearch_impl.py:787` | 连接客户端生命周期 |
| `OpenSearchKVStorage` | `opensearch_impl.py:992` | 持有 `_write_generation`(:1004)/`_refreshed_generation`(:1005)；`_flush_pending_kv_ops`(:1541) |
| `OpenSearchReferencesIntactError` | `opensearch_impl.py:123` | 继承 `ReferencesIntactFlushError`，证明 bulk/refresh 失败未丢引用 |
| `_FailedBulkOp` | `opensearch_impl.py:150` | 缓冲里失败的单次操作（区分可重试 vs 永久失败） |
| `WorkspaceIndexCollisionError` | `opensearch_impl.py:434` | 工作区索引冲突 |

## 3. 关键调用链

**调用链一：Mongo 即时写**
1. `MongoKVStorage.upsert` 直接 `replace_one`/`insert_many` 落文档。
2. `index_done_callback`（`mongo_impl.py:574`）近似 no-op——写已即时持久。

**调用链二：OpenSearch 延迟缓冲与刷新代际**
1. `OpenSearchKVStorage.upsert` 把操作压入每进程缓冲。
2. `index_done_callback`（`opensearch_impl.py:1789`）调 `_flush_pending_kv_ops`（:1541）：bulk 写入后 `_write_generation += 1`（:1616）。
3. 仅当欠刷新（`_write_generation != _refreshed_generation`）才 `_refresh_for_search`，并记录 `_refreshed_generation`（:1839）——空闲命名空间提交是 no-op。
4. 瞬时 bulk/refresh 失败抛 `OpenSearchReferencesIntactError`（:1650/:1838），告诉上层"引用没丢、下次重试即可"。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `MONGO_URI`/`MONGO_DATABASE` | Mongo 连接 | `STORAGE_ENV_REQUIREMENTS`（kg/__init__.py:54） |
| `OPENSEARCH_HOSTS` | OpenSearch 连接 | `STORAGE_ENV_REQUIREMENTS`（kg/__init__.py:107） |
| workspace | collection/index 前缀隔离 | 各 impl |

## 5. 错误与重试语义

- **可重试 vs 永久失败分流**：OpenSearch bulk 保留可重试失败（408/429/5xx）缓冲下次 flush；永久 4xx 操作被移出缓冲后抛错——避免重放孤儿行。
- **引用完整性分类**：bulk 调用本身抛错或 `indices.refresh` 抛错都抛 `OpenSearchReferencesIntactError`（证明缓冲操作仍在、未丢引用）；其他异常才被视为"引用可能已丢"。
- **不要求单写者**：两者都是服务端、有行/文档级并发，`requires_single_writer=False`。

## 6. 并发细节

- 多 worker 共享同一服务端，靠数据库自身并发控制。
- OpenSearch 缓冲是**每进程**的：一个进程的提交不捎带别的进程的操作（与 JsonKV 整命名空间快照不同）。
- OpenSearch 近实时模型：bulk 确认的行立即可 `get`（按 id 实时查 translog），但要经 search 可见需 refresh——故 doc-status 写直接带 `refresh="wait_for"`。

## 7. 系统边界

**In-Scope**：`lightrag/kg/mongo_impl.py`、`lightrag/kg/opensearch_impl.py`。

**Out-of-Scope（不在本仓库源码内）**：MongoDB 服务、OpenSearch/Lucene 服务、`pymongo`/`opensearch-py` 客户端。

## 8. 与相邻子系统交互

- **上游**：工厂按 `"MongoKVStorage"`/`"OpenSearchKVStorage"` 等字符串解析。
- **下游**：缓冲/提交语义被 PurgeRecoveryContract 的"LLM 缓存可达性"一节跨层消费（引用先于行、提交先于行）。

## 9. 语言专项适配口径

按 `language-python.md`：多后端矩阵中的服务端实现，与文件型共用 ABC 契约；OpenSearch 的延迟缓冲是生产者-消费者批量，刷新代际是"循环内隐藏状态机"。外部客户端标注"不在本仓库源码内"。

**披露缺口**：Mongo 四类内部的具体查询/索引设计（约 5246 行）与 OpenSearch 图/向量（约 3000 行）未逐行精读，仅以类布局与共性机制覆盖；代表机制（延迟缓冲、刷新代际、引用完整性分类）已覆盖。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 文档/搜索库家族拓扑 | `document-db-storage-architecture.html` | architecture | showcase |
| OpenSearch 缓冲刷新数据流 | `document-db-flush-dataflow.html` | dataflow | showcase |

两图均一次通过 showcase。

省略说明：不补 sequence——OpenSearch 提交时序与 L11 PG 向量提交同构，避免重复。

JSON IR 源文件位于 `json/` 目录。
