# pipeline 域总览

> 本文是 pipeline 域的域级总览，链接到本域全部叶子子系统文档。
> 输出根：`docs_archify/architecture/`。
> 源码基准：`lightrag-hku` main，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 域职责

pipeline 域负责文档摄入管线：从入队、解析、分块、实体提取到写存储，以及多模态上下文装配与唤醒机制。它是 LightRAG 数据写入侧的主干。

关键机制（权威契约口径）：
- **并发契约**：`pipeline_status` 共享 dict + `get_namespace_lock`；`busy` 不阻塞 enqueue；`destructive_busy`/`scanning_exclusive`/`manual_freeze_requested` 三态拒绝入队。
- **ingress mailbox**：管线唯一唤醒通道；doc_status 才是真相，漏通知由 strict scan 兜底。
- **FAILED 文档**：只经手动重试进入，每次一次尝试。
- **Purge 恢复**：失败闭合（RecoveryAnchorMissingError → HTTP 409），除非 anchors/pre_graph/journal/empty_scope 四证之一。

## 叶子索引

| 叶子 | 职责 | 文档 |
|---|---|---|
| L04 pipeline-mixin | _PipelineMixin 摄入管线、解析器分发、多模态分析、worker 脚手架 | [pipeline-mixin.md](pipeline-mixin/pipeline-mixin.md) |
| L05 pipeline-ingress | workspace 摄入邮箱、扫描任务存储 | [pipeline-ingress.md](pipeline-ingress/pipeline-ingress.md) |
| L06 utils-pipeline | doc-status 字段访问、文档身份哈希、解析产物路径、载荷归一 | [utils-pipeline.md](utils-pipeline/utils-pipeline.md) |
| L07 multimodal-context | 多模态上下文装配、提示词、分块 schema 归一 | [multimodal-context.md](multimodal-context/multimodal-context.md) |

## 域级机制细节

- **L04 ↔ L05**：入队后投递消息到 ingress mailbox，feeder 订阅被唤醒。
- **L04 ↔ L06**：处理文档时用 utils-pipeline 做字段访问、去重、路径解析。
- **L04 ↔ L07**：多模态分析 worker 调 multimodal-context 装配上下文喂 VLM。
- **L04 ↔ core**：`apipeline_*` 经 `_PipelineMixin` 被主类 `ainsert` 调用。

## 域级图表

| 图 | 文件 | 类型 | 说明 |
|---|---|---|---|
| pipeline 域组件架构 | `pipeline-architecture.html` | architecture | 四叶子协作 |
| 摄入管线数据流 | `pipeline-dataflow.html` | dataflow | 入队→解析→提取→写存储 |
| 文档状态机 | `pipeline-lifecycle.html` | lifecycle | PENDING→PROCESSING→终态 |

本域不补 sequence（入队时序已在 L05 叶子图覆盖）。JSON IR 位于 `json/`。
