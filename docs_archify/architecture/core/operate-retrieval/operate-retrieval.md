# 提取与多模式检索（operate-retrieval）

> 本文是 `core` 域下的叶子子系统文档。域级总览见 `../core.md`。
> 本文展开实体/关系提取编排、5 模式检索（local/global/hybrid/naive/mix）、重排与查询校验；
> 主类入口 `aquery` 如何路由到本叶子见 `../lightrag-core/lightrag-core.md`，
> LLM 角色化调用见 `../llm-roles/llm-roles.md`。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 实体/关系提取 | 从分块文本批量提取实体与关系，支持 gleaning 多轮与 JSON 结构化输出 | `lightrag/operate.py:3942` `extract_entities` |
| 提取结果合并 | 单实体/单关系结果解析、归一化、写图与向量库 | `operate.py:2429` `_merge_nodes_then_upsert`、`operate.py:2782` `_merge_edges_then_upsert` |
| 描述合并摘要 | 同实体多描述去重合并，可选 LLM 摘要 | `operate.py:532` `_summarize_descriptions` |
| KG 查询主流程 | 按 `query_param.mode` 分发 local/global/hybrid/mix 检索 | `operate.py:4693` `kg_query` |
| 关键词抽取 | 从查询抽取 high/low level 关键词（命中即短路） | `operate.py:4975` `get_keywords_from_query`、`operate.py:5112` `extract_keywords_only` |
| KG 搜索 | 按关键词在实体/关系向量库召回并查图邻域 | `operate.py:5280` `_perform_kg_search` |
| 上下文拼装 | 实体/关系/chunk 上下文按 token 预算截断拼装 | `operate.py:5873` `_build_context_str`、`operate.py:6075` `_build_query_context` |
| Naive 检索 | 纯向量 chunk 检索，不走图 | `operate.py:6827` `naive_query` |
| 重排 | chunk 分桶、聚合打分、Cohere/Jina/Ali 重排 API | `lightrag/rerank.py:240` `generic_rerank_api` |
| 查询校验 | 空查询拒绝、宽字符权重最小长度校验 | `lightrag/query_validation.py:130` `validate_rag_query` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `QueryResult` | `operate.py:114` 附近 | 统一查询返回（content/stream/raw_data） |
| `QueryProgress` | `operate.py:114` | 进度回调阶段枚举（提取关键词/搜索/…） |
| `KGRebuildReport` | `operate.py:137` | 从 chunk 重建图谱的报告 |
| `kg_query` | `operate.py:4693` | KG 模式查询主入口，入参为各存储句柄 |
| `extract_entities` | `operate.py:3942` | 批量提取，入参 chunks dict |
| `generic_rerank_api` | `rerank.py:240` | 通用重排 HTTP 调用 |
| `validate_rag_query` | `query_validation.py:130` | 查询准入校验，抛 `QueryValidationError` 子类 |

## 3. 关键调用链

**调用链 A：KG 查询（`operate.py:4693` `kg_query`）**
1. 取角色化 LLM 函数 `global_config["role_llm_funcs"]["query"]`（priority 5）。
2. `get_keywords_from_query`（`operate.py:4975`）抽 high/low 关键词；空关键词按 mode 告警。
3. `_perform_kg_search`（`operate.py:5280`）按 mode 分支：`local`(5366) 走实体邻域、`global`(5377) 走关系邻域；`mix`(5411) 额外查 chunks_vdb。
4. `_build_query_context`（`operate.py:6075`）→ `_build_context_str`（`operate.py:5873`）按 `max_*_tokens` 截断拼装。
5. 调 query LLM 生成（流式/非流式），包成 `QueryResult` 返回。

**调用链 B：实体提取（`operate.py:3942` `extract_entities`）**
1. 入口先查 `pipeline_status["cancellation_requested"]`，命中抛 `PipelineCancelledException`（`operate.py:3958`）。
2. 取 `role_llm_funcs["extract"]`，按 `entity_extract_max_gleaning` 做多轮 gleaning。
3. 每 chunk 结果经 `_process_extraction_result`（`operate.py:1508`）解析，`_merge_nodes_then_upsert`（`operate.py:2429`）写图节点与实体向量库，`_merge_edges_then_upsert`（`operate.py:2782`）写边与关系向量库。
4. 截断计数 `TokenLimitTruncationTally` 聚合后回传管线。

