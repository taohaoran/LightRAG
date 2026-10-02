# 摄入邮箱与扫描任务存储（pipeline-ingress）

> 本文是 `pipeline` 域下的叶子子系统文档。域级总览见 `../pipeline.md`。
> 本文展开 workspace 级摄入邮箱（管线唯一唤醒通道）与扫描任务状态存储；
> 消费邮箱的 feeder 见 `../pipeline-mixin/pipeline-mixin.md`。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 摄入邮箱协议 | `PipelineIngress` Protocol 抽象投递/订阅接口 | `lightrag/kg/pipeline_ingress.py:215` |
| 邮箱实现 | `PipelineIngressMailbox` 单工作区邮箱 | `pipeline_ingress.py:256` |
| 邮箱枢纽 | `PipelineIngressHub` 按 workspace 路由到各邮箱 | `pipeline_ingress.py:503` |
| 多进程邮箱 | `ManagerPipelineIngress`（多 worker 用 Manager().dict()） | `pipeline_ingress.py:677` |
| 单进程邮箱 | `AsyncioPipelineIngress`（单进程 asyncio） | `pipeline_ingress.py:743` |
| 手动重试发布 | 手动重试文档经邮箱投递，带一次尝试配额 | `pipeline_ingress.py:130` |
| 消息校验 | `_validate_document_message` / `_validate_manual_request` | `pipeline_ingress.py:184/204` |
| 扫描任务存储 | `AsyncioScanJobStore` 记录扫描任务状态与样本桶 | `lightrag/kg/scan_job_store.py:249` |
| 扫描任务枢纽 | `ScanJobStoreHub` 按 workspace 路由 | `scan_job_store.py:623` |
| 扫描状态枚举 | `ScanJobStatus` / `ScanJobCreateOutcome` / `ScanJobUpdateConflict` | `scan_job_store.py:83/112/123` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `PipelineIngress` | `pipeline_ingress.py:215` | 邮箱协议（投递/订阅/关闭） |
| `PipelineIngressMessage` | `pipeline_ingress.py:113` | 邮箱消息载荷 |
| `PipelineIngressMailbox` | `pipeline_ingress.py:256` | 单工作区有界邮箱 |
| `get_pipeline_ingress(workspace)` | 共享存储 | 取工作区邮箱（管线唯一唤醒通道） |
| `AsyncioScanJobStore` | `scan_job_store.py:249` | 扫描任务记录存储 |
| `ScanJobStatus` | `scan_job_store.py:83` | 扫描任务状态枚举 |

## 3. 关键调用链

**调用链 A：入队唤醒（pipeline 契约）**
1. `apipeline_enqueue_documents` 写 doc_status 后，把消息投入 `get_pipeline_ingress(workspace)` 邮箱。
2. 正在等待的管线处理循环被邮箱消息唤醒，取新批处理。
3. 若管线未在跑，邮箱消息不丢失——下次 strict scan 兜底发现待处理文档。

**调用链 B：手动重试**
1. `/documents/reprocess_failed` 经 `apipeline_enqueue_error_documents` 把 FAILED 文档标记待处理并投入邮箱。
2. 每条手动重试只给**一次**尝试；失败后回到 FAILED，不再自动入队。

**调用链 C：扫描任务**
1. 扫描任务在 `AsyncioScanJobStore` 建记录（`ScanJobCreateOutcome`）。
2. 分类期持 `scanning_exclusive` 拒绝外部写；处理期投递文档到邮箱。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `manual_channel_capacity` | 手动重试通道容量 | `pipeline_ingress.py:85` |
| 工作区隔离 | 每 workspace 独立邮箱/扫描存储 | hub 路由 |

## 5. 错误与重试语义

- **漏通知自愈**：邮箱是"加速唤醒"，不是唯一真相来源；doc_status 才是真相。漏投消息由下次 strict scan 兜底。
- **手动重试一次**：FAILED 文档手动重试只给一次机会，失败后回 FAILED。
- **扫描冲突**：`ScanJobUpdateConflict` 处理并发更新冲突。

## 6. 并发细节

- 邮箱按 workspace 隔离；多进程部署用 `ManagerPipelineIngress` 共享内存，单进程用 `AsyncioPipelineIngress`。
- hub 是代理层（`BaseProxy`），按 workspace 路由到对应邮箱实例。
- 邮箱是 asyncio 唤醒通道，不是持久队列。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/kg/pipeline_ingress.py` 邮箱协议与多实现
- `lightrag/kg/scan_job_store.py` 扫描任务存储

**Out-of-Scope（不在本仓库源码内）**
- 邮箱消费方 feeder——见 pipeline-mixin
- 持久 doc_status——见 storage 域

## 8. 与相邻子系统交互

- 上游：pipeline-mixin 入队 → 投递消息到本邮箱
- 下游：feeder（pipeline-mixin）→ 订阅邮箱被唤醒
- 横切：workspace 命名空间隔离（storage 协调）

## 9. 语言专项适配口径（纯 Python 框架）

- **双轨实现**：同一 `PipelineIngress` 协议有 `AsyncioPipelineIngress`（单进程）与 `ManagerPipelineIngress`（多进程共享内存）两套实现，按部署形态选择，是典型双轨模式。
- **协议即扩展点**：`PipelineIngress` 是 Protocol，hub 按 workspace 路由，便于测试替身。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 摄入邮箱与扫描存储架构 | `pipeline-ingress-architecture.html` | architecture | standard（一次渲染通过） |
| 入队唤醒时序 | `pipeline-ingress-sequence.html` | sequence | standard（一次渲染通过） |

本叶子不补 dataflow/lifecycle（邮箱是唤醒通道非数据管道；文档状态机已由 pipeline-mixin lifecycle 覆盖）。JSON IR 位于 `json/`。两图均 standard 渲染通过、HTML >798KB。
