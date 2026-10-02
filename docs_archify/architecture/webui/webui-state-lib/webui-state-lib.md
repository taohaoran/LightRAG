# 全局状态库与基础设施（webui-state-lib）

> 本文是 `webui` 域下的叶子子系统文档。域级总览见 `../webui.md`。
> 本文展开 zustand stores、lib 工具、bootstrap/migrations 等全局基础设施；
> 深读 settings/graph/querySettings 三个代表 store，其余以共性表格说明并披露缺口。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`；前端 `lightrag_webui/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 设置 store | useSettingsStore：apiKey、主题、语言等，zustand persist 到 localStorage | `src/stores/settings.ts:100` |
| 图谱 store | useGraphStore：RawGraph 持有、布局监督、边更新队列 | `stores/graph.ts:192` |
| 查询设置 store | useQuerySettingsStore：独立 persist key | `stores/querySettings.ts:47` |
| 检索历史 store | retrievalHistory/webui/workspace 三套 | `stores/retrievalHistory.ts` 等 |
| 其他 store | aiContentNotice/customization/state/healthCheckRace/documentTitleSync/storageSplit/tokenValidation | `stores/` |
| 通用工具 lib | base64url/browserLanguage/constants/fileTypes/guardedStorage/pathPrefix/queryDefaults/runtimeConfig/storageKeys/storageQuota/utils/versionCheckCache/loginIdentity | `src/lib/` |
| 引导/迁移 | bootstrap/目录、migrations/目录（localStorage schema 迁移） | `src/bootstrap/`、`src/migrations/` |
| 上下文/hooks/services | contexts/、hooks/、services/ | `src/` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `useSettingsStore` | `stores/settings.ts:100` | 全局设置（persist） |
| `useGraphStore` | `stores/graph.ts:192` | 图谱状态 |
| `useQuerySettingsStore` | `stores/querySettings.ts:47` | 查询设置（独立 persist） |
| `RawGraph` | `stores/graph.ts:55` | 图谱数据类 |
| `guardedStorage` | `lib/guardedStorage.ts` | 带错误防护的 localStorage 封装 |
| `storageQuota` | `lib/storageQuota.ts` | localStorage 配额管理 |

## 3. 关键调用链

### 3.1 设置持久化（代表 store）

1. 组件调 `useSettingsStore.getState().setApiKey(...)`（`stores/settings.ts:100`）。
2. zustand `persist` 中间件（`:2`）写 localStorage；`partialize` 决定哪些字段持久化。
3. 迁移逻辑（`:243`）处理旧 schema 字段。

### 3.2 图谱状态（代表 store）

1. L36 `useLightragGraph` 拉图后写 `useGraphStore`（`stores/graph.ts:192`）。
2. `RawGraph` 类（`:55`）持有节点/边；`LayoutSupervisorHandle`（`:9`）管布局。
3. `EdgeToUpdate`（`:49`）队列批量更新边。

### 3.3 查询设置快照（代表 store）

- `useQuerySettingsStore`（`querySettings.ts:47`）独立 persist key `lightrag::query-settings-storage`，与 settings store 解耦。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| persist keys | settings/querySettings 各有独立 key | `settings.ts`、`querySettings.ts:14` |
| storageQuota | localStorage 配额 | `lib/storageQuota.ts` |
| runtimeConfig | 运行时配置 | `lib/runtimeConfig.ts` |

## 5. 错误与重试语义

- **guardedStorage**：localStorage 读写异常（隐私模式/配额）被捕获降级，不崩应用。
- **storageQuota**：超配额时清理旧历史。
- **迁移**：旧 schema 字段迁移失败时回退默认值。

## 6. 并发细节

- **zustand**：单 store 同步更新，无锁；跨 store 用 selector 订阅。
- **healthCheckRace**：健康检查竞态处理。
- **documentTitleSync**：标题同步。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 全局状态、工具库、引导与迁移

**Out-of-Scope（不在本仓库源码内）**
- zustand 库、React 本身、localStorage 浏览器 API

## 8. 与相邻子系统交互

- 上游：所有 feature 组件与 hook 消费这些 store/lib。
- 本叶子 → 下游：被 L32–L37 广泛依赖；自身不发请求（请求在 L32）。

## 9. 语言专项适配口径（TypeScript/React 口径）

按 `language-ts.md` 落写：

- **capability seam**：本叶子是 Provider seam——zustand stores 提供全局状态，lib 提供纯函数工具；各 feature 是 Consumer。
- **依赖图**：stores 经 persist 中间件依赖 localStorage；lib 是无状态工具。
- **外部边界**：zustand、浏览器 localStorage 标注"不在本仓库源码内"。
- **图型侧重**：architecture（store 分层 + lib 工具）+ dataflow（用户交互→store→persist 流）。

**未深读清单（披露缺口）**：stores/ 下 aiContentNotice、customization、documentTitleSync、state、healthCheckRace、webuiRetrievalHistory、workspaceRetrievalHistory、storageSplit、tokenValidation 未逐一精读；lib/ 下 base64url、browserLanguage、constants、fileTypes、pathPrefix、queryDefaults、versionCheckCache 等以共性工具归类；contexts/services/hooks/bootstrap/migrations 未逐一展开。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 状态库架构图 | `webui-state-lib-architecture.html` | architecture | showcase |
| 状态持久化数据流 | `webui-state-lib-dataflow.html` | dataflow | showcase |

本叶子补 dataflow（交互→store→persist→localStorage 的数据流）；未补 sequence：与 dataflow 重复；未补 lifecycle：store 状态由 React 渲染驱动，无显式状态机。

JSON IR 源文件位于 `json/` 目录。
