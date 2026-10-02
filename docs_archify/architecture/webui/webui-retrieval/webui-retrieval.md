# 检索问答交互（webui-retrieval）

> 本文是 `webui` 域下的叶子子系统文档。域级总览见 `../webui.md`。
> 本文展开问答会话交互组件与流式会话 hook；不展开后端检索实现（core 域）。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`；前端 `lightrag_webui/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 问答会话 hook | `useQuerySession`：消息状态、流式提交、停止、清空、进度 | `src/features/retrieval/useQuerySession.ts:151` |
| 提交查询 | `submitQuery` useCallback：组装请求→调 queryTextStream→逐 chunk 更新消息 | `useQuerySession.ts:247` |
| 停止生成 | `handleStop`：AbortController 中止 + 冷却 | `useQuerySession.ts:612`、`:214` |
| LaTeX/think 标签分块 | 未闭合 `$$`/`$` 与 `<think>` 标签的流式渲染缓冲 | `useQuerySession.ts:47`、`:62` |
| 查询输入框 | QueryComposer 组件：输入、发送、停止按钮 | `features/retrieval/QueryComposer.tsx` |
| 消息列表 | MessageList：渲染 user/assistant 消息流与错误气泡 | `features/retrieval/MessageList.tsx` |
| 请求序列化 | serializeQueryRequest：把 UI 选项转 QueryRequest | `features/retrieval/serializeQueryRequest.ts` |
| 查询输入解析 | queryInput：输入预处理 | `features/retrieval/queryInput.ts` |
| 流式内容键 | streamedContentKey：按消息 id 定位流式累积文本 | `features/retrieval/streamedContentKey.ts` |
| 流式错误块 | streamErrorChunk：把 SSE 错误转错误气泡 | `features/retrieval/streamErrorChunk.ts` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `useQuerySession` | `useQuerySession.ts:151` | 问答会话状态机 hook |
| `MessageWithError` | `useQuerySession.ts:159` | 消息 + 错误标记类型 |
| `submitQuery` | `useQuerySession.ts:247` | 提交一次检索查询 |
| `handleStop` | `useQuerySession.ts:612` | 中止当前流 |
| `QueryComposer` | `QueryComposer.tsx` | 输入区组件 |
| `MessageList` | `MessageList.tsx` | 消息渲染组件 |

## 3. 关键调用链

### 3.1 提交一次流式查询

1. 用户在 QueryComposer 输入 → 调 `submitQuery`（`useQuerySession.ts:247`）。
2. `serializeQueryRequest`（`serializeQueryRequest.ts`）把 UI 选项转 `QueryRequest`。
3. 经 L32 `queryTextStream`（`api/lightrag.ts:879`）发起 SSE；`abortControllerRef`（`useQuerySession.ts:214`）持有句柄。
4. 逐 chunk 到达 → `setMessages` 累积 assistant 文本；未闭合 LaTeX/`<think>` 暂缓冲（`:47`/`:62`）。
5. 结束 → 写入 retrievalHistory store；`AuthenticationRequiredError` 只结束 loading 不渲染错误气泡。

### 3.2 停止生成

1. 用户点停止 → `handleStop`（`useQuerySession.ts:612`）。
2. `abortControllerRef.abort()`（`:214`）；进入停止冷却（`stopCooldownTimerRef`，`:200`）防连击。
3. `isUserAbortError`（L32 `lightrag.ts:683`）识别为用户取消，不当失败。

### 3.3 首 token 计时

- `responseStartRef`/`firstTokenRecordedRef`（`:203`/`:208`）记录首 token 延迟。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| QueryMode | naive/local/global/hybrid/mix/bypass | L32 `lightrag.ts:180` |
| 停止冷却 | stopCooldownTimerRef 防连击 | `useQuerySession.ts:200` |
| 流式累积 | streamedContentKey 按消息 id | `streamedContentKey.ts` |

## 5. 错误与重试语义

- **流式错误 chunk**：streamErrorChunk 转成 assistant 错误气泡渲染。
- **认证失败**：`AuthenticationRequiredError` 不渲染气泡、不入历史（L32 errors.ts 契约）。
- **用户取消**：不视为失败。
- **无自动重试**：失败即展示错误气泡。

## 6. 并发细节

- **单流互斥**：`isStreamingRef`（`:210`）防止并发提交。
- **AbortController**：取消在途请求；组件卸载清理 timer（`:219` useEffect）。
- **timer refs**：responseTimerRef/stopCooldownTimerRef 在卸载时 clear。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 问答交互组件、会话 hook、流式文本分块、错误气泡

**Out-of-Scope（不在本仓库源码内）**
- 后端检索生成（core 域）、SSE 协议、axios 客户端（L32）

## 8. 与相邻子系统交互

- 上游：L35 workspace 工作区、页面布局渲染 QueryComposer/MessageList。
- 本叶子 → 下游：调 L32 `queryTextStream`；写 retrievalHistory store（L38）。

## 9. 语言专项适配口径（TypeScript/React 口径）

按 `language-ts.md` 落写：

- **capability seam**：本叶子是 Consumer——消费 L32 API 客户端，组合 useState/useRef 会话状态。
- **异步调用链**：submitQuery→SSE 逐 chunk→setMessages 是 Promise/流链，体现在 sequence 图。
- **事件流**：SSE chunk 流入 → 分块 → 渲染，是事件流，可补 dataflow。
- **外部边界**：SSE 后端、浏览器 AbortController 标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 检索交互架构图 | `webui-retrieval-architecture.html` | architecture | showcase |
| 流式查询时序 | `webui-retrieval-sequence.html` | sequence | showcase（参与者标签缩短为"会话 hook"后通过） |

本叶子补 sequence（流式提交调用链）；未补 lifecycle：会话状态由 hook useState 隐式管理，无显式状态机图；未补 dataflow：SSE chunk 流与 sequence 信息重复。

JSON IR 源文件位于 `json/` 目录。
