# API 路由层（api-routers）

> 本文是 `api` 域下的叶子子系统文档。域级总览见 `../api.md`。
> 本文展开 5 个路由模块的端点划分、请求校验与到 LightRAG 核心的调用桥接；
> 不展开应用装配（见 `../api-server-core/api-server-core.md`）、认证依赖实现（见 `../api-auth/api-auth.md`）。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 文档管理路由工厂 | `create_document_routes(rag, doc_manager, api_key)` 返回 APIRouter，含 17 个端点 | `lightrag/api/routers/document_routes.py:4497` |
| 文档扫描与恢复 | `POST /documents/scan`（异步分类入队）、`GET /documents/scan/status/{track_id}`、`POST /documents/reprocess_failed`、`POST /documents/recovery/force_reset`、`POST /documents/cancel_pipeline` | `document_routes.py:4509`、`:4878`、`:6939`、`:7088`、`:7389` |
| 文档摄入 | `POST /documents/upload`（文件上传）、`POST /documents/text`（单段文本）、`POST /documents/texts`（批量） | `document_routes.py:5225`、`:5560`、`:5691` |
| 文档查询与删除 | `GET /documents/paginated`、`GET /documents/status_counts`、`GET /documents/track_status/{track_id}`、`DELETE /documents/delete_document` | `document_routes.py:6699`、`:6880`、`:6625`、`:5854` |
| 源码冲突处理 | `GET /documents/source_conflicts`、`POST /documents/source_conflicts/repair` | `document_routes.py:4920`、`:4995` |
| 流水线状态 | `GET /documents/pipeline_status`、`GET /documents/supported_file_types` | `document_routes.py:6376`、`:6906` |
| 图谱路由工厂 | `create_graph_routes(rag, api_key)`：12 个图谱读写端点 | `lightrag/api/routers/graph_routes.py:214` |
| 图谱只读 | `/graph/label/list`、`/graph/label/popular`、`/graph/label/search`、`/graphs`、`/graph/entity/exists` | `graph_routes.py:223`、`:238`、`:260`、`:284`、`:320` |
| 图谱写操作 | `/graph/entity/edit`、`/graph/relation/edit`、`/graph/entity/create`、`/graph/relation/create`、`/graph/entities/merge`、两个 DELETE 端点 | `graph_routes.py:341`、`:541`、`:594`、`:675`、`:779`、`:875` |
| 检索查询路由工厂 | `create_query_routes(rag, api_key, top_k)`：3 个查询端点 | `lightrag/api/routers/query_routes.py:367` |
| 同步查询 | `POST /query` → `rag.aquery_llm`，返回 JSON（response + references） | `query_routes.py:376`、`:602` |
| 流式查询 | `POST /query/stream` → SSE 流式返回 token | `query_routes.py:745` |
| 查询数据导出 | `POST /query/data` → 检索到的原始 chunk/实体/关系数据 | `query_routes.py:1115` |
| Ollama 兼容 API | `OllamaAPI` 类：`/api/version`、`/api/tags`、`/api/ps`、`/api/generate`、`/api/chat`（流式），把 LightRAG 包装成 Ollama 服务器 | `lightrag/api/routers/ollama_api.py:381`、`:390` |
| UI 定制路由 | `create_ui_customization_routes(...)`：`GET /`（公开只读定制信息）、`GET /assets/{hash}/{id}` | `lightrag/api/routers/ui_customization_routes.py:46`、`:71` |
| 流水线忙预检 | `check_pipeline_busy_or_raise(rag)`：图谱细粒度写操作前读 `pipeline_status['busy']`，忙则 409；`admin` 持有者豁免 | `document_routes.py:1803` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `create_document_routes(...)` | `document_routes.py:4497` | 工厂：闭包持有 rag/doc_manager/api_key，内部定义 router 与全部端点 |
| `create_query_routes(...)` | `query_routes.py:367` | 工厂：注入 rag 与默认 top_k |
| `create_graph_routes(...)` | `graph_routes.py:214` | 工厂：图谱读写端点 |
| `OllamaAPI` | `ollama_api.py:381` | 类持有 rag/top_k/api_key；`setup_routes()` 在实例 router 上注册 Ollama 协议端点 |
| `create_ui_customization_routes(...)` | `ui_customization_routes.py:46` | 工厂：注入 ui_customization_snapshot |
| `check_pipeline_busy_or_raise(rag)` | `document_routes.py:1803` | 同步预检：pipeline 忙→409；admin 持有者豁免；快照检查，真正的排队在 core 的 `_admin_write_gate` |
| `QueryRequest.to_query_params(stream)` | `query_routes.py:595` | 把 HTTP 请求体转成 `QueryParam` |

