# PostgreSQL 家族存储（postgres-storage）

> 本文是 `storage` 域下的叶子子系统文档。域级总览见 `../storage.md`。
> 本文展开基于 PostgreSQL 的四类存储实现：KV、向量（pgvector）、图（Apache AGE 与关系表双轨）、文档状态。
> 契约层见 `../storage-contracts/`，跨进程原语见 `../storage-coordination/`。
>
> 源码基准：`lightrag` main 分支，commit `453dce83`。postgres_impl.py 约 10032 行，本文定点引用。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 底层连接池 | `asyncpg` 连接池封装，工作区隔离用列过滤 | `lightrag/kg/postgres_impl.py:420`（`PostgreSQLDB`） |
| KV 存储 | `text_chunks`/`llm_response_cache`/块追踪等命名空间，jsonb 列 | `lightrag/kg/postgres_impl.py:3121`（`PGKVStorage`） |
| 向量存储 | pgvector 列嵌入；延迟嵌入缓冲 `_PendingPGVectorDoc` | `lightrag/kg/postgres_impl.py:3867`（`PGVectorStorage`）、`:3852` |
| 文档状态存储 | 调度状态表，分页/计数/源冲突 | `lightrag/kg/postgres_impl.py:5517`（`PGDocStatusStorage`） |
| 图存储（AGE 轨） | Apache AGE 图扩展，Cypher 风格查询；边写丢失可检测 | `lightrag/kg/postgres_impl.py:7269`（`PGGraphStorage`） |
| 图存储（关系表轨） | 不用 AGE，用普通关系表存节点/边，带重试与分片 | `lightrag/kg/pgtable_impl.py:267`（`PGTableGraphStorage`） |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `PostgreSQLDB` | `postgres_impl.py:420` | 连接池、DDL 初始化、工作区列过滤 |
| `ClientManager` | `postgres_impl.py:2869` | 客户端/连接生命周期管理 |
| `PGKVStorage` | `postgres_impl.py:3121` | 即时写后端：`index_done_callback`(:3750) 近似空操作，写在 `upsert`(:3512) 落库 |
| `PGVectorStorage` | `postgres_impl.py:3867` | 与文件型一样的延迟嵌入协议：`upsert`(:4595) 缓冲 `vector=None`，`index_done_callback`(:4948) 时嵌入 |
| `PGDocStatusStorage` | `postgres_impl.py:5517` | 调度状态写即时落库；`index_done_callback`(:6875) |
| `PGGraphStorage` | `postgres_impl.py:7269` | AGE 图；`requires_single_writer=False`（行级并发，无需 admin 写门串行化） |
| `PGGraphEdgeWriteLostError` | `postgres_impl.py:7154` | 边写在 AGE 扩展中丢失的可检测异常 |
| `PGTableGraphStorage` | `pgtable_impl.py:267` | 关系表图；`_with_retry`(:435) 处理瞬时错误、`_chunked_delete`(:393) 分片删除 |

## 3. 关键调用链

**调用链一：PG 向量延迟嵌入（与文件型同协议）**
1. `upsert`（`postgres_impl.py:4595`）把记录压入 `_PendingPGVectorDoc`（:3852），`vector=None`。
2. 批末 `index_done_callback`（:4948）批量调嵌入模型后一次性 upsert 进 pgvector 列。
3. 同一 id 多次 upsert 只嵌入一次。

**调用链二：PG KV 即时写**
1. `upsert`（:3512）直接 INSERT ... ON CONFLICT 写 jsonb 行。
2. `index_done_callback`（:3750）近似 no-op——无缓冲需要刷。

**调用链三：AGE 图写**
1. `upsert_node`（:7928）/`upsert_edge`（:7984）写 AGE 图。
2. `PGGraphEdgeWriteLostError`（:7154）捕获边写丢失——与文件型"拒绝提交"不同，PG 靠事务/行级并发保证，失败即时抛出。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `POSTGRES_HOST/PORT/USER/PASSWORD/DATABASE` | 连接串 | `STORAGE_ENV_REQUIREMENTS`（kg/__init__.py:59） |
| `workspace` | 列过滤隔离 | `PostgreSQLDB` 初始化 |
| `cosine_better_than_threshold` | 向量检索阈值 | LightRAG 构造参数 |

## 5. 错误与重试语义

- **瞬时错误重试**：`PGTableGraphStorage._with_retry`（`pgtable_impl.py:435`）对瞬态错误退避重试；`_is_transient_error`（:44）判定。
- **边写丢失可检测**：AGE 轨用 `PGGraphEdgeWriteLostError` 而非静默成功。
- **无跨存储事务**：图/KV/向量/文档状态分表，跨表写无事务；一致性靠"自愈或方向无害"残留（见域总览）。
- **不要求单写者**：PG 是服务端、有行级并发，`requires_single_writer=False`，admin 写不经 `_admin_write_gate` 串行化。

## 6. 并发细节

- 多 worker 进程共享同一 PostgreSQL 服务，靠数据库行锁/事务，不靠本仓库文件锁。
- 向量延迟嵌入缓冲是每进程内存，提交时一次性落 pgvector。
- 工作区隔离靠 `workspace` 列过滤，collection 名加前缀。

## 7. 系统边界

**In-Scope**：`lightrag/kg/postgres_impl.py`、`lightrag/kg/pgtable_impl.py`。

**Out-of-Scope（不在本仓库源码内）**：PostgreSQL 服务进程、`asyncpg` 驱动、pgvector 扩展、Apache AGE 扩展本身。

## 8. 与相邻子系统交互

- **上游**：LightRAG 工厂按 `kv_storage="PGKVStorage"` 等字符串解析到本叶子类。
- **下游**：连接池 → PostgreSQL 服务；嵌入函数来自 LLM 域。

## 9. 语言专项适配口径

按 `language-python.md`：本叶子是多后端矩阵中的**服务端实现**，与文件型实现共用 `BaseKVStorage`/`BaseVectorStorage`/`BaseGraphStorage` 契约；双轨图实现（AGE vs 关系表）是同能力两实现，由配置切换。外部 `asyncpg`/pgvector/AGE 标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| PG 家族四类存储与连接池 | `postgres-storage-architecture.html` | architecture | showcase |
| PG 向量延迟嵌入提交时序 | `postgres-storage-flush-sequence.html` | sequence | showcase |

两图均一次通过 showcase。时序图首轮自消息"缓冲 vector=None"跨度不足（0px < 60px）且末条 y 越界，改为把缓冲语义并入 m1 文案、收末条纵坐标后通过。

省略说明：不补 lifecycle——文档状态状态机已由域级/文档状态契约表达，避免重复。

JSON IR 源文件位于 `json/` 目录。
