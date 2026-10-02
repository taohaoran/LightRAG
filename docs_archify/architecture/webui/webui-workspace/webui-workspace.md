# 工作区模式（webui-workspace）

> 本文是 `webui` 域下的叶子子系统文档。域级总览见 `../webui.md`。
> 本文展开工作区入口、登录引导、凭据探测与欢迎/查询视图；不展开检索会话本身（L33）。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`；前端 `lightrag_webui/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 工作区根组件 | WorkspaceApp：按认证/凭据状态分发欢迎/查询/登录视图 | `src/features/workspace/WorkspaceApp.tsx:37` |
| 查询视图 | WorkspaceQueryView：挂载检索交互（L33） | `features/workspace/WorkspaceQueryView.tsx` |
| 欢迎页 | WorkspaceWelcome：空状态引导 | `features/workspace/WorkspaceWelcome.tsx` |
| 空状态 | WorkspaceEmptyState | `features/workspace/WorkspaceEmptyState.tsx` |
| 登录引导 | authBootstrap：启动时探测认证态、guest token 引导 | `features/workspace/authBootstrap.ts` |
| 凭据探测 | credentialProbe：验证 apiKey/token 是否可用 | `features/workspace/credentialProbe.ts` |
| 查询设置快照 | querySettingsSnapshot：进入工作区时冻结设置快照 | `features/workspace/querySettingsSnapshot.ts` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `WorkspaceApp` | `WorkspaceApp.tsx:37` | 工作区入口分发组件 |
| `authBootstrap` | `authBootstrap.ts` | 启动认证引导 |
| `credentialProbe` | `credentialProbe.ts` | 凭据可用性探测 |
| `querySettingsSnapshot` | `querySettingsSnapshot.ts` | 设置快照 |

## 3. 关键调用链

### 3.1 工作区启动

1. WorkspaceApp（`WorkspaceApp.tsx:37`）挂载。
2. `authBootstrap`（`authBootstrap.ts`）探测认证态：未配置认证→guest token；已配置→跳登录页。
3. `credentialProbe`（`credentialProbe.ts`）验证 apiKey/token；失败→登录视图。
4. 凭据就绪 → 渲染 WorkspaceQueryView（挂载 L33 useQuerySession）。

### 3.2 设置快照

- 进入工作区时 `querySettingsSnapshot`（`querySettingsSnapshot.ts`）冻结查询设置，避免运行中被全局设置改动打断。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 认证模式 | 未配 AUTH_ACCOUNTS→guest；已配→登录 | L29/L37 |
| 凭据探测 | credentialProbe | `credentialProbe.ts` |

## 5. 错误与重试语义

- **凭据失效**：credentialProbe 失败 → 跳登录页，不进入查询视图。
- **认证失败**：复用 L32 `AuthenticationRequiredError` 分支。

## 6. 并发细节

- 启动探测串行：authBootstrap → credentialProbe → 渲染。
- 无长驻轮询。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 工作区分发、登录引导、凭据探测、欢迎/查询视图

**Out-of-Scope（不在本仓库源码内）**
- 登录表单本身（L37）、检索会话（L33）

## 8. 与相邻子系统交互

- 上游：根路由渲染 WorkspaceApp。
- 本叶子 → 下游：凭据就绪后挂载 L33 retrieval；未就绪跳 L37 login。

## 9. 语言专项适配口径（TypeScript/React 口径）

按 `language-ts.md` 落写：

- **capability seam**：本叶子是编排/路由 seam——按认证态分发到不同视图。
- **异步调用链**：authBootstrap→credentialProbe→渲染 是启动期 Promise 链。
- **外部边界**：后端 /auth-status、浏览器路由标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 工作区架构图 | `webui-workspace-architecture.html` | architecture | showcase（删穿越边后通过） |
| 工作区启动时序 | `webui-workspace-sequence.html` | sequence | showcase |

本叶子补 sequence（启动探测调用链）；未补 lifecycle：工作区状态由条件渲染表达，与 sequence 重复；未补 dataflow：无数据管道。

JSON IR 源文件位于 `json/` 目录。
