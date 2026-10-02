# 向量与缓存数据库矩阵（vector-db-matrix）

> 本文是 `storage` 域下的叶子子系统文档。域级总览见 `../storage.md`。
> 本文按"目录即实例库"口径覆盖 Milvus、Qdrant（向量库）与 Redis（缓存/KV+文档状态）三家后端矩阵。
> 深读共性协议（延迟嵌入缓冲），逐家差异以表格说明并披露缺口。
>
> 源码基准：`lightrag` main 分支，commit `453dce83`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Milvus 向量 | Milvus collection 存嵌入；集合门控 + 索引配置 | `lightrag/kg/milvus_impl.py:729`（`MilvusVectorDBStorage`） |
| Qdrant 向量 | Qdrant collection；workspace 用 payload 分区隔离 | `lightrag/kg/qdrant_impl.py:178`（`QdrantVectorDBStorage`） |
| Redis KV | Redis 缓存 KV（LLM 响应缓存等） | `lightrag/kg/redis_impl.py:390`（`RedisKVStorage`） |
| Redis 文档状态 | Redis 调度状态 | `lightrag/kg/redis_impl.py:937`（`RedisDocStatusStorage`） |
| 延迟嵌入缓冲 | 三家向量后端均 `upsert` 时 `vector=None`，提交时批量嵌入 | `_PendingVectorDoc`（milvus:295、qdrant:26） |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `MilvusVectorDBStorage` | `milvus_impl.py:729` | Milvus collection 向量；`_CollectionGate`(:193) 集合并发门控、`MilvusIndexConfig`(:472) 索引参数 |
| `QdrantVectorDBStorage` | `qdrant_impl.py:178` | Qdrant；workspace 经 payload 过滤隔离（AGENTS.md 明确口径） |
| `RedisKVStorage` | `redis_impl.py:390` | 即时写缓存；`RedisConnectionManager`(:259) 连接池 |
| `RedisDocStatusStorage` | `redis_impl.py:937` | 调度状态 |
| `_PendingVectorDoc` | milvus:295 / qdrant:26 | 延迟嵌入缓冲记录（content + vector=None） |

## 3. 关键调用链

**调用链：延迟嵌入（三家向量后端共性）**
1. `upsert` 把记录压入 `_PendingVectorDoc` 缓冲，`vector=None`，不调嵌入模型。
2. `index_done_callback`（milvus:2855 / qdrant:838）批量嵌入后一次性写 collection。
3. Redis 是即时写：`upsert` 直接写 Redis，`index_done_callback`（redis:780）近似 no-op。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `MILVUS_URI`/`MILVUS_DB_NAME` | Milvus 连接 | `STORAGE_ENV_REQUIREMENTS`（kg/__init__.py:86） |
| `QDRANT_URL`/`QDRANT_API_KEY` | Qdrant 连接（key 默认 None） | `STORAGE_ENV_REQUIREMENTS`（kg/__init__.py:93） |
| `REDIS_URI` | Redis 连接 | `STORAGE_ENV_REQUIREMENTS`（kg/__init__.py:58） |
| workspace | collection 前缀 / payload 分区 | 各 impl |

## 5. 错误与重试语义

- 三家均为服务端、行/记录级并发，`requires_single_writer=False`。
- Milvus 用 `_CollectionGate` 控制集合级并发；Qdrant 靠服务端。
- 连接失败由客户端库重试，本仓库不额外包退避。

## 6. 并发细节

- 多 worker 共享服务端，靠数据库自身并发。
- 向量缓冲是每进程内存，提交时批量落库。
- Qdrant 的 workspace 隔离是 payload 过滤（区别于文件型子目录、关系型列）。

## 7. 系统边界

**In-Scope**：`lightrag/kg/milvus_impl.py`、`qdrant_impl.py`、`redis_impl.py`。

**Out-of-Scope（不在本仓库源码内）**：Milvus 服务、Qdrant 服务、Redis 服务、各自 Python 客户端。

## 8. 与相邻子系统交互

- **上游**：工厂按字符串解析到本叶子类。
- **下游**：嵌入模型来自 LLM 域；外部向量/缓存服务。

## 9. 语言专项适配口径

按 `language-python.md`：多后端矩阵的服务端实现，共用 `BaseVectorStorage`/`BaseKVStorage` 契约；延迟嵌入是生产者-消费者批量。**披露缺口**：Milvus（约 3626 行）与 Redis（约 2860 行）的具体 collection/索引/序列化设计未逐行精读，仅覆盖类布局与共性延迟嵌入协议；这是"目录即实例库"型叶子的预期取舍。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 向量与缓存后端矩阵 | `vector-db-matrix-architecture.html` | architecture | showcase |
| 延迟嵌入共性数据流 | `vector-db-deferred-dataflow.html` | dataflow | showcase |

两图均一次通过 showcase。

省略说明：不补 sequence——延迟嵌入提交时序与 L11 同构，避免重复。

JSON IR 源文件位于 `json/` 目录。