**调用链 C：重排（`rerank.py:36`）**
1. `chunk_documents_for_rerank` 把候选 chunk 按重排上下文窗口分桶。
2. `generic_rerank_api`（`rerank.py:240`）调用远端重排服务（Cohere/Jina/Ali，`rerank.py:448/515/555`）。
3. `aggregate_chunk_scores`（`rerank.py:165`）跨桶聚合 chunk 分数。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `query_param.mode` | `local/global/hybrid/mix/naive`；mix 推荐配 reranker | `operate.py:5326-5411` |
| `top_k` / `chunk_top_k` | 实体/关系召回数与 chunk 召回数 | 见 lightrag-core |
| `max_entity_tokens` / `max_relation_tokens` / `max_total_tokens` | 上下文 token 预算 | `_apply_token_truncation`（`operate.py:5501`） |
| `entity_extract_max_gleaning` | 提取轮数 | `operate.py:3975` |
| `entity_extraction_use_json` | JSON 结构化输出开关 | `operate.py:4004` |
| `MAX_EXTRACT_INPUT_TOKENS` | gleaning 输入上限，0 禁用 | `operate.py:3984` |
| `enable_rerank` | mix 模式下是否重排 | rerank.py |

## 5. 错误与重试语义

- **空关键词**：local 模式无 low-level、global 模式无 high-level 仅告警，不失败；两者皆空且查询很短时走兜底。
- **取消传播**：提取入口每批开头检查 `cancellation_requested`，命中即抛 `PipelineCancelledException` 中断。
- **重排失败**：重排为增强路径，远端重排服务不可用时回退到原始向量打分排序（不阻断查询）。
- **查询校验**：空查询抛 `EmptyQueryError`；过短查询抛 `RAGQueryTooShortError`（按宽字符权重计）。

## 6. 并发细节

- 提取以 chunk 为任务单元并发；`PipelineStatusLogger` 共享状态日志句柄，首写后缓存列表引用避免重复取。
- 关键词、向量召回按 mode 预批量取 embedding（`operate.py:5310` 单次批量），避免逐 chunk 往返。
- 提取写图与写向量库跨存储无事务，遵循 Consistency without transactions。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/operate.py` 提取与检索编排
- `lightrag/rerank.py` 重排客户端
- `lightrag/query_validation.py` 查询准入
- `lightrag/table_markup.py` 表格标记

**Out-of-Scope（不在本仓库源码内）**
- 外部 LLM/embedding/重排服务实际推理（OpenAI/Cohere/Jina/Ali）——见 llm 域
- 存储后端读写实现——见 storage 域
- chunker 分块策略——见 chunker 域

## 8. 与相邻子系统交互

- 上游：`LightRAG.aquery_llm` → `kg_query` / `naive_query`；`_process_extract_entities` → `extract_entities`
- 下游：operate → 图存储（`chunk_entity_relation_graph`）、实体/关系/chunk 向量库、text_chunks KV
- 横切：operate → `role_llm_funcs["extract"/"query"]`（L03），→ 重排远端服务

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝**：operate 是检索/提取编排层，不持有数据；通过入参注入各存储句柄，便于测试替身。
- **配置驱动分派**：同一 `kg_query` 代码经 `query_param.mode` 字符串路由到 local/global/hybrid/mix 四条分支，是框架级静态多态。
- **角色化函数注册表**：LLM 调用不直接绑定 provider，而是从 `global_config["role_llm_funcs"]["extract"/"query"]` 取可调用对象，由 L03 装配。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 提取与检索模块架构图 | `operate-retrieval-architecture.html` | architecture | standard（跨存储列长横线穿节点、标签压组件；已删重排服务外链、清空冲突边标签） |
| KG 查询调用时序 | `operate-retrieval-sequence.html` | sequence | standard |
| 实体提取数据流 | `operate-retrieval-dataflow.html` | dataflow | standard（并行"嵌入"标签重叠；已加 labelDy 错峰） |

本叶子补 sequence（kg_query 多方调用链）与 dataflow（chunk→提取→合并→存储管道）；不补 lifecycle（无单实体多状态迁移，文档状态机在 pipeline 域）。JSON IR 位于 `json/`。三图均 standard 渲染通过、HTML >800KB。
