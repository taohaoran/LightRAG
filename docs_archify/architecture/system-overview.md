# LightRAG 系统级总览

> 本文是 LightRAG 项目的系统级架构总览，基于源码（`lightrag-hku`，branch `main`，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`，2026-09-26）深度分析产出。
> 覆盖 9 个域、38 个叶子子系统。各域详情见对应域目录，各叶子详情见对应叶子文档。

## 1. 项目概述

**定位**：LightRAG 是一个基于图的轻量级检索增强生成（RAG）框架，通过从文档中提取实体与关系构建知识图谱，结合多种检索模式（`local` / `global` / `hybrid` / `naive` / `mix`）实现增强的信息检索与问答。

**架构范式**：纯 Python 框架库（编排/适配层）+ TypeScript/React 19 前端。核心采用"能力缝 + 注册表/工厂 + 插件体系"设计：4 类存储（KV / Vector / Graph / DocStatus）各有可插拔后端矩阵，解析器支持 4 引擎路由，分块策略通过注册表扩展，LLM 提供商以目录即实例库方式组织。

**代码规模**：

| 模块 | 文件数 | 行数 |
|------|--------|------|
| Python 核心 `lightrag/` | 177 个 .py | 约 166,052 行 |
| 前端 `lightrag_webui/src/` | 231 个 .ts/.tsx | 约 37,109 行 |
| 测试 `tests/` | — | 约 7,800 个离线测试 |

**关键大文件**：`lightrag.py`（8,431 行，主类）、`pipeline.py`（8,228 行，摄入管线）、`utils.py`（8,293 行）、`operate.py`（7,126 行，检索与提取）、`postgres_impl.py`（10,032 行，PG 存储家族）。

## 2. 功能总览

| 领域 | 功能模块 | 归属域 / 叶子 |
|------|----------|---------------|
| 核心编排 | LightRAG 主类（mixin 组合）、admin 写门、初始化/终结、ainsert/aquery 入口 | core / L01 lightrag-core |
| 检索提取 | 实体关系提取、分块编排、5 模式检索、重排序、查询校验 | core / L02 operate-retrieval |
| LLM 角色 | RoleSpec/ROLES 注册表、角色规范化、包装器重建、运行时配置更新 | core / L03 llm-roles |
| 文档摄入 | 摄入管线 mixin、解析分发、多模态分析、worker 脚手架 | pipeline / L04 pipeline-mixin |
| 摄入协调 | workspace 摄入邮箱、扫描任务存储 | pipeline / L05 pipeline-ingress |
| 管线工具 | doc-status 字段访问、文档身份、解析产物路径、载荷归一化 | pipeline / L06 utils-pipeline |
| 多模态 | 多模态实体增强、多模态提示词、分块 schema | pipeline / L07 multimodal-context |
| 存储契约 | 4 类存储抽象基类、后端注册表、工厂解析 | storage / L08 storage-contracts |
| 存储协调 | 命名空间/锁、storage_updated 围栏、pipeline_status、写序号 | storage / L09 storage-coordination |
| 文件型存储 | 五存储家族、两通道围栏、requires_single_writer | storage / L10 file-backed-stores |
| PG 存储 | PG 家族 KV/向量/图/文档状态 | storage / L11 postgres-storage |
| 文档库存储 | MongoDB / OpenSearch 家族 | storage / L12 document-db-storage |
| 向量矩阵 | Milvus / Qdrant / Redis 向量缓存 | storage / L13 vector-db-matrix |
| 图数据库 | Neo4j / Memgraph 图存储 | storage / L14 graph-db-storage |
| 图操作 | merge/edit/delete/purge 锚点、kg_write_state 元数据 | storage / L15 graph-operations |
| 解析引擎 | 4 引擎路由（legacy/native/mineru/docling）、注册分发、CLI | parser / L16 parser-engine |
| DOCX 解析 | smart_heading 四件套、标题流、表格/图片提取、IR 构建 | parser / L17 docx-parser |
| 外部解析 | MinerU / Docling HTTP 适配器 | parser / L18 external-parsers |
| Markdown 解析 | Markdown / HTML 表格解析 | parser / L19 markdown-parser |
| Sidecar | Sidecar 文件格式 writer、IRDoc → *.parsed/ | parser / L20 sidecar |
| 分块注册 | 分块策略注册表/工厂 | chunker / L21 chunker-registry |
| 基础分块 | token-size / recursive-character 分块 | chunker / L22 token-recursive-chunkers |
| 语义分块 | semantic-vector / paragraph-semantic 分块 | chunker / L23 semantic-chunkers |
| LLM 通用 | 错误处理、视觉工具、废弃接口 | llm / L24 llm-common |
| OpenAI 家族 | OpenAI / Azure / NVIDIA + binding_options | llm / L25 llm-openai-family |
| LLM 提供商矩阵 | Ollama / Gemini / Anthropic / Bedrock 等 11 家 | llm / L26 llm-provider-matrix |
| API 核心 | FastAPI 应用组装、Gunicorn、生命周期、运行时校验 | api / L27 api-server-core |
| API 路由 | 文档/图谱/查询/Ollama 兼容/UI 定制 5 路由 | api / L28 api-routers |
| API 认证 | 鉴权、准入控制、请求体限制、登录限流、密码哈希 | api / L29 api-auth |
| API 配置 | env 配置面、UI 定制配置 | api / L30 api-config |
| 运维工具 | 9 个维护工具 + RAG 质量评估 + 离线检索检查 | tools / L31 ops-tools |
| 前端 API 客户端 | REST/流式客户端封装 | webui / L32 webui-api-client |
| 前端检索 | QueryComposer、useQuerySession、流式问答 | webui / L33 webui-retrieval |
| 前端文档管理 | DocumentManager、文档状态/删除/恢复 | webui / L34 webui-document-manager |
| 前端工作区 | WorkspaceApp、authBootstrap、工作区查询 | webui / L35 webui-workspace |
| 前端图谱查看 | sigma/graphology 图谱可视化、17 控制组件 | webui / L36 webui-graph-viewer |
| 前端登录 | LoginPage、令牌校验、协议同意 | webui / L37 webui-login-auth |
| 前端状态库 | 14 个 store + lib + hooks + contexts + migrations | webui / L38 webui-state-lib |

## 3. 解决的问题

| 用户痛点 | LightRAG 解法 |
|----------|---------------|
| 传统 RAG 仅靠向量相似度检索，缺乏语义关联与推理上下文 | 构建实体-关系知识图谱，支持 local（实体上下文）、global（社区摘要）、hybrid（图谱+向量）、mix（多源融合+重排）等多模式检索 |
| 文档格式多样（PDF/DOCX/Markdown/图片/音视频），解析质量参差 | 统一解析引擎路由 4 引擎（legacy/native/mineru/docling），native DOCX 解析含 smart_heading 智能标题流，外部 MinerU/Docling 处理复杂版式，多模态实体增强覆盖图片/音视频 |
| 存储后端绑定，难以适配不同部署环境 | 4 类存储（KV/Vector/Graph/DocStatus）各有可插拔后端矩阵：文件型（NetworkX/JSON/Faiss/NanoVectorDB）、PostgreSQL、MongoDB、Neo4j、Milvus、Qdrant、Redis、OpenSearch、Memgraph 等，通过 STORAGE_IMPLEMENTATIONS 注册表与工厂解析 |
| 大规模文档摄入的并发与恢复 | Pipeline 并发契约：ingress mailbox 唯一唤醒通道，busy 不阻塞 enqueue，三态拒绝（destructive_busy/scanning_exclusive/manual_freeze_requested），FAILED 文档仅手动重试一次，doc_status 作为恢复锚点 |
| 跨存储写入无事务导致的数据不一致 | "Consistency without transactions" 设计哲学：可接受"自愈或方向无害"的不一致残留，残留必须文档化；失败必须 fail loud；purge 操作失败闭合（RecoveryAnchorMissingError → HTTP 409），除非四证之一（anchors/pre_graph/journal/empty_scope） |
| LLM 提供商多样，切换成本高 | LLM 提供商以"目录即实例库"组织（11+ 家），统一 EmbeddingFunc 与 complete 接口契约，llm_roles 角色系统支持按角色（entity/relation/query/summary 等）配置不同模型 |
| 缺少可视化管理界面 | React 19 + TypeScript WebUI：文档管理、图谱可视化（sigma/graphology）、检索问答、工作区隔离、登录鉴权 |

## 4. 系统边界

### 上边界（用户 / 第三方接入）
- **WebUI 前端**：React 19 + TypeScript + Vite + Bun + Tailwind，通过 REST API 与后端交互，支持流式问答（SSE）
- **REST API**：FastAPI 服务，提供文档管理、图谱操作、检索查询、UI 定制等端点
- **Ollama 兼容 API**：`/ollama/*` 端点，允许 Ollama 生态客户端直接接入
- **Python 核心 API**：`LightRAG` 类可直接在 Python 代码中实例化使用（`ainsert` / `aquery` / `ainsert_custom_kg`）

### 下边界（基础设施）
- **LLM 服务**：OpenAI / Azure OpenAI / Ollama / Gemini / Anthropic / Bedrock / NVIDIA / VoyageAI / Jina / Zhipu 等（**不在本仓库源码内**，通过 HTTP API 调用）
- **嵌入模型服务**：同上提供商的 embedding 端点（**不在本仓库源码内**）
- **存储后端**：PostgreSQL / MongoDB / Neo4j / Milvus / Qdrant / Redis / OpenSearch / Memgraph 等（**不在本仓库源码内**，本仓库仅提供适配层）
- **外部解析服务**：MinerU / Docling（**不在本仓库源码内**，通过 HTTP 适配器调用）

### 内边界（本仓库 vs 扩展 / 外部）
- **本仓库实现**：核心编排、检索提取、摄入管线、4 类存储抽象与适配层、解析引擎与 native 解析器、分块策略、LLM 接口封装、API Server、WebUI、运维工具
- **可扩展点**：存储后端（实现 BaseKVStorage/BaseVectorStorage/BaseGraphStorage/BaseDocStatusStorage 并注册）、解析引擎（注册到 parser registry）、分块策略（注册到 chunker registry）、LLM 提供商（遵循 complete/embed 接口契约）
- **不做什么**：不提供 LLM 模型训练/推理运行时（依赖外部 API）；不提供存储服务的部署/运维（仅适配层）；不提供分布式调度（单进程多 worker，Gunicorn 多进程）；不实现文档 OCR 引擎（依赖 native 解析或外部 MinerU/Docling）

### 侧边界
- **多 workspace 隔离**：每个 LightRAG 实例可传 `workspace` 参数实现数据隔离——文件型用子目录、collection 型用前缀、关系型 DB 用列过滤、Qdrant 用 payload 分区
- **多角色 LLM 配置**：llm_roles 系统支持按提取/查询/摘要等角色配置不同模型与参数
- **Sidecar 格式**：解析产物以 `*.parsed/` 目录持久化，支持离线复用与调试

## 5. 系统架构图说明

![系统架构图](system-architecture.html)

系统架构分为四层：

**接入层**：WebUI 前端（React 19 + TS）与第三方客户端（Ollama 兼容 / REST）统一接入 API Server（FastAPI + Gunicorn），请求先经认证鉴权层（auth / admission / 限流 / 请求体限制）预检。

**核心编排层**：LightRAG 主类由 mixin 组合（`_RoleLLMMixin → _StorageMigrationMixin → _PipelineMixin → object`，@final），向下调度三个子系统：
- **检索与提取**（operate + rerank）：实体关系提取、5 模式检索、重排序，调用外部 LLM 与嵌入模型
- **摄入管线**（pipeline mixin）：文档摄入、解析分发、多模态分析、worker 脚手架
- **LLM 角色管理**（llm_roles）：角色规范化、包装器重建、运行时配置更新

**处理层**：解析引擎（4 引擎路由：legacy / native / mineru / docling）与分块策略（注册表 4 策略：token-size / recursive-character / semantic-vector / paragraph-semantic），由摄入管线调度。

**4 类存储层**：KV 存储（LLM 缓存 / chunk / 文档信息）、向量存储（实体 / 关系 / chunk 嵌入）、图存储（实体关系图）、文档状态存储（pipeline 恢复锚点）。每类存储有可插拔后端矩阵，通过外部存储后端（PostgreSQL / Neo4j / Redis 等）适配。

**关键设计决策**：
- 跨存储无事务（Consistency without transactions），可接受自愈残留
- 文件型五存储家族采用两通道围栏（文件 `(st_mtime_ns, st_size)` OR `storage_updated` 标志）
- NetworkXStorage 唯一声明 `requires_single_writer`，admin 写操作在 `_admin_write_gate` 下串行
- 检索引擎读写 KV / 向量 / 图存储，摄入管线追踪文档状态

## 6. 核心时序图说明

![查询请求时序图](system-query-sequence.html)

以查询请求（`aquery(mode=mix)`）为例的核心调用链：

1. **用户输入问题** → WebUI 前端组装查询请求
2. **POST /query** → API Server 接收，调用鉴权层进行 token 校验
3. **鉴权通过** → API Server 调用 `LightRAG.aquery()`，传入查询参数（mode、top_k、token 预算等）
4. **多模式检索** → LightRAG 调度检索引擎，根据 mode 执行对应检索策略（mix 模式融合图谱检索与向量检索）
5. **查 4 类存储** → 检索引擎查询图存储（实体/关系/社区）、向量存储（相似 chunk）、KV 存储（缓存/chunk 文本），返回候选上下文
6. **重排序** → 检索引擎对候选上下文进行重排序（如启用 rerank），返回精炼上下文
7. **LLM 生成** → LightRAG 将 prompt + 上下文发送给外部 LLM 服务，获取流式生成结果
8. **SSE 流返回** → 生成结果以 SSE 流式响应经 API Server 返回 WebUI，前端逐块渲染答案

**关键特性**：流式生成（SSE）、多模式检索可配置、重排序可选、鉴权预检、查询参数校验（query_validation）。

## 7. 系统数据流图说明

![文档摄入数据流图](system-ingestion-dataflow.html)

文档摄入的端到端数据流：

**输入阶段**：原始文档提交至摄入邮箱（ingress mailbox），这是 pipeline 的唯一唤醒通道。

**解析阶段**：摄入邮箱唤醒后，解析引擎根据文档类型与配置路由到对应解析器（native / MinerU / Docling / legacy），解析结果写入 Sidecar 产物目录（`*.parsed/`），包含 IR 文档结构。

**分块与提取阶段**：Sidecar 中的 IR 文本送入分块策略（token-size / recursive / semantic），生成 chunks；chunks 送入实体关系提取（含多模态增强），提取实体、关系与属性。

**向量化阶段**：提取的实体与关系通过嵌入模型计算向量表示。

**持久化阶段**：
- 嵌入向量 → 向量存储
- chunk 文本与 LLM 缓存 → KV 存储
- 实体关系图 → 图存储
- 文档处理状态 → 文档状态存储（作为 pipeline 恢复锚点）

**关键机制**：
- ingress mailbox 是唯一唤醒通道，doc_status 是状态真相来源
- FAILED 文档不会自动恢复，仅通过手动重试获得一次尝试机会
- 跨存储写入无事务，采用"可自愈残留"策略
- purge 操作失败闭合（RecoveryAnchorMissingError → 409），除非四证之一
