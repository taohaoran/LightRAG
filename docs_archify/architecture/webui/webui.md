# webui 域（前端域）

> 本域是 LightRAG 的 React 19 + TypeScript 前端：把后端 REST/SSE 包装成检索问答、
> 文档管理、图谱可视化、登录鉴权与工作区交互。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`；前端 `lightrag_webui/`（Vite + Bun + Tailwind + i18next）。

## 域职责

- **API 客户端封装**：单例 axios、拦截器、token 续期、REST/SSE 函数组。
- **检索问答交互**：流式会话 hook、消息渲染、停止/取消。
- **文档管理**：分页列表、上传/删除、刷新队列与熔断器。
- **工作区模式**：认证引导、凭据探测、视图分发。
- **图谱可视化**：sigma/graphology 渲染与 17 个控制组件。
- **登录鉴权**：登录表单、同意、身份逻辑。
- **全局状态与基础设施**：zustand stores、lib 工具、引导与迁移。

## 叶子索引

| 叶子 | 职责 | 文档 |
|---|---|---|
| L32 webui-api-client | axios 实例、拦截器、token 续期、REST/SSE 函数 | [webui-api-client/webui-api-client.md](webui-api-client/webui-api-client.md) |
| L33 webui-retrieval | 流式问答会话、QueryComposer/MessageList | [webui-retrieval/webui-retrieval.md](webui-retrieval/webui-retrieval.md) |
| L34 webui-document-manager | 文档列表、上传/删除、刷新队列与熔断 | [webui-document-manager/webui-document-manager.md](webui-document-manager/webui-document-manager.md) |
| L35 webui-workspace | 工作区入口、认证引导、视图分发 | [webui-workspace/webui-workspace.md](webui-workspace/webui-workspace.md) |
| L36 webui-graph-viewer | sigma/graphology 图谱渲染与控制组件 | [webui-graph-viewer/webui-graph-viewer.md](webui-graph-viewer/webui-graph-viewer.md) |
| L37 webui-login-auth | 登录表单、同意、身份逻辑 | [webui-login-auth/webui-login-auth.md](webui-login-auth/webui-login-auth.md) |
| L38 webui-state-lib | zustand stores、lib 工具、引导迁移 | [webui-state-lib/webui-state-lib.md](webui-state-lib/webui-state-lib.md) |

## 域级机制细节

### 状态分层

- L38 stores 是全局状态 single source；feature 组件订阅 store、调 L32 API。
- settings/querySettings 各有独立 persist key，避免互相污染。

### 错误分流

- L32 `AuthenticationRequiredError`：API 层已跳转登录，查询层不渲染错误气泡。
- 4xx 永久错误不触发文档列表熔断器。

### 异步约定

- SSE 流式查询逐 chunk 累积；AbortController 取消；用户取消不当失败。

## 图表清单

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| webui 域架构图 | `webui-architecture.html` | architecture | showcase（删穿越边后通过） |
| 用户交互数据流 | `webui-dataflow.html` | dataflow | showcase（删穿越流后通过） |
| 登录后启动时序 | `webui-sequence.html` | sequence | showcase |

域图省略 lifecycle：前端会话状态由 React 条件渲染与 hook 隐式管理，无显式状态机图；省略 workflow：无多角色审批流。

JSON IR 位于 `json/` 目录。
