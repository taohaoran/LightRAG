# LightRAG 主类核心编排（lightrag-core）

> 本文是 `core` 域下的叶子子系统文档。域级总览见 `../core.md`。
> 本文展开 LightRAG 主类的组合结构、存储生命周期、SDK 入口与 admin 写门；
> 实体/关系提取与检索的具体算法见 `../operate-retrieval/operate-retrieval.md`，
> 角色化 LLM 配置见 `../llm-roles/llm-roles.md`，文档摄入管线细节见 `../../pipeline/pipeline-mixin/pipeline-mixin.md`。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| LightRAG 主类组装 | `@final @dataclass` 类，由三个 mixin 分层组合，对外暴露统一公共 API | `lightrag/lightrag.py:684` |
| 存储实例句柄装配 | 构造期解析 4 类存储后端字符串为 12 个存储句柄（KV/向量/图/文档状态） | `lightrag/lightrag.py:696-706` |
| 存储初始化 | 逐个 `await storage.initialize()`，设置 owning loop 与 pipeline_status | `lightrag/lightrag.py:1986` |
| 存储终结 | 缓存对提交、隔离脏缓存、关闭解析线程池、置 FINALIZED | `lightrag/lightrag.py:2163` |
| SDK 文档写入入口 | `ainsert`（固定 token 分块 F 策略）→ enqueue + process 两步 | `lightrag/lightrag.py:2310` |
| SDK 查询入口 | `aquery` 为 `aquery_llm` 的向后兼容包装，只返回 LLM 文本 | `lightrag/lightrag.py:4843` |
| 自定义 KG 直写 | `ainsert_custom_kg`：调用方构造实体/关系直接入图，走 admin 写门 | `lightrag/lightrag.py:4340` |
| Admin 写门 | `_admin_write_gate`：仅当图存储 `requires_single_writer` 时串行化 admin 写 | `lightrag/lightrag.py:7564` |
| 可观测附加参数 | `ObservableAddonParams`：dict 子类，变更时回调刷新缓存 | `lightrag/addon_params.py:107` |
| 数据迁移 | 启动期 `check_and_migrate_data` 迁移实体/关系与 chunk 追踪 | `lightrag/storage_migrations.py:31` |
| 提取入口编排 | `_process_extract_entities` 驱动单 chunk 的实体/关系提取 | `lightrag/lightrag.py:3762` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `LightRAG` | `lightrag/lightrag.py:684` | 主编排类；继承 `_RoleLLMMixin`→`_StorageMigrationMixin`→`_PipelineMixin` |
| `StoragesStatus` | `lightrag/base.py:2060` | 存储生命周期枚举：`NOT_CREATED/CREATED/INITIALIZED/FINALIZED` |
| `ObservableAddonParams` | `lightrag/addon_params.py:107` | 可观察 dict；`__setitem__` 等写操作触发 `_changed()` 回调 |
| `_AdminHoldCeiling` | `lightrag/lightrag.py:370` | admin 写挂起时限上下文；超时取消在飞提交，读取消邮票上报 |
| `_PurgeRecoveryProof` | `lightrag/lightrag.py:531` | purge 恢复四证之一（anchors/pre_graph/journal/empty_scope） |
| `_StorageMigrationMixin` | `lightrag/storage_migrations.py:22` | 迁移 mixin：`check_and_migrate_data` 等 |
| `default_addon_params` / `normalize_addon_params` | `lightrag/addon_params.py:41,60` | 附加参数默认值与归一化 |

## 3. 关键调用链

**调用链 A：存储初始化（`lightrag.py:1986` `initialize_storages`）**
1. 进入时断言 `_storages_status == CREATED`（由 `__post_init__` 末置位，`lightrag.py:1984`），记录 `_owning_loop = asyncio.get_running_loop()`。
2. `initialize_pipeline_status(workspace=...)` 为本工作区自动初始化调度状态。
3. 依次对 12 个存储句柄（`full_docs`、`text_chunks`、`full_entities`、`full_relations`、`entity_chunks`、`relation_chunks`、`entities_vdb`、`relationships_vdb`、`chunks_vdb`、`chunk_entity_relation_graph`、`llm_response_cache`、`doc_status`）逐个 `await storage.initialize()`——逐个初始化是为避免跨后端死锁。
4. `enforce_strict_storage_capabilities(...)` 按后端启动期真实能力判定严格读，最后置 `_storages_status = INITIALIZED`。

**调用链 B：SDK 写入（`lightrag.py:2310` `ainsert`）**
1. 生成 `track_id`；`resolve_chunk_options` 把 F 策略运行时参数（`split_by_character` 等）收敛为 `chunk_options` 快照。
2. `await self.apipeline_enqueue_documents(...)`（来自 `_PipelineMixin`）入队。
3. `await self.apipeline_process_enqueue_documents()` 同步驱动处理，返回 track_id。
4. 注意：REST/服务端**不调用**本方法，而是直接走 enqueue+process 以选择 R/V/P/C 分块策略。

