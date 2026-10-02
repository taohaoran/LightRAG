# 图谱可视化（webui-graph-viewer）

> 本文是 `webui` 域下的叶子子系统文档。域级总览见 `../webui.md`。
> 本文展开 sigma/graphology 图谱渲染、控制组件与数据加载 hook；不展开后端图谱存储（kg 域）。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`；前端 `lightrag_webui/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 图谱主组件 | GraphViewer：挂载 sigma、组合控制组件 | `src/features/GraphViewer.tsx` |
| 数据加载 hook | useLightragGraph：fetchGraph→createSigmaGraph→sigma 实例 | `src/hooks/useLightragGraph.tsx:328` |
| 拉取图谱 | fetchGraph：按 label/maxDepth/maxNodes 拉图，指数退避重试 | `useLightragGraph.tsx:150` |
| 建图 | createSigmaGraph：RawGraph→graphology UndirectedGraph | `useLightragGraph.tsx:248` |
| 位置哈希 | hashNodeIdToPosition：节点 id→坐标 | `useLightragGraph.tsx:46` |
| 类型配色 | createTypeColorResolver | `useLightragGraph.tsx:89` |
| 帧率预算 | FRAME_BUDGET_MS=12/yieldToBrowser 避免渲染卡 UI | `:80`、`:65` |
| 17 个控制组件 | GraphControl/ZoomControl/LayoutsControl/Search/Legend/PropertiesView/MergeDialog 等 | `components/graph/` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `useLightragGraph` | `useLightragGraph.tsx:328` | 图谱加载与 sigma 实例 hook |
| `fetchGraph` | `:150` | 拉取 RawGraph |
| `createSigmaGraph` | `:248` | 构造 graphology 图 |
| `NodeType`/`EdgeType` | `:134`/`:142` | 节点/边类型 |
| sigma / graphology | 第三方库 | 渲染引擎（不在本仓库源码内） |

## 3. 关键调用链

### 3.1 加载并渲染图谱

1. GraphViewer 挂载 → `useLightragGraph`（`useLightragGraph.tsx:328`）。
2. `fetchGraph(label, maxDepth, maxNodes)`（`:150`）→ L32 `queryGraphs`（`api/lightrag.ts:598`）；失败按 MAX_FETCH_RETRIES=3 指数退避 1s→2s→4s（`:23`/`:24`）。
3. `createSigmaGraph(rawGraph)`（`:248`）建 graphology UndirectedGraph；`hashNodeIdToPosition`（`:46`）定坐标。
4. sigma 实例渲染；`yieldToBrowser`（`:65`）按 FRAME_BUDGET_MS=12（`:80`）让出主线程避免卡顿。

### 3.2 控制组件交互

- GraphSearch/LayoutsControl/ZoomControl 操作 sigma 相机/布局；PropertiesView 展示选中节点属性；MergeDialog/PropertyEditDialog 调 L32 updateEntity/updateRelation。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| MAX_FETCH_RETRIES | 3 | `useLightragGraph.tsx:23` |
| RETRY_BASE_DELAY_MS | 1000，指数退避 | `:24` |
| FRAME_BUDGET_MS | 12ms | `:80` |
| CHECK_EVERY | 256 采样间隔 | `:81` |

## 5. 错误与重试语义

- **拉图失败**：指数退避重试 3 次；仍失败展示错误。
- **节点/边解析**：parseEdgeWeight/safeNodeLabel 容错。

## 6. 并发细节

- **帧率让步**：大图谱渲染时 yieldToBrowser 分帧，避免阻塞主线程。
- **sigma 实例**：单例，组件卸载销毁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 图谱加载 hook、sigma 封装、17 个控制组件

**Out-of-Scope（不在本仓库源码内）**
- sigma.js / graphology 渲染库、后端图谱存储

## 8. 与相邻子系统交互

- 上游：页面布局渲染 GraphViewer。
- 本叶子 → 下游：调 L32 queryGraphs/updateEntity/updateRelation；读 graph store（L38）。

## 9. 语言专项适配口径（TypeScript/React 口径）

按 `language-ts.md` 落写：

- **capability seam**：Consumer——消费 L32 图谱 API，封装第三方 sigma/graphology 为 React hook。
- **异步调用链**：fetch→重试→建图→渲染 是 Promise 链。
- **外部边界**：sigma.js、graphology 标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 图谱查看架构图 | `webui-graph-viewer-architecture.html` | architecture | showcase |
| 图谱加载时序 | `webui-graph-viewer-sequence.html` | sequence | showcase（消息 y 间距调整后通过） |

本叶子补 sequence（拉图+重试+建图调用链）；未补 dataflow：与 sequence 重复；未补 lifecycle：sigma 实例由 hook 管，无显式状态机。

JSON IR 源文件位于 `json/` 目录。
