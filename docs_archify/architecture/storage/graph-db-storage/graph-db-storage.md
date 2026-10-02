# 图数据库家族（graph-db-storage）

> 本文是 `storage` 域下的叶子子系统文档。域级总览见 `../storage.md`。
> 本文覆盖 Neo4j 与 Memgraph 两家专用图数据库后端——它们只实现 `BaseGraphStorage`（不做 KV/向量/文档状态）。
> 按"目录即实例库"口径覆盖两家，共性说明并披露缺口。
>
> 源码基准：`lightrag` main 分支，commit `453dce83`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Neo4j 图存储 | Cypher 写/查实体关系图 | `lightrag/kg/neo4j_impl.py:79`（`Neo4JStorage`） |
| Memgraph 图存储 | 与 Neo4j 线协议兼容，复用 Cypher 驱动 | `lightrag/kg/memgraph_impl.py:35`（`MemgraphStorage`） |
| 图提交回调 | `index_done_callback` 清理会话/统计 | neo4j:481、memgraph:130 |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `Neo4JStorage` | `neo4j_impl.py:79` | 实现 `BaseGraphStorage`：节点/边 upsert、邻居/度查询、基于标签的检索 |
| `MemgraphStorage` | `memgraph_impl.py:35` | 继承/对齐 Neo4j 线协议，Memgraph 原生兼容 Bolt |

## 3. 关键调用链

**调用链：图节点/边写**
1. LightRAG 提取阶段把实体/关系交给 `upsert_node`/`upsert_edge`。
2. 经 neo4j Python 驱动发 Cypher MERGE 写图。
3. `index_done_callback`（neo4j:481）提交/清理会话，不做延迟缓冲——图写即时落库。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `NEO4J_URI`/`NEO4J_USERNAME`/`NEO4J_PASSWORD` | Neo4j 连接 | `STORAGE_ENV_REQUIREMENTS`（kg/__init__.py:101） |
| `MEMGRAPH_URI`/`MEMGRAPH_USERNAME`/`MEMGRAPH_PASSWORD` | Memgraph 连接 | `STORAGE_ENV_REQUIREMENTS`（kg/__init__.py:105） |

## 5. 错误与重试语义

- 专用图数据库、服务端并发，`requires_single_writer=False`。
- 写失败由驱动抛出，fail loud；不静默成功。

## 6. 并发细节

- 多 worker 共享图数据库，靠数据库自身事务/锁。
- 无本仓库文件型两通道围栏（那是文件存储家族专属）。

## 7. 系统边界

**In-Scope**：`lightrag/kg/neo4j_impl.py`、`memgraph_impl.py`。

**Out-of-Scope（不在本仓库源码内）**：Neo4j 服务、Memgraph 服务、`neo4j` Python 驱动。

## 8. 与相邻子系统交互

- **上游**：工厂按 `"Neo4JStorage"`/`"MemgraphStorage"` 字符串解析。
- **下游**：Cypher 驱动 → 图数据库服务。

## 9. 语言专项适配口径

按 `language-python.md`：多后端矩阵的图存储实现，与文件型 NetworkX、PG AGE 共用 `BaseGraphStorage` 契约；Memgraph 是 Neo4j 线协议兼容实现（双轨兼容）。**披露缺口**：neo4j_impl.py（约 2165 行）的具体 Cypher 语句与索引设计未逐行精读，仅覆盖类布局与写路径。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 图数据库双后端拓扑 | `graph-db-storage-architecture.html` | architecture | showcase |
| 图写 Cypher 提交时序 | `graph-db-write-sequence.html` | sequence | showcase |

两图均一次通过 showcase（时序图末条消息 y 越界后上移通过）。

JSON IR 源文件位于 `json/` 目录。
