# 存储契约与后端注册表（storage-contracts）

> 本文是 `storage` 域下的叶子子系统文档。域级总览见 `../storage.md`。
> 本文只展开**四类存储的抽象基类契约、后端注册表与工厂解析**，不展开各后端实现内部
> （文件型家族见 `../file-backed-stores/`，PostgreSQL 见 `../postgres-storage/`，其余后端矩阵见各自叶子）。
>
> 源码基准：`lightrag` main 分支，commit `453dce83`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 存储命名空间基类 | 所有后端的共同抽象层，定义 `initialize`/`finalize`/`index_done_callback`/`drop` 生命周期与缓冲排空钩子 | `lightrag/base.py:193`（`StorageNameSpace`） |
| KV 存储契约 | 键值读写：`get_by_id`/`get_by_id_strict`/`get_by_ids`/`filter_keys`/`upsert`/`delete`/`is_empty` | `lightrag/base.py:535`（`BaseKVStorage`） |
| 向量存储契约 | 嵌入检索与写入：`query`/`upsert`/`get_by_id`/`get_by_ids`/`get_vectors_by_ids`/`delete`/`delete_entity`/`delete_entity_relation`，持有 `embedding_func` | `lightrag/base.py:334`（`BaseVectorStorage`） |
| 图存储契约 | 实体-关系图操作：节点/边的存在性、度数、读写、批量接口、知识库导出；声明 `requires_single_writer` 能力位 | `lightrag/base.py:671`（`BaseGraphStorage`） |
| 文档状态存储契约 | 继承 KV，叠加调度查询：`get_status_counts`/`get_docs_by_statuses`/`get_docs_paginated`/`update_doc_status_fields` | `lightrag/base.py:1558`（`DocStatusStorage`） |
| 文档状态枚举与数据结构 | `PENDING→PARSING→ANALYZING→PROCESSING→PROCESSED|FAILED` 状态机与 `DocProcessingStatus` dataclass | `lightrag/base.py:1317`（`DocStatus`）、`lightrag/base.py:1333`（`DocProcessingStatus`） |
| 后端注册表 | 四类存储名 → 实现类清单 + 每类必需方法校验；实现类名 → 模块路径映射；环境变量依赖清单 | `lightrag/kg/__init__.py:1`（`STORAGE_IMPLEMENTATIONS`）、`:122`（`STORAGES`）、`:51`（`STORAGE_ENV_REQUIREMENTS`） |
| 兼容性校验 | 校验某实现类是否归属某存储类型、是否缺失必需方法 | `lightrag/kg/__init__.py:153`（`verify_storage_implementation`） |
| 后端类工厂 | 把配置字符串（如 `"PGKVStorage"`）解析为具体实现类；四个默认后端直接 import，其余惰性动态导入 | `lightrag/kg/factory.py:17`（`get_storage_class`） |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `StorageNameSpace` | `base.py:193` | 抽象基类。持有 `namespace`/`workspace`/`global_config` 三属性；`index_done_callback` 是**唯一抽象提交点**，docstring 强制后端在抛错时说明"引用是否可能已丢失"（`ReferencesIntactFlushError` 语义） |
| `BaseVectorStorage` | `base.py:334` | dataclass 抽象。`requires_embedding_func: ClassVar[bool]=True` 声明是否需要嵌入函数；`query`/`upsert` 为抽象 |
| `BaseKVStorage` | `base.py:535` | dataclass 抽象。承载 LLM 响应缓存、文本块（`text_chunks`）、文档信息、块追踪（`entity_chunks`/`relation_chunks`）等命名空间 |
| `BaseGraphStorage` | `base.py:671` | 抽象。**`requires_single_writer: ClassVar[bool]=False`** 是关键能力位：仅 `NetworkXStorage` 置 `True`，驱动 `LightRAG._admin_write_gate` 的加锁策略；`discard_uncommitted_mutations()` 默认返回 `False` |
| `DocStatusStorage` | `base.py:1558` | 继承 `BaseKVStorage`，叠加调度面查询；`resolve_status_filter_values`/`_row_points_at_as_duplicate` 等为去重与源冲突判定提供共享逻辑 |
| `DocStatus` / `DocProcessingStatus` | `base.py:1317` / `:1333` | 文档状态枚举（str Enum）与不可变数据结构；`metadata` dict 承载 `kg_write_state`/`kg_purge` 等恢复锚点 |
| `STORAGE_IMPLEMENTATIONS` | `kg/__init__.py:1` | 注册表主表：四类存储各自列出合法实现类名 + `required_methods` |
| `STORAGES` | `kg/__init__.py:122` | 实现类名 → 相对模块路径（如 `"PGKVStorage": ".kg.postgres_impl"`），多后端共享同一模块（mongo/postgres/opensearch 各四个类挤一个文件） |
| `get_storage_class` | `kg/factory.py:17` | 工厂：四个默认后端（JsonKV/NanoVector/NetworkX/JsonDocStatus）直接 import 保证零依赖可用；其余经 `importlib.import_module(STORAGES[name], package="lightrag")` 惰性加载 |

