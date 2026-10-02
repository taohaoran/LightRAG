# 文档摄入管线（pipeline-mixin）

> 本文是 `pipeline` 域下的叶子子系统文档。域级总览见 `../pipeline.md`。
> 本文展开 `_PipelineMixin` 的入队、调度、feeder 与三类 worker 脚手架；
> 摄入邮箱与扫描任务存储见 `../pipeline-ingress/pipeline-ingress.md`，
> 文档身份/字段工具见 `../utils-pipeline/utils-pipeline.md`，多模态分析见 `../multimodal-context/multimodal-context.md`。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 文档入队 | 校验/生成 MD5 id、内容去重、写初始 doc_status、过滤已处理、入队 | `lightrag/pipeline.py:740` `apipeline_enqueue_documents` |
| 错误文档入队 | 把失败文档重新标记待处理（手动重试） | `pipeline.py:1650` `apipeline_enqueue_error_documents` |
| 管线处理主循环 | 调度分页、feeder 喂入、worker 处理、决定下一步 | `pipeline.py:1724` `apipeline_process_enqueue_documents` |
| 准入控制 | 按 `MAX_PENDING_DOCUMENTS` 限制待入队容量 | `pipeline.py:638` `_reserve_ingress_slot` |
| feeder 喂入 | 从 ingress 邮箱取批、准入、路由、退避 | `pipeline.py:2507` `_pipeline_feeder` |
| 解析 worker | 分发 parse_native / parse_mineru / parse_docling | `pipeline.py:4409` `_parse_worker` |
| 多模态分析 worker | VLM 分析图片/音视频实体增强 | `pipeline.py:4805` `_analyze_worker` |
| 处理 worker | 单文档状态机：分块→KG 提取→合并 | `pipeline.py:4950` `_process_worker`、`pipeline.py:4991` `process_single_document` |
| 手动排空/冻结 | 扫描/手动重试期间排空管线到空闲 | `pipeline.py:3158` `_begin_manual_drain` |
| 取消传播 | 阶段间检查 `cancellation_requested` | `pipeline.py:6156` `_raise_if_cancelled` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `_PipelineMixin` | `pipeline.py:624` | 管线混入类，装配到 LightRAG |
| `_BatchRunContext` | `pipeline.py:575` | 单批运行上下文（信号量、owner、epoch） |
| `PipelineNextStep` / `PipelineNextDecision` | `pipeline.py:327/365` | 管线调度下一步决策枚举 |
| `DocProcessingStatus` | doc_status 存储 | 文档处理状态记录 |
| `process_single_document` | `pipeline.py:4991` | 单文档分块→提取→合并状态机 |

## 3. 关键调用链

**调用链 A：入队（`pipeline.py:740` `apipeline_enqueue_documents`）**
1. 未提供 id 时按内容/路径生成 MD5 id，内容去重。
2. 准入守卫 `_reserve_ingress_slot`（`pipeline.py:638`）按 `MAX_PENDING_DOCUMENTS` 限流。
3. 写初始 doc_status（PENDING），过滤已 PROCESSED 文档，把待处理文档投入 ingress 邮箱。
4. 拒绝入队的三态：`destructive_busy`（clear/delete）、`scanning_exclusive`（扫描分类期）、`manual_freeze_requested`；`busy`（并发索引）**不**拒绝，仅通知邮箱。

**调用链 B：处理主循环（`pipeline.py:1724` `apipeline_process_enqueue_documents`）**
1. `_next_scheduling_page`（`pipeline.py:3631`）按页取待处理文档。
2. `_pipeline_feeder`（`pipeline.py:2507`）从 ingress 邮箱喂批，`_admit_fed_document`（`pipeline.py:2307`）准入。
3. `_run_pipeline_batch`（`pipeline.py:2619`）并行驱动 `_parse_worker` → `_analyze_worker` → `_process_worker`。
4. `_process_worker` 调 `process_single_document`（`pipeline.py:4991`）：取 full_docs → 分块（F/R/V/P）→ `extract_entities` → `merge_nodes_and_edges`，终态写 doc_status。
5. `_decide_pipeline_next_step`（`pipeline.py:3818`）决定继续/排空/结束。

