# storage 域总览

> 本域是 LightRAG 的持久化层：定义四类存储抽象契约，并在它们之上提供多套可插拔后端实现、
> 跨进程协调原语、文件型快照一致性协议，以及图库管理员写操作与 Purge 恢复。
>
> 源码基准：`lightrag` main 分支，commit `453dce83`。`lightrag/kg/` 合计约 50,863 行。

## 域职责

LightRAG 把持久化拆成**四类存储抽象**，每类都有一组可插拔后端：

- **KV_STORAGE**：LLM 响应缓存、文本块（text_chunks）、文档信息。
- **VECTOR_STORAGE**：实体 / 关系 / 文本块的嵌入向量。
- **GRAPH_STORAGE**：实体-关系图结构。
- **DOC_STATUS_STORAGE**：文档处理状态（调度/恢复锚点）。

一个 `LightRAG` 实例用 `workspace` 参数做数据隔离；隔离方式按后端形态分化：
文件型用子目录、collection 型用前缀、关系型 DB 用列过滤、Qdrant 用 payload 分区。

## 域级核心机制

### 1. 四类存储抽象 + 后端注册表

抽象基类在 `lightrag/base.py`；后端注册表与工厂在 `kg/__init__.py` / `kg/factory.py`
（`STORAGE_IMPLEMENTATIONS`/`STORAGES`、`get_storage_class()`）。字符串配置（如 `kv_storage="PGKVStorage"`）
在运行时解析到具体类。详见叶子 [storage-contracts](#)。

### 2. 一致性：无事务跨存储

跨四类存储**无事务**，每次多存储写都存在中间态。LightRAG 的取舍是：
**一个不一致残留只有在"会自愈（重试/重建）或方向无害（多保留对象比丢数据好）"时才可接受**；
丢失数据永远不可接受；残留必须写进文档；失败必须 fail loud，禁止静默成功。

### 3. 文件型五存储家族与两通道围栏

五个文件型存储在内存持有数据、靠整文件重写发布：

- **NetworkX / NanoVectorDB / Faiss**：进程内单副本，靠**两通道围栏**检测磁盘被越写——
  文件自身 `(st_mtime_ns, st_size)`（权威状态）OR `storage_updated` 标志（加速事件）。
  写冲突时：**图 store 拒绝提交**（图载荷"读上累加"，无 buffer 可重放），
  **向量 store 重载并重放** pending buffer + redo log。
- **JsonKV / JsonDocStatus**：用 `Manager().dict()` 全进程共享，**`storage_updated` 语义相反**——
  `True` 表示"脏数据待刷盘"，而非"磁盘有新数据要重载"；无 reload 机制。
- **NetworkX 唯一声明 `requires_single_writer=True`**：admin 图写在 `LightRAG._admin_write_gate` 下串行化。

详见叶子 [file-backed-stores](#)。

### 4. Purge 恢复失败闭合

Purge 删除某文档贡献时**失败闭合**：除非四证之一（`anchors` / `pre_graph` / `journal` / `empty_scope`），
否则抛 `RecoveryAnchorMissingError`（HTTP 409）、零删除。`kg_write_state` 只写一次、单调、禁止回填。
详见叶子 [graph-operations](#)。

### 5. 关系权重契约

图边上 `weight ≥ 去重真实 source_id 数`；空串与 `manual_creation`/`UNKNOWN` 不算证据。

## 叶子索引

| 叶子 | 职责 | 主要源码 |
|---|---|---|
| [storage-contracts](storage-contracts/storage-contracts.md) | 四类存储 ABC 契约 + 后端注册表 + 工厂解析 | `base.py`、`kg/__init__.py`、`kg/factory.py` |
| [storage-coordination](storage-coordination/storage-coordination.md) | 命名空间锁、`storage_updated` 围栏、pipeline_status、写序号 | `kg/shared_storage.py`、`namespace.py`、`kg/write_seq.py` |
| [file-backed-stores](file-backed-stores/file-backed-stores.md) | 文件型五存储、两通道围栏、单写者门 | `networkx_impl.py`、`nano_vector_db_impl.py`、`faiss_impl.py`、`json_kv_impl.py`、`json_doc_status_impl.py`、`file_fingerprint.py` |
| [postgres-storage](postgres-storage/postgres-storage.md) | PG 家族（KV/向量/AGE 图/表图/文档状态） | `postgres_impl.py`、`pgtable_impl.py` |
| [document-db-storage](document-db-storage/document-db-storage.md) | MongoDB + OpenSearch 家族 | `mongo_impl.py`、`opensearch_impl.py` |
| [vector-db-matrix](vector-db-matrix/vector-db-matrix.md) | Milvus / Qdrant 向量 + Redis 缓存 | `milvus_impl.py`、`qdrant_impl.py`、`redis_impl.py` |
| [graph-db-storage](graph-db-storage/graph-db-storage.md) | Neo4j / Memgraph 专用图库 | `neo4j_impl.py`、`memgraph_impl.py` |
| [graph-operations](graph-operations/graph-operations.md) | admin 图写、合并、Purge 锚点、关系权重 | `utils_graph.py` |

## 域级图表

| 图 | 文件 | 类型 |
|---|---|---|
| 四类存储 × 后端矩阵 | `storage-architecture.html` | architecture |
| 跨存储写入数据流 | `storage-write-dataflow.html` | dataflow |
| 跨存储写入时序 | `storage-write-sequence.html` | sequence |