## 3. 关键调用链

### 3.1 同步检索请求 `/query`

1. 客户端 `POST /query`（`query_routes.py:376`）→ 先经 `Depends(combined_auth)` 认证（见 L29）。
2. 处理函数把请求体经 `request.to_query_params(False)` 转 `QueryParam`，强制 `param.stream = False`（`query_routes.py:595-599`）。
3. `result = await rag.aquery_llm(request.query, param=param)`（`query_routes.py:602`）——进入 core 域的多模式检索（local/global/hybrid/naive/mix）。
4. 从结果取 `llm_response.content` 与 `data.references`；空内容时替换为"No relevant context found"并标记 `llm_generated=False`（`query_routes.py:615-618`）。
5. 若 `include_references && include_chunk_content`，把 `data.chunks` 按 reference_id 聚合成列表回填（`query_routes.py:621-639`），返回 `QueryResponse`。

### 3.2 文档上传摄入 `/documents/upload`

1. `POST /documents/upload`（`document_routes.py:5225`）→ 认证 → 可选 AdmissionMiddleware 容量预留。
2. 路由保存上传文件到 `input_dir`，调用 `rag.ainsert(...)` 进入文档摄入管线（pipeline 域 L04）。
3. 管线异步入队，路由立即返回 track_id；客户端轮询 `GET /documents/track_status/{track_id}`（`:6625`）或 `/scan/status/{track_id}`（`:4879`）。

### 3.3 图谱编辑写操作 `/graph/entity/edit`

1. `POST /graph/entity/edit`（`graph_routes.py:341`）→ 认证。
2. 调用 `check_pipeline_busy_or_raise(rag)`（`document_routes.py:1803`）：读 `pipeline_status['busy']`，若被 pipeline/destructive job 占用则 409 提前失败（省掉 embedding 往返）；`admin` 持有者豁免。
3. 真正的排队与 reservation 在 core 的 `LightRAG._admin_write_gate` 内完成（仅 NetworkXStorage 这类 `requires_single_writer` 后端），路由预检只是"embedding 之前的快速失败"。

### 3.4 Ollama 兼容 `/api/chat`

1. `OllamaAPI.setup_routes`（`ollama_api.py:390`）注册 `/api/chat`（`:647`）。
2. 请求体经 `_split_chat_messages`（`:94`）把多轮消息拼成 query，`_bound_input_chars`（`:79`）做字符上限，`parse_query_mode`（`:310`）解析 mode 参数。
3. 内部同样走 `rag.aquery`，但响应形状包装成 Ollama 的 `message`/`done` 协议；流式时用 SSE `stream_generator`（`:722`）逐块下发。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `top_k` | 查询默认实体/关系检索数（create_query_routes 传入，env `TOP_K` 默认 60） | `query_routes.py:367` |
| `LIGHTRAG_WORKSPACE` 请求头 | 多工作区隔离；路由从请求头取 workspace 传给 rag（见 L27 `get_workspace_from_request`） | `lightrag_server.py:1864` |
| Ollama 版本号 | 硬编码 `"0.9.3"` 模拟 Ollama 服务器 | `ollama_api.py:397` |
| Ollama 模型元数据 | 来自 `rag.ollama_server_infos`（LIGHTRAG_MODEL/CREATED_AT/SIZE/DIGEST） | `ollama_api.py:404` |
| 查询最小长度 | RAG 模式要求 query 至少英文等价长度 3（CJK 字符计 2）；bypass 模式仅要求非空 | `query_routes.py:576` |

## 5. 错误与重试语义