## 3. 关键调用链

**调用链一：配置字符串 → 具体后端类（工厂解析）**
1. `LightRAG` 构造时读 `kv_storage`/`vector_storage`/`graph_storage`/`doc_status_storage` 四个字符串配置。
2. 调用 `get_storage_class(storage_name)`（`kg/factory.py:17`）。
3. 若是四个默认后端之一，直接 `from lightrag.kg.<impl> import <Class>` 返回（`kg/factory.py:19-34`）——这是为了不依赖注册表即可开箱即用。
4. 否则查 `STORAGES[name]` 得到模块相对路径，`importlib.import_module(import_path, package="lightrag")` 惰性导入后 `getattr(module, name)` 返回类（`kg/factory.py:40-42`）。
5. 实例化后由 `LightRAG` 统一 `await rag.initialize_storages()` 逐个 `await storage.initialize()`。

**调用链二：存储类型兼容性校验**
1. 配置装配阶段调用 `verify_storage_implementation(storage_type, storage_name)`（`kg/__init__.py:153`）。
2. 先查 `storage_type in STORAGE_IMPLEMENTATIONS`，未知类型抛 `ValueError`（`kg/__init__.py:163`）。
3. 再查 `storage_name in storage_info["implementations"]`，不兼容则列出全部合法实现抛错（`kg/__init__.py:167-171`）。

**调用链三：提交点语义（跨后端统一约定）**
1. 每个后端在批处理结束时被调用 `index_done_callback()`（`base.py:207`，抽象方法）。
2. 即时写后端（PostgreSQL/Redis/Mongo/Neo4j 等）此方法为 `pass` 或近似空操作——写入在 `upsert` 时已落库。
3. 延迟写后端（`JsonKVStorage`、`NanoVectorDBStorage`、`FaissVectorDBStorage`、`OpenSearchKVStorage`）在此真正刷盘/刷缓冲。
4. 抛错时若后端能证明"缓冲操作一条没丢"，必须抛 `ReferencesIntactFlushError` 子类；否则调用方（`LightRAG._flush_storages`）按"引用可能已丢"隔离缓存行（见 PurgeRecoveryContract.md 的 LLM 缓存可达性一节）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `kv_storage` | `"JsonKVStorage"`，可选值见 `STORAGE_IMPLEMENTATIONS["KV_STORAGE"]` | `kg/__init__.py:2` |
| `vector_storage` | `"NanoVectorDBStorage"`，可选 8 个（含 `NoopVectorDBStorage` 空实现） | `kg/__init__.py:24` |
| `graph_storage` | `"NetworkXStorage"`，可选 7 个 | `kg/__init__.py:12` |
| `doc_status_storage` | `"JsonDocStatusStorage"`，可选 5 个 | `kg/__init__.py:38` |
| `STORAGE_ENV_REQUIREMENTS` | 每个非文件后端声明必需环境变量（如 PG 三件套、Neo4j 三件套、Mongo 两件套），缺失即启动失败 | `kg/__init__.py:51` |
| `workspace` | 工作区隔离：文件型用子目录、collection 型用前缀、关系型 DB 用列过滤、Qdrant 用 payload 分区 | `base.py:194`（属性约定） |

## 5. 错误与重试语义