**调用链 C：admin 写门（`lightrag.py:7564` `_admin_write_gate`，生成器上下文）**
1. `_admin_write_gate_required()` 为假时（图存储非 NetworkX）直接 `yield` 放行。
2. 先取 **admin lock**（等待、有界超时 `ADMIN_WRITE_LOCK_ACQUIRE_TIMEOUT`），超时抛 `AdminWriteGateRefusedError`（fence=admin_lock）。
3. 再向 `pipeline_status` 申请 **busy 预约**（`owner_kind="admin"`，`reject_when=(busy/scanning)` 立即拒绝，不等待）。
4. 进入业务体（第八个 admin 写手 `ainsert_custom_kg` 在 `lightrag.py:4373` `async with self._admin_write_gate(...)` 内执行图写与 commit）。
5. `finally` 释放预约与锁，并 `_drive_pipeline_if_deferred` 唤醒被推迟的管线启动。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `working_dir` | `./rag_storage`，缓存与临时目录 | `lightrag.py:690` |
| `kv_storage` / `vector_storage` / `graph_storage` / `doc_status_storage` | 默认 `JsonKVStorage` / `NanoVectorDBStorage` / `NetworkXStorage` / `JsonDocStatusStorage`（文件型小数据后端） | `lightrag.py:696-705` |
| `workspace` | env `WORKSPACE`，默认空串，用于数据隔离 | `lightrag.py:711` |
| `top_k` / `chunk_top_k` / `max_*_tokens` | env 驱动，查询 token 预算 | `lightrag.py:722-743` |
| `admin_write_max_hold_seconds` | 默认按 embedding timeout 推导；仅 NetworkX 后端生效 | `lightrag.py:1679` 附近 |
| `pipeline_scheduling_page_size` / `max_pending_documents` | 0 表示禁用分页/准入；负值 fail-fast | `lightrag.py:1658-1672` |

## 5. 错误与重试语义

- **初始化陷阱**：未 `await initialize_storages()` 即使用，典型表现为 `AttributeError: __aenter__` 或 `KeyError: 'history_messages'`。
- **admin 写门失败**：等待 admin lock 超时 → 409（`ADMIN_WRITE_LOCK_BUSY_PREFIX`）；管线 busy/scanning → 立即 409（`ADMIN_WRITE_PIPELINE_BUSY_PREFIX`）；两者以前缀区分。
- **可接受残留**：挂起时限取消一个在飞写、允许已提交的 commit 完成——被报告为失败的操作可能已持久化；恢复方式是重读对象而非盲目重试（盲目重试会变成 "entity already exists"）。
- **`ainsert_custom_kg` 警告**：此路径在文档级恢复保证**之外**，无持久操作日志、不预写 `full_entities/full_relations` 锚点；崩溃后只能靠离线工具 `kg_integrity_repair` 事后重建锚点。

## 6. 并发细节

- **mixin 分层固定顺序**：`LightRAG → _RoleLLMMixin → _StorageMigrationMixin → _PipelineMixin → object`，`@final` 禁止外部子类化。
- **owning loop 绑定**：初始化时记录 `_owning_loop`，同步包装器若从别的 loop（`run_in_executor`/线程）驱动则快速失败。
- **admin 写门获取顺序固定**：admin lock（等待）→ busy 预约（拒绝）→ per-entity keyed locks；per-entity 锁必须**在** admin 门内，`amerge_entities` 一次取多把，反序会死锁。
- **延迟管线驱动**：挂起期间被推迟的管线启动在释放时 `_drive_pipeline_if_deferred` 驱动一次。
- 存储逐个 `initialize()` 是为避免跨后端死锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/lightrag.py` 主类组合、入口编排、admin 写门
- `lightrag/addon_params.py` 可观察附加参数
- `lightrag/storage_migrations.py` 启动期数据迁移
- `lightrag/types.py` Pydantic 提取结果模型

**Out-of-Scope（不在本仓库源码内）**
- 各存储后端实现细节（Neo4j/PostgreSQL/Redis/Milvus 等外部数据库服务）——见 storage 域
- LLM/embedding 实际推理计算（OpenAI/Ollama/Gemini 等远端服务）——见 llm 域
- FastAPI 服务端路由装配——见 api 域

## 8. 与相邻子系统交互

- 上游：SDK 用户 / `lightrag.api` 服务端 → `ainsert` / `aquery` / `ainsert_custom_kg`
- 下游（摄入方向）：`LightRAG` → `_PipelineMixin.apipeline_enqueue_documents` → pipeline 域（L04–L07）
- 下游（检索方向）：`aquery_llm` → `operate.py` 多模式检索（L02）
- 横切：`LightRAG` → `llm_roles._RoleLLMMixin`（L03）提供角色化 LLM 包装器；→ `kg/factory.get_storage_class` 解析后端

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝**：`LightRAG` 是编排/适配层，本身不做实体计算；实际提取/检索委托给 operate 域、存储后端与外部 LLM。
- **mixin 组合 = 静态混入式能力缝**：三个 mixin 按固定顺序水平切分关注点（角色配置 / 迁移 / 管线），`@final` 关闭了运行时子类扩展面，扩展点在存储注册表与 LLM 角色注册表而非子类化。
- **配置驱动分派**：构造参数 `kv_storage` 等字符串 → `kg/factory.get_storage_class()` 路由到具体后端类，同一主类代码经不同配置切换后端。
- **可观察参数钩子链**：`ObservableAddonParams` 写操作触发 `_changed` → `_refresh_addon_params_cache`，是框架级可插拔回调缝。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 主类组合与存储装配架构图 | `lightrag-core-architecture.html` | architecture | standard（边界贴近画布边、factory→存储四路竖线需显式 fromSide/toSide 路由；已加 viewBox、钉死边侧） |
| 存储生命周期状态机 | `lightrag-core-lifecycle.html` | lifecycle | standard（viewBox 高度须 ≥566；已加高画布） |
| Admin 写门获取时序 | `lightrag-core-sequence.html` | sequence | standard（参与者标签超 86px 框、末消息 y 越界；已缩短参与者名、压缩时间轴） |

本叶子补 lifecycle（`StoragesStatus` 四态迁移是显式状态机）与 sequence（admin 写门三方交互）；不补 dataflow（数据管道归属 pipeline 域，避免与 L04 重复）。JSON IR 位于 `json/`。三张图均一次 standard 渲染通过、HTML >790KB。
