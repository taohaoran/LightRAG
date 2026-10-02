# LightRAG 系统架构文档

> 基于 LightRAG 源码（`lightrag-hku`，Python >= 3.10，branch `main`，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`，约 177 个 Python 文件 / 16.6 万行 + 231 个 TS/TSX 文件 / 3.7 万行）深度分析产出，覆盖系统级、9 个域、38 个叶子子系统的功能、问题域、系统边界、架构图、时序图与数据流图。所有图表由 archify 渲染为自包含交互式 HTML。

## 文档导航

### 系统级

| 文档 | 说明 | 图表 |
|------|------|------|
| [system-overview.md](system-overview.md) | 功能总览、解决的问题、系统边界、架构/时序/数据流图说明 | [系统架构图](system-architecture.html) · [查询时序图](system-query-sequence.html) · [摄入数据流图](system-ingestion-dataflow.html) |

---

### core 核心编排域（`core/`）

| 子系统 | 文档 | 架构图 | 其他图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [core.md](core/core.md) | [架构图](core/core-architecture.html) | [时序](core/core-sequence.html) · [数据流](core/core-dataflow.html) | LightRAG 主类组合、检索提取、LLM 角色 |
| L01 lightrag-core | [lightrag-core.md](core/lightrag-core/lightrag-core.md) | [架构图](core/lightrag-core/lightrag-core-architecture.html) | [时序](core/lightrag-core/lightrag-core-sequence.html) · [生命周期](core/lightrag-core/lightrag-core-lifecycle.html) | LightRAG 主类 mixin 组合、admin 写门、初始化、ainsert/aquery 入口 |
| L02 operate-retrieval | [operate-retrieval.md](core/operate-retrieval/operate-retrieval.md) | [架构图](core/operate-retrieval/operate-retrieval-architecture.html) | [时序](core/operate-retrieval/operate-retrieval-sequence.html) · [数据流](core/operate-retrieval/operate-retrieval-dataflow.html) | 实体关系提取、5 模式检索、重排序、查询校验 |
| L03 llm-roles | [llm-roles.md](core/llm-roles/llm-roles.md) | [架构图](core/llm-roles/llm-roles-architecture.html) | [时序](core/llm-roles/llm-roles-sequence.html) | RoleSpec/ROLES 注册表、角色规范化、包装器重建 |

### pipeline 文档摄入域（`pipeline/`）

| 子系统 | 文档 | 架构图 | 其他图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [pipeline.md](pipeline/pipeline.md) | [架构图](pipeline/pipeline-architecture.html) | [数据流](pipeline/pipeline-dataflow.html) · [生命周期](pipeline/pipeline-lifecycle.html) | 文档摄入管线、并发控制、多模态 |
| L04 pipeline-mixin | [pipeline-mixin.md](pipeline/pipeline-mixin/pipeline-mixin.md) | [架构图](pipeline/pipeline-mixin/pipeline-mixin-architecture.html) | [数据流](pipeline/pipeline-mixin/pipeline-mixin-dataflow.html) · [生命周期](pipeline/pipeline-mixin/pipeline-mixin-lifecycle.html) | 摄入管线 mixin、解析分发、多模态分析、worker 脚手架 |
| L05 pipeline-ingress | [pipeline-ingress.md](pipeline/pipeline-ingress/pipeline-ingress.md) | [架构图](pipeline/pipeline-ingress/pipeline-ingress-architecture.html) | [时序](pipeline/pipeline-ingress/pipeline-ingress-sequence.html) | workspace 摄入邮箱、扫描任务存储 |
| L06 utils-pipeline | [utils-pipeline.md](pipeline/utils-pipeline/utils-pipeline.md) | [架构图](pipeline/utils-pipeline/utils-pipeline-architecture.html) | [数据流](pipeline/utils-pipeline/utils-pipeline-dataflow.html) | doc-status 字段访问、文档身份、解析产物路径 |
| L07 multimodal-context | [multimodal-context.md](pipeline/multimodal-context/multimodal-context.md) | [架构图](pipeline/multimodal-context/multimodal-context-architecture.html) | [数据流](pipeline/multimodal-context/multimodal-context-dataflow.html) | 多模态实体增强、多模态提示词、分块 schema |

### storage 图与存储域（`storage/`）

| 子系统 | 文档 | 架构图 | 其他图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [storage.md](storage/storage.md) | [架构图](storage/storage-architecture.html) | [数据流](storage/storage-write-dataflow.html) · [时序](storage/storage-write-sequence.html) | 4 类存储抽象、后端矩阵、一致性模型 |
| L08 storage-contracts | [storage-contracts.md](storage/storage-contracts/storage-contracts.md) | [架构图](storage/storage-contracts/storage-contracts-architecture.html) | [时序](storage/storage-contracts/storage-contracts-resolution-sequence.html) | 4 类存储抽象基类、后端注册表、工厂解析 |
| L09 storage-coordination | [storage-coordination.md](storage/storage-coordination/storage-coordination.md) | [架构图](storage/storage-coordination/storage-coordination-architecture.html) | [时序](storage/storage-coordination/storage-coordination-fence-sequence.html) | 命名空间/锁、storage_updated 围栏、pipeline_status |
| L10 file-backed-stores | [file-backed-stores.md](storage/file-backed-stores/file-backed-stores.md) | [架构图](storage/file-backed-stores/file-backed-stores-architecture.html) | [数据流](storage/file-backed-stores/file-backed-stores-replay-dataflow.html) · [生命周期](storage/file-backed-stores/file-backed-stores-commit-lifecycle.html) | 文件型五存储家族、两通道围栏、requires_single_writer |
| L11 postgres-storage | [postgres-storage.md](storage/postgres-storage/postgres-storage.md) | [架构图](storage/postgres-storage/postgres-storage-architecture.html) | [时序](storage/postgres-storage/postgres-storage-flush-sequence.html) | PG 家族 KV/向量/图/文档状态 |
| L12 document-db-storage | [document-db-storage.md](storage/document-db-storage/document-db-storage.md) | [架构图](storage/document-db-storage/document-db-storage-architecture.html) | [数据流](storage/document-db-storage/document-db-flush-dataflow.html) | MongoDB / OpenSearch 文档搜索库家族 |
| L13 vector-db-matrix | [vector-db-matrix.md](storage/vector-db-matrix/vector-db-matrix.md) | [架构图](storage/vector-db-matrix/vector-db-matrix-architecture.html) | [数据流](storage/vector-db-matrix/vector-db-deferred-dataflow.html) | Milvus / Qdrant / Redis 向量缓存矩阵 |
| L14 graph-db-storage | [graph-db-storage.md](storage/graph-db-storage/graph-db-storage.md) | [架构图](storage/graph-db-storage/graph-db-storage-architecture.html) | [时序](storage/graph-db-storage/graph-db-write-sequence.html) | Neo4j / Memgraph 图数据库 |
| L15 graph-operations | [graph-operations.md](storage/graph-operations/graph-operations.md) | [架构图](storage/graph-operations/graph-operations-architecture.html) | [时序](storage/graph-operations/graph-operations-admin-sequence.html) | merge/edit/delete/purge 锚点、kg_write_state 元数据 |

### parser 解析域（`parser/`）

| 子系统 | 文档 | 架构图 | 其他图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [parser.md](parser/parser.md) | [架构图](parser/parser-domain-architecture.html) | [数据流](parser/parser-domain-dataflow.html) · [时序](parser/parser-domain-sequence.html) | 4 引擎解析路由、DOCX 深度解析、Sidecar |
| L16 parser-engine | [parser-engine.md](parser/parser-engine/parser-engine.md) | [架构图](parser/parser-engine/parser-engine-architecture.html) | [时序](parser/parser-engine/parser-engine-sequence.html) | 解析引擎路由/注册/分发/CLI |
| L17 docx-parser | [docx-parser.md](parser/docx-parser/docx-parser.md) | [架构图](parser/docx-parser/docx-parser-architecture.html) | [时序](parser/docx-parser/docx-parser-sequence.html) | smart_heading 四件套、标题流、表格/图片提取 |
| L18 external-parsers | [external-parsers.md](parser/external-parsers/external-parsers.md) | [架构图](parser/external-parsers/external-parsers-architecture.html) | [时序](parser/external-parsers/external-parsers-sequence.html) | MinerU / Docling HTTP 适配器 |
| L19 markdown-parser | [markdown-parser.md](parser/markdown-parser/markdown-parser.md) | [架构图](parser/markdown-parser/markdown-parser-architecture.html) | [时序](parser/markdown-parser/markdown-parser-sequence.html) | Markdown / HTML 表格解析 |
| L20 sidecar | [sidecar.md](parser/sidecar/sidecar.md) | [架构图](parser/sidecar/sidecar-architecture.html) | [数据流](parser/sidecar/sidecar-dataflow.html) | Sidecar 文件格式 writer、IRDoc → *.parsed/ |

### chunker 分块域（`chunker/`）

| 子系统 | 文档 | 架构图 | 其他图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [chunker.md](chunker/chunker.md) | [架构图](chunker/chunker-domain-architecture.html) | [数据流](chunker/chunker-domain-dataflow.html) · [时序](chunker/chunker-domain-sequence.html) | 分块策略注册表、4 种分块策略 |
| L21 chunker-registry | [chunker-registry.md](chunker/chunker-registry/chunker-registry.md) | [架构图](chunker/chunker-registry/chunker-registry-architecture.html) | [时序](chunker/chunker-registry/chunker-registry-sequence.html) | 分块策略注册表/工厂 |
| L22 token-recursive-chunkers | [token-recursive-chunkers.md](chunker/token-recursive-chunkers/token-recursive-chunkers.md) | [架构图](chunker/token-recursive-chunkers/token-recursive-chunkers-architecture.html) | [数据流](chunker/token-recursive-chunkers/token-recursive-chunkers-dataflow.html) | token-size / recursive-character 分块 |
| L23 semantic-chunkers | [semantic-chunkers.md](chunker/semantic-chunkers/semantic-chunkers.md) | [架构图](chunker/semantic-chunkers/semantic-chunkers-architecture.html) | [数据流](chunker/semantic-chunkers/semantic-chunkers-dataflow.html) | semantic-vector / paragraph-semantic 分块 |

### llm 模型接入域（`llm/`）

| 子系统 | 文档 | 架构图 | 其他图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [llm.md](llm/llm.md) | [架构图](llm/llm-architecture.html) | [数据流](llm/llm-dataflow.html) · [时序](llm/llm-sequence.html) | LLM 接口封装、OpenAI 家族、11+ 提供商矩阵 |
| L24 llm-common | [llm-common.md](llm/llm-common/llm-common.md) | [架构图](llm/llm-common/llm-common-architecture.html) | [数据流](llm/llm-common/llm-common-dataflow.html) | 错误处理、视觉工具、废弃接口 |
| L25 llm-openai-family | [llm-openai-family.md](llm/llm-openai-family/llm-openai-family.md) | [架构图](llm/llm-openai-family/llm-openai-family-architecture.html) | [时序](llm/llm-openai-family/llm-openai-family-sequence.html) | OpenAI / Azure / NVIDIA + binding_options |
| L26 llm-provider-matrix | [llm-provider-matrix.md](llm/llm-provider-matrix/llm-provider-matrix.md) | [架构图](llm/llm-provider-matrix/llm-provider-matrix-architecture.html) | [数据流](llm/llm-provider-matrix/llm-provider-matrix-dataflow.html) | Ollama / Gemini / Anthropic / Bedrock 等 11 家 |

### api 服务域（`api/`）

| 子系统 | 文档 | 架构图 | 其他图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [api.md](api/api.md) | [架构图](api/api-architecture.html) | [时序](api/api-sequence.html) · [数据流](api/api-dataflow.html) | FastAPI 服务、5 路由、认证鉴权、配置 |
| L27 api-server-core | [api-server-core.md](api/api-server-core/api-server-core.md) | [架构图](api/api-server-core/api-server-core-architecture.html) | [时序](api/api-server-core/api-server-core-sequence.html) | FastAPI 应用组装、Gunicorn、生命周期 |
| L28 api-routers | [api-routers.md](api/api-routers/api-routers.md) | [架构图](api/api-routers/api-routers-architecture.html) | [时序](api/api-routers/api-routers-sequence.html) | 文档/图谱/查询/Ollama/UI 定制 5 路由 |
| L29 api-auth | [api-auth.md](api/api-auth/api-auth.md) | [架构图](api/api-auth/api-auth-architecture.html) | [时序](api/api-auth/api-auth-sequence.html) | 鉴权、准入控制、限流、密码哈希 |
| L30 api-config | [api-config.md](api/api-config/api-config.md) | [架构图](api/api-config/api-config-architecture.html) | [数据流](api/api-config/api-config-dataflow.html) | env 配置面、UI 定制配置 |

### tools 运维与评估域（`tools/`）

| 子系统 | 文档 | 架构图 | 其他图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [tools.md](tools/tools.md) | [架构图](tools/tools-domain-architecture.html) | — | 9 个运维工具 + RAG 质量评估 |
| L31 ops-tools | [ops-tools.md](tools/ops-tools/ops-tools.md) | [架构图](tools/ops-tools/ops-tools-architecture.html) | [数据流](tools/ops-tools/ops-tools-dataflow.html) | 9 个维护工具 + 评估/离线检索检查 |

### webui 前端域（`webui/`）

| 子系统 | 文档 | 架构图 | 其他图 | 职责一句话 |
|--------|------|--------|--------|-----------|
| 域总览 | [webui.md](webui/webui.md) | [架构图](webui/webui-architecture.html) | [数据流](webui/webui-dataflow.html) · [时序](webui/webui-sequence.html) | React 19 + TS 前端、7 功能模块 |
| L32 webui-api-client | [webui-api-client.md](webui/webui-api-client/webui-api-client.md) | [架构图](webui/webui-api-client/webui-api-client-architecture.html) | [时序](webui/webui-api-client/webui-api-client-sequence.html) | REST/流式客户端封装 |
| L33 webui-retrieval | [webui-retrieval.md](webui/webui-retrieval/webui-retrieval.md) | [架构图](webui/webui-retrieval/webui-retrieval-architecture.html) | [时序](webui/webui-retrieval/webui-retrieval-sequence.html) | QueryComposer、useQuerySession、流式问答 |
| L34 webui-document-manager | [webui-document-manager.md](webui/webui-document-manager/webui-document-manager.md) | [架构图](webui/webui-document-manager/webui-document-manager-architecture.html) | [数据流](webui/webui-document-manager/webui-document-manager-dataflow.html) | DocumentManager、文档状态/删除/恢复 |
| L35 webui-workspace | [webui-workspace.md](webui/webui-workspace/webui-workspace.md) | [架构图](webui/webui-workspace/webui-workspace-architecture.html) | [时序](webui/webui-workspace/webui-workspace-sequence.html) | WorkspaceApp、authBootstrap、工作区查询 |
| L36 webui-graph-viewer | [webui-graph-viewer.md](webui/webui-graph-viewer/webui-graph-viewer.md) | [架构图](webui/webui-graph-viewer/webui-graph-viewer-architecture.html) | [时序](webui/webui-graph-viewer/webui-graph-viewer-sequence.html) | sigma/graphology 图谱可视化、17 控制组件 |
| L37 webui-login-auth | [webui-login-auth.md](webui/webui-login-auth/webui-login-auth.md) | [架构图](webui/webui-login-auth/webui-login-auth-architecture.html) | [时序](webui/webui-login-auth/webui-login-auth-sequence.html) | LoginPage、令牌校验、协议同意 |
| L38 webui-state-lib | [webui-state-lib.md](webui/webui-state-lib/webui-state-lib.md) | [架构图](webui/webui-state-lib/webui-state-lib-architecture.html) | [数据流](webui/webui-state-lib/webui-state-lib-dataflow.html) | 14 个 store + lib + hooks + contexts + migrations |

---

## 产出统计

| 层级 | MD | HTML | JSON | 说明 |
|------|-----|------|------|------|
| 系统级 | 2（README + system-overview） | 3 | 3 | architecture + sequence + dataflow |
| core 域 | 4（域总览 + 3 叶子） | 12 | 12 | 每域 ≥3 图，每叶子 ≥2 图 |
| pipeline 域 | 5（域总览 + 4 叶子） | 14 | 14 | |
| storage 域 | 9（域总览 + 8 叶子） | 20 | 20 | 最重域 |
| parser 域 | 6（域总览 + 5 叶子） | 13 | 13 | |
| chunker 域 | 4（域总览 + 3 叶子） | 9 | 9 | |
| llm 域 | 4（域总览 + 3 叶子） | 9 | 9 | |
| api 域 | 5（域总览 + 4 叶子） | 10 | 10 | |
| tools 域 | 2（域总览 + 1 叶子） | 3 | 5 | 单叶子域 |
| webui 域 | 8（域总览 + 7 叶子） | 17 | 17 | TS 前端 |
| **合计** | **49** | **108** | **110** | **9 域 / 38 叶子** |

## 覆盖范围与说明

### 叶子清单（38 个，L01–L38）
core（3）：L01 lightrag-core、L02 operate-retrieval、L03 llm-roles
pipeline（4）：L04 pipeline-mixin、L05 pipeline-ingress、L06 utils-pipeline、L07 multimodal-context
storage（8）：L08 storage-contracts、L09 storage-coordination、L10 file-backed-stores、L11 postgres-storage、L12 document-db-storage、L13 vector-db-matrix、L14 graph-db-storage、L15 graph-operations
parser（5）：L16 parser-engine、L17 docx-parser、L18 external-parsers、L19 markdown-parser、L20 sidecar
chunker（3）：L21 chunker-registry、L22 token-recursive-chunkers、L23 semantic-chunkers
llm（3）：L24 llm-common、L25 llm-openai-family、L26 llm-provider-matrix
api（4）：L27 api-server-core、L28 api-routers、L29 api-auth、L30 api-config
tools（1）：L31 ops-tools
webui（7）：L32 webui-api-client、L33 webui-retrieval、L34 webui-document-manager、L35 webui-workspace、L36 webui-graph-viewer、L37 webui-login-auth、L38 webui-state-lib

### 质量档位说明
- 系统级图：architecture（standard，组件跨层连接多为预期内）、sequence（showcase）、dataflow（standard）
- 叶子级图：目标 showcase，部分落 standard（均已在对应叶子 MD 第 10 节披露失败检查名与修复动作）
- 所有图 render 退出码 0、HTML > 700KB（最小 796KB）为硬门槛，已全部满足

### 语言适配口径
- **Python 核心**（core/pipeline/storage/parser/chunker/llm/api/tools 域）：按 language-python.md 口径——能力缝归组、注册表与工厂、代码生成与 DRY、可选依赖与懒加载、多框架矩阵、双轨实现、配置驱动、目录即实例库
- **TypeScript 前端**（webui 域）：按 language-ts.md 口径——capability seam 分组、异步调用链、事件流、依赖图
- 两份语言专项的适配口径分别在对应叶子 MD 第 9 节落写

### 缺口如实披露
- "目录即实例库"叶子（L13 vector-db-matrix、L14 graph-db-storage、L26 llm-provider-matrix、L12 document-db-storage）：深读 1–2 个代表后端，其余以共性表格归纳，未逐函数精读的清单已在各叶子 MD 第 9 节披露
- L38 webui-state-lib：深读 settings/graph/querySettings 三个代表 store，其余 11 个 store 与 lib 工具以共性表格归类
- L17 docx-parser：深读 parse_document.py 与 heading_flow.py 作为代表，smart_heading 其余模块以共性说明
- 外部系统（LLM 提供商、存储后端、MinerU/Docling 服务）均标注"不在本仓库源码内"，仅分析适配层

### 源码事实校正
- 所有叶子 MD 的关键调用链均含真实 `文件:函数:行号`，来自源码定点精读
- 权威契约口径（AGENTS.md + docs/design/*.md）已在 storage/pipeline 相关叶子中严格遵守：两通道围栏、JsonKV/JsonDocStatus 的 storage_updated 语义相反、purge 失败闭合四证、kg_write_state 单调禁止回填、pipeline 三态拒绝、关系权重契约