- **未知存储类型 / 不兼容实现**：`verify_storage_implementation` 直接抛 `ValueError`，启动期失败、fail loud，不降级。
- **惰性导入失败**：`get_storage_class` 对非默认后端用 `importlib.import_module`，缺依赖（如未装 `psycopg`/`pymongo`）时在导入点抛 `ImportError`——这是纯 Python 框架"可选依赖 + 懒加载"口径：顶层 `import lightrag` 不强加载重后端。
- **提交失败语义**：`index_done_callback` 是唯一把缓冲变持久的点；其异常分类（`ReferencesIntactFlushError` vs 其他）决定上层是否隔离缓存行（契约见 `base.py:207` docstring 与 PurgeRecoveryContract.md）。
- **`drop_pending_index_ops`**：批处理中止时由 pipeline 调用，延迟写后端清缓冲；即时写后端保持默认 `None`/no-op。

## 6. 并发细节

- 本叶子只定义契约与注册表，**不持有锁、不管理并发**；并发语义下沉到各后端（文件型家族的两通道围栏见 `../file-backed-stores/`，跨进程原语见 `../storage-coordination/`）。
- `BaseGraphStorage.requires_single_writer` 是一个 `ClassVar[bool]`（`base.py:690`），故意用 ClassVar 而非实例属性——因为存储基类是 dataclass，裸注解会变成 `__init__` 字段、改变每个后端的构造签名。该位是 admin 写门加锁策略的唯一开关。
- `discard_uncommitted_mutations`（`base.py:692`）刻意设计为**同步方法**：它跑在已取消的任务上，每个 `await` 都是二次被中断的窗口，普通属性写入不会。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/base.py`：全部抽象基类、数据结构、枚举。
- `lightrag/kg/__init__.py`：注册表与兼容性校验。
- `lightrag/kg/factory.py`：类工厂。

**Out-of-Scope（不在本仓库源码内）**
- 所有后端实现内部：NetworkX/PostgreSQL/MongoDB/Neo4j/Redis/Milvus/Qdrant/Faiss/Memgraph/OpenSearch——这些是"实现方"，各自归对应叶子。
- 外部数据库进程本身（PostgreSQL 服务、MongoDB 集群等）一律不在本仓库源码内。
- pipeline 调度与 admin 写门如何消费 `requires_single_writer`：归 `../graph-operations/` 与 core 域。

## 8. 与相邻子系统交互

- **上游（core 域）**：`LightRAG` 构造函数读四个存储字符串配置 → `get_storage_class` 解析 → 实例化 → `initialize_storages()` 统一初始化。
- **下游（storage 域各叶子）**：本叶子定义的四个 ABC 是所有后端的"消费方接口"——实现方大量注册、契约在消费方基类（纯 Python 框架的能力缝模式）。
- **协作**：`base.py:207` 的 `index_done_callback` 异常分类语义被 `LightRAG._flush_storages` 与 LLM 缓存可达性契约跨层消费。

## 9. 语言专项适配口径

按 `language-python.md` 口径，本叶子是纯 Python 框架的**核心能力缝**：

1. **抽象基类体系（接口在消费方）**：四个 ABC 定义契约，20+ 后端为实现方——典型"接口定义在消费方基类、实现方大量注册"。
2. **注册表 + 工厂字符串路由**：`STORAGES` 映射表是"名称 → 模块路径"的纯字符串表；`get_storage_class` 是运行时字符串路由（类似 transformers 的 Auto 模式）。注册时机为**模块导入时静态填表**（非装饰器、非 import 副作用动态注入）。
3. **懒加载 + 可选依赖**：`factory.py` 对非默认后端 `importlib.import_module` 惰性加载，顶层 import 不强制加载 `psycopg`/`pymongo` 等重依赖；四个默认后端直接 import 保证零依赖开箱即用——双轨实现口径（纯文件兜底 + 外部服务后端）。
4. **配置驱动静态分派**：运行时后端选择由四个配置字符串决定，同一 `LightRAG` 代码经不同配置切换后端，是框架级静态多态核心。
5. **多后端矩阵**：四类存储 × 多实现，外部数据库库（psycopg/pymongo/neo4j driver 等）标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 存储契约分层与后端矩阵 | `storage-contracts-architecture.html` | architecture | standard |
| 后端类工厂解析时序 | `storage-contracts-resolution-sequence.html` | sequence | showcase |

架构图落 standard：首轮 showcase 校验报告两处边标签与组件重叠（`label_route_clearance`），已删除两条垂直堆叠边的文案标签（c2 工厂→注册表、c10 PG→外部）后通过渲染；组件跨层连接较密，按系统级口径接受 standard。时序图一次通过 showcase。

JSON IR 源文件位于 `json/` 目录。