- **认证失败**：所有端点 `Depends(combined_auth)`，凭据缺失/错误由认证依赖返回 401（见 L29）。
- **流水线忙**：图谱写操作在 embedding 前 409；客户端应稍后重试。FAILED 文档不自动恢复，只能经 `/documents/reprocess_failed` 手动重试一次。
- **参数校验**：`QueryRequest` 模型校验失败由 FastAPI 422；自定义 `validation_exception_handler` 按端点形状返回。
- **未捕获异常**：由 L27 的全局 handler 兜成带 error_id 的 500。
- **Ollama 输入超长**：`_bound_aggregate_text`（`:117`）在 Pydantic 模型层聚合后截断并告警。
- **purge 锚点缺失**：文档删除路径若缺写前锚点，`_purge_kg_contributions` 失败闭合为 HTTP 409（见 PurgeRecoveryContract）。

## 6. 并发细节

- **路由层无共享可变状态**：每个路由处理函数无状态；rag 实例在 lifespan 内初始化，worker 间不共享。
- **预检与提交分离**：`check_pipeline_busy_or_raise` 是请求入口的快照，真正的 reservation 在 core `_admin_write_gate` 内跨请求持有；路由预检不设任何 flag，不会反向阻塞管线。
- **流式响应**：`/query/stream` 与 Ollama `/api/chat` 用 SSE generator，长连接占用 worker 事件循环；取消由客户端断连触发。
- **多工作区**：`LIGHTRAG-WORKSPACE` 头经 `re.sub(r"[^a-zA-Z0-9_]", "_")` 净化（`lightrag_server.py:1884`），非法字符告警后替换。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 5 个路由模块的端点定义、请求/响应模型、到 rag 的调用桥接、流水线忙预检
- Ollama 协议适配层（把 LightRAG 包装成 Ollama 服务器）
- UI 定制只读表面

**Out-of-Scope（不在本仓库源码内）**
- 实际检索/抽取/图存储写入（core 域与 storage 域）
- 摄入管线 worker（pipeline 域）
- 认证密码哈希与准入中间件（L29）
- Ollama 客户端生态（如 `ollama` CLI、Open WebUI 等第三方客户端，`不在本仓库源码内`）

## 8. 与相邻子系统交互

- 上游：FastAPI 应用（L27）`include_router` 五个工厂；客户端/第三方 Ollama 客户端发 HTTP 请求。
- 本叶子 → 下游：
  - 文档路由 → `rag.ainsert` / 管线摄入（pipeline 域）；
  - 查询路由 → `rag.aquery_llm` / `rag.aquery`（core L01 + operate L02）；
  - 图谱路由 → `rag.adelete_by_doc_id` / merge/edit（L15 graph-operations）；
  - 认证依赖来自 L29；配置来自 L30。

## 9. 语言专项适配口径（纯 Python 框架口径）

按 `language-python.md` 落写：

- **工厂函数注入依赖**：五个路由都不是模块级 router，而是 `create_xxx_routes(rag, ...)` 工厂——把 rag 实例闭包注入路由处理函数，与 FastAPI 自带的依赖注入互补。这是纯 Python Web 框架的常见能力缝：路由定义与运行时实例解耦，便于测试替换。
- **配置驱动行为**：Ollama 模型元数据来自 `rag.ollama_server_infos` 配置对象，路由本身不含硬编码模型信息；`parse_query_mode` 把字符串 mode 映射到 `SearchMode` 枚举。
- **可选依赖懒加载**：`check_pipeline_busy_or_raise` 内部 `from lightrag.kg.shared_storage import ...` 函数内 import，避免路由模块加载期就拉起重存储依赖。
- **外部边界**：Ollama 兼容层是对外部协议（Ollama HTTP API）的适配，外部客户端不在本仓库源码内。
- **图型侧重**：architecture（5 路由模块分组 + 端点分类）+ sequence（`/query` 同步请求调用链）；本叶子无独立数据管道（数据管道在 pipeline/operate 域）、无单实体状态机（文档状态机在 pipeline 域），故不补 dataflow/lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 路由层架构图 | `api-routers-architecture.html` | architecture | standard（组件多、跨列连线复杂，showcase 校验报 label-route-clearance，降 standard 渲染成功） |
| 同步查询请求时序 | `api-routers-sequence.html` | sequence | showcase |

本叶子未补 dataflow 图：数据从文档→chunk→向量→图的加工链属于 pipeline 域与 operate 域，路由层只是 HTTP 桥接，无独立变换管道。未补 lifecycle：文档状态机（pending/processing/processed/failed）在 pipeline 域定义，路由只做状态查询透传。

JSON IR 源文件位于 `json/` 目录。
