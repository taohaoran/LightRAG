# 文档管理界面（webui-document-manager）

> 本文是 `webui` 域下的叶子子系统文档。域级总览见 `../webui.md`。
> 本文展开文档列表、上传/删除、状态轮询与熔断器；不展开后端文档处理管道（pipeline 域）。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`；前端 `lightrag_webui/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 文档管理主组件 | DocumentManager：分页列表、状态过滤、排序、多选 | `src/features/DocumentManager.tsx:381` |
| 状态计数聚合 | getCountValue/getAggregateCount/hasActiveDocumentsStatus | `DocumentManager.tsx:76`、`:86`、`:89` |
| 文件名截断 | getDisplayFileName | `DocumentManager.tsx:94` |
| 文档详情对话框 | DocumentStatusDetailsDialog | `DocumentManager.tsx:233` |
| 上传对话框 | UploadDocumentsDialog | `components/documents/UploadDocumentsDialog.tsx` |
| 删除对话框 | DeleteDocumentsDialog/ClearDocumentsDialog | `components/documents/` |
| 管道状态对话框 | PipelineStatusDialog | `components/documents/PipelineStatusDialog.tsx` |
| 刷新队列 | createRefreshQueue：合并/串行化刷新请求 | `features/documentRefreshQueue.ts` |
| 刷新熔断器 | 连续失败后开熔断，避免打后端 | `features/documentRefreshCircuitBreaker.ts` |
| 刷新错误分类 | 4xx 永久错误 vs 网络错误，4xx 不触熔断 | `features/documentRefreshErrors.ts` |
| 删除探测 | documentDeletionProbe：删除后核对 | `features/documentDeletionProbe.ts` |
| 恢复告警 | documentRecoveryWarnings | `features/documentRecoveryWarnings.ts` |
| 状态计数/过滤 | documentStatusCounts/documentStatusFilters | `features/` |
| 文档标题 | documentTitle | `lib/documentTitle.ts` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `DocumentManager` | `DocumentManager.tsx:381` | 文档管理页面组件 |
| `createRefreshQueue` | `documentRefreshQueue.ts` | 刷新请求队列 |
| 熔断器 | `documentRefreshCircuitBreaker.ts` | 连续失败熔断 |
| `DocStatusResponse`/`PaginationInfo` | L32 `lightrag.ts:300`/`:330` | 文档状态/分页类型 |

## 3. 关键调用链

### 3.1 加载文档列表

1. DocumentManager 挂载 → `getDocumentsPaginated`（L32 `lightrag.ts:1365`）按 `DocumentsRequest` 拉页。
2. 返回 `PaginatedDocsResponse` → `setCurrentPageDocs`/`setPagination`（`DocumentManager.tsx:415`/`:416`）。
3. 状态计数经 `setStatusCounts`（`:424`）；4xx 错误被 documentRefreshErrors 判为永久错误，不触熔断。

### 3.2 轮询刷新

1. `createRefreshQueue`（`:935`）把刷新请求串行化合并。
2. 连续失败 → `documentRefreshCircuitBreaker` 开闸；恢复后半开探测。
3. 删除操作后 `documentDeletionProbe` 核对服务端状态。

### 3.3 上传/删除

1. UploadDocumentsDialog → `uploadDocument`/`batchUploadDocuments`（L32 `lightrag.ts:1003`/`:1026`）。
2. DeleteDocumentsDialog → `deleteDocuments`（L32 `:1057`）；Clear → `clearDocuments`（`:1048`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 排序 | 默认 updated_at desc | `DocumentManager.tsx:438` |
| 状态过滤 | all/pending/processing/processed/failed | `:442`、`documentStatusFilters.ts` |
| 刷新失败 toast | DOCUMENT_REFRESH_FAILURE_TOAST_ID | `:379` |
| 熔断器阈值 | documentRefreshCircuitBreaker.ts | |

## 5. 错误与重试语义

- **4xx 永久错误**：不触发熔断（客户端错，重试无意义）。
- **网络/5xx**：进熔断器计数，连续失败后开闸暂停轮询。
- **删除后核对**：deletionProbe 确认删除生效。
- **初始加载失败**：initialLoadError 展示错误区，不静默。

## 6. 并发细节

- **刷新队列**：createRefreshQueue 串行合并，避免并发刷新打爆后端。
- **AbortController**：分页请求可取消（L32 abortDocumentsPaginated）。
- **多选状态**：selectedDocIds 在翻页/过滤切换时重置（`:1442` previousSelectionDeps）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 文档管理 UI、刷新队列、熔断器、状态计数/过滤

**Out-of-Scope（不在本仓库源码内）**
- 后端文档处理管道（pipeline 域）、文件上传存储

## 8. 与相邻子系统交互

- 上游：页面布局渲染 DocumentManager。
- 本叶子 → 下游：调 L32 文档/管道 API；读 pipelineStatus 展示。

## 9. 语言专项适配口径（TypeScript/React 口径）

按 `language-ts.md` 落写：

- **capability seam**：Consumer——消费 L32 文档 API，组合 useState/useReducer 管理分页/过滤/多选。
- **事件流**：轮询刷新 → 队列合并 → 熔断器是事件流。
- **外部边界**：后端文档 API、浏览器 AbortController 标注"不在本仓库源码内"。
- **图型侧重**：architecture（组件+队列+熔断器分层）+ dataflow（轮询刷新数据流）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 文档管理架构图 | `webui-document-manager-architecture.html` | architecture | showcase |
| 文档轮询刷新数据流 | `webui-document-manager-dataflow.html` | dataflow | standard |

本叶子补 dataflow（轮询刷新→队列→熔断→渲染 的数据管道）；未补 sequence：与 dataflow 信息重复；未补 lifecycle：文档状态机在后端 pipeline 域。

JSON IR 源文件位于 `json/` 目录。