**调用链 C：解析分发（`pipeline.py:4409` `_parse_worker`）**
- `pending_parse` 文档在此按 `parse_engine` 分发到 parse_native（线程池）/ parse_mineru / parse_docling（外部 HTTP），解析结果回传 `parsed_data`。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `max_parallel_insert` | 并发插入任务数，默认 3，建议 ≤10 | lightrag.py |
| `max_parallel_parse_native` | native 解析线程池大小 | `pipeline.py:2732` |
| `MAX_PENDING_DOCUMENTS` | 0 禁用准入控制；负值 fail-fast | lightrag.py __post_init__ |
| `PIPELINE_SCHEDULING_PAGE_SIZE` | 调度分页；0 禁用分页 | lightrag.py |
| `process_options` | 每文档 i/t/e/!/F/R/V/P/C 选择 | `pipeline.py:777` |
| `chunk_options` | 每文档分块参数快照 | `pipeline.py:781` |

## 5. 错误与重试语义

- **FAILED 文档不自动恢复**：只能经 `/documents/scan` 或 `/documents/reprocess_failed` 的手动重试重新进入，每次重试给一次机会。
- **取消传播**：`_raise_if_cancelled`（`pipeline.py:6156`）在提取/合并阶段边界检查，命中抛 `PipelineCancelledException`。
- **失败终态**：`_finalize_doc_failure`（`pipeline.py:6401`）把文档置 FAILED 并记录阶段，不自动重试。
- **跨存储无事务**：遵循 Consistency without transactions，中间态靠重试/下次运行自愈。

## 6. 并发细节

- `pipeline_status` 是工作区共享 dict，受 `get_namespace_lock("pipeline_status")` 保护。
- `busy` 不阻塞入队；ingress 邮箱是管线唯一唤醒通道，漏通知由下次 strict scan 兜底。
- `_BatchRunContext.semaphore` 限制单文档并发；parse/analyze/process 三类 worker 并行。
- admin 图写在 `_admin_write_gate` 下与管线互斥（见 lightrag-core）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/pipeline.py` 管线调度与 worker 脚手架

**Out-of-Scope（不在本仓库源码内）**
- parse_mineru / parse_docling 外部解析 HTTP 服务——见 parser 域
- 实体提取与合并算法——见 operate-retrieval
- 存储后端——见 storage 域

## 8. 与相邻子系统交互

- 上游：`LightRAG.ainsert` / REST 文档路由 → `apipeline_enqueue_documents`
- 下游：pipeline → ingress 邮箱（L05）→ parse/analyze/process workers → operate 域提取 → 存储
- 横切：pipeline_status 共享状态（storage 域协调）、doc_status 存储

## 9. 语言专项适配口径（纯 Python 框架）

- **生产者-消费者能力缝**：feeder 生产批、workers 消费，经 `_BatchRunContext` 信号量与 ingress 邮箱解耦。
- **状态机隐藏在循环里**：单文档 `process_single_document` 是 PROCESSING→PROCESSED/FAILED 状态机，管线主循环是"取页→喂批→处理→决策"调度循环。
- **配置驱动分派**：`process_options`/`parse_engine` 字符串选择分块策略与解析引擎。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 管线组件架构图 | `pipeline-mixin-architecture.html` | architecture | standard（标签压组件；已加 labelDy 错峰） |
| 文档处理数据流 | `pipeline-mixin-dataflow.html` | dataflow | standard（并行"节点边/嵌入"标签重叠；已 labelDy 错峰） |
| 文档状态生命周期 | `pipeline-mixin-lifecycle.html` | lifecycle | standard（回迁边穿 processed、标签压状态；已调列位、去标签） |

本叶子补 dataflow（enqueue→parse→analyze→chunk→extract→merge 管道）与 lifecycle（单文档 PENDING→PROCESSING→PROCESSED/FAILED 状态机）；不补 sequence（与 dataflow 信息重复）。JSON IR 位于 `json/`。三图均 standard 渲染通过、HTML >798KB。
