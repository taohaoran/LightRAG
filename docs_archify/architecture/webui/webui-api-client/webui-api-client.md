# WebUI API 客户端（webui-api-client）

> 本文是 `webui` 域下的叶子子系统文档。域级总览见 `../webui.md`。
> 本文展开前端对后端 REST/SSE 的统一封装；不展开后端路由实现（见 `../../api/api-routers/api-routers.md`）。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`；前端 `lightrag_webui/`（React 19 + TS + Vite + Bun）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Axios 实例 | 单例 axios，baseURL=backendBaseUrl，注入 token/apiKey | `src/api/lightrag.ts:389` |
| 请求拦截器 | 自动附 `Authorization: Bearer <token>` 与 `X-API-Key`；`X-Skip-Interceptor` 跳过 | `lightrag.ts:447` |
| 响应拦截器 | 读 `x-new-token` 自动续 guest token；401 触发静默刷新 | `lightrag.ts:491` |
| Guest token 静默刷新 | 并发去重（`isRefreshingGuestToken`），重放同一 Promise | `lightrag.ts:402` `silentRefreshGuestToken` |
| 文档检索查询 | `queryText`（非流式）与 `queryTextStream`（SSE 流式） | `lightrag.ts:670`、`:879` |
| 文档增删 | insertText/insertTexts/uploadDocument/batchUpload/clearDocuments/deleteDocuments | `lightrag.ts:993-1057` |
| 文档分页查询 | getDocumentsPaginated + AbortController 取消（abortDocumentsPaginated） | `lightrag.ts:1365`、`:1326` |
| 图谱操作 | queryGraphs/getGraphLabels/updateEntity/updateRelation/checkEntityNameExists | `lightrag.ts:598`、`:1170`、`:1192` |
| 认证 | getAuthStatus/loginToServer/verifyCredentials/checkHealth | `lightrag.ts:1068`、`:1147`、`:651`、`:622` |
| 管道状态 | getPipelineStatus/cancelPipeline/scanNewDocuments/reprocessFailedDocuments | `lightrag.ts:1132`、`:1139`、`:660` |
| 错误类型 | `HttpRequestError`（带 status）、`AuthenticationRequiredError` | `lightrag.ts:476`、`errors.ts:11` |
| UI 定制拉取 | fetchUICustomization（5s 超时） | `customization.ts:106` |
| 测试桩 | `__setAxiosAdapterForTests`/`__resetPaginatedDocumentRequestsForTests` | `lightrag.ts:1356`、`:1338` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `axiosInstance` | `lightrag.ts:389` | 全局唯一 HTTP 客户端 |
| `QueryRequest`/`QueryResponse` | `lightrag.ts:192`/`:240` | 查询请求/响应类型 |
| `DocStatus` | `lightrag.ts:291` | 文档状态联合类型（pending/processing/processed/failed 等） |
| `HttpRequestError` | `lightrag.ts:476` | 带 HTTP status 的错误标记 |
| `AuthenticationRequiredError` | `errors.ts:11` | 会话失效信号（API 层已跳转登录页，查询层不渲染错误气泡） |
| `UICustomization` | `customization.ts:34` | 品牌定制内容类型 |

## 3. 关键调用链

### 3.1 带 token 的请求发出

1. 业务调用 `queryTextStream(req)`（`lightrag.ts:879`）→ 经 axiosInstance。
2. 请求拦截器（`lightrag.ts:447`）读 `localStorage['LIGHTRAG-API-TOKEN']` 与 settings store 的 apiKey，注入 `Authorization`/`X-API-Key`。
3. 响应拦截器（`lightrag.ts:491`）检查 `x-new-token`，有则写回 localStorage 并更新 auth store。
4. 401 且未在刷新中 → `silentRefreshGuestToken`（`lightrag.ts:402`）重放 `/auth-status`，并发请求共享同一 Promise。

### 3.2 流式检索

1. `queryTextStream`（`lightrag.ts:879`）POST `/query` + `stream=true`。
2. 后端 SSE 逐 chunk 推送，前端按文本累积；`isUserAbortError`（`:683`）识别用户主动取消。

### 3.3 认证失败分支

1. 响应 401 且无 token 可刷新 → 抛 `AuthenticationRequiredError`（`errors.ts:11`）。
2. API 层已跳转未认证路由；查询会话层只结束 loading、不渲染错误气泡、不写入历史。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `backendBaseUrl` | 后端 baseURL（同源或环境变量） | `lightrag.ts:389` |
| `CUSTOMIZATION_REQUEST_TIMEOUT_MS` | 5000ms | `customization.ts:92` |
| token 存储键 | `LIGHTRAG-API-TOKEN`（localStorage） | `lightrag.ts:423` |
| AbortController | 分页文档请求可取消 | `lightrag.ts:1326` |

## 5. 错误与重试语义

- **网络/HTTP 错误**：统一包成 `HttpRequestError` 带 status；4xx 被文档列表视为永久错误、不触发熔断（见 L34）。
- **401 guest token**：自动静默刷新一次；并发去重；刷新失败才抛 `AuthenticationRequiredError`。
- **用户取消**：`isUserAbortError` 识别，不当作失败。
- **无自动重试**：除 guest token 刷新外，业务错误不重试。

## 6. 并发细节

- **token 刷新去重**：`isRefreshingGuestToken` 布尔 + 共享 Promise，避免并发 401 触发多次刷新。
- **请求取消**：分页文档查询用 AbortController（`abortDocumentsPaginated`）。
- **流式响应**：SSE 长连接，前端按 chunk 累积，无并发队列。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- axios 实例、拦截器、REST/SSE 函数封装、错误类型、UI 定制拉取

**Out-of-Scope（不在本仓库源码内）**
- 后端 REST 路由（api 域）
- axios 库本身
- localStorage/sessionStorage 浏览器 API

## 8. 与相邻子系统交互

- 上游：L33 retrieval、L34 document-manager、L36 graph-viewer、L37 login-auth 等组件与 store 调用本叶子函数。
- 本叶子 → 下游：HTTP 请求 → api 域后端（L28）。
- 状态依赖：读 `useSettingsStore`（apiKey）与 `useAuthStore`（登录态）。

## 9. 语言专项适配口径（TypeScript/React 口径）

按 `language-ts.md` 落写：

- **capability seam 分组**：本叶子是 Service Definition + Provider seam——定义后端 REST 的 TS 类型契约（QueryRequest/DocStatus 等）并提供 axios 客户端实现；Consumer 是各 feature/store。
- **异步调用链**：请求→拦截器→后端→响应拦截器→错误/续 token 是 Promise 链，体现在 sequence 图。
- **外部边界**：axios 库、localStorage 浏览器 API、后端 REST 服务均标注"不在本仓库源码内"。
- **图型侧重**：architecture（axios 实例+拦截器+函数组分层）+ sequence（带 token 刷新的请求调用链）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| API 客户端架构图 | `webui-api-client-architecture.html` | architecture | standard（variant 枚举仅 default/emphasis/security/dashed，return 改 dashed 后通过） |
| 请求与 token 刷新时序 | `webui-api-client-sequence.html` | sequence | showcase |

本叶子未补 dataflow：REST 是请求-响应而非数据管道；未补 lifecycle：无单实体状态机。

JSON IR 源文件位于 `json/` 目录。
