# core 域总览

> 本文是 core 域的域级总览，链接到本域全部叶子子系统文档。
> 输出根：`docs_archify/architecture/`（项目已有 `docs/`，按规则用 `docs_archify`）。
> 源码基准：`lightrag-hku` main，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 域职责

core 域是 LightRAG 的"核心编排层"：组装主类、驱动检索与提取、管理 LLM 角色。它不持有持久存储（那是 storage 域），不直接处理文档摄入调度（那是 pipeline 域），而是把二者粘合为统一的 `LightRAG` 公共 API。

关键机制（权威契约口径）：
- **mixin 组合**：`LightRAG → _RoleLLMMixin → _StorageMigrationMixin → _PipelineMixin → object`，`@final` 保留。
- **初始化陷阱**：必须 `await rag.initialize_storages()` 之后才可安全使用。
- **admin 写门**：`_admin_write_gate` 在图存储声明 `requires_single_writer` 时串行化管理写。
- **查询模式**：local / global / hybrid / naive / mix（mix 推荐配 reranker）。
- **跨存储无事务**：可接受"自愈或方向无害"的不一致残留。

## 叶子索引

| 叶子 | 职责 | 文档 |
|---|---|---|
| L01 lightrag-core | LightRAG 主类组合、admin 写门、initialize/finalize、ainsert/aquery 入口、addon_params、存储迁移 | [lightrag-core.md](lightrag-core/lightrag-core.md) |
| L02 operate-retrieval | 实体/关系提取、分块编排、5 模式检索、重排、查询校验 | [operate-retrieval.md](operate-retrieval/operate-retrieval.md) |
| L03 llm-roles | RoleSpec/ROLES 注册表、角色规范化、包装器重建、运行时配置更新 | [llm-roles.md](llm-roles/llm-roles.md) |

## 域级机制细节

- **L01 ↔ L03**：主类在初始化时经 `_RoleLLMMixin` 构建各角色 LLM 包装器；`addon_params` 变化时重建包装器。
- **L01 ↔ L02**：`aquery` 委托 operate 的多模式检索；`ainsert` 委托 pipeline 域（core 域只暴露入口）。
- **L02 ↔ L03**：提取/检索都通过角色注册表拿到对应 LLM 函数。

## 域级图表

| 图 | 文件 | 类型 | 说明 |
|---|---|---|---|
| core 域组件架构 | `core-architecture.html` | architecture | 三叶子与外部边界 |
| 主类初始化时序 | `core-sequence.html` | sequence | initialize_storages → 角色构建 |
| 查询管线数据流 | `core-dataflow.html` | dataflow | aquery 多模式检索 |

本域不补 lifecycle（文档状态机归 pipeline 域）。JSON IR 位于 `json/`。
