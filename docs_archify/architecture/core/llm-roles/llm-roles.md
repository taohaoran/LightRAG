# 角色化 LLM 配置（llm-roles）

> 本文是 `core` 域下的叶子子系统文档。域级总览见 `../core.md`。
> 本文展开 `_RoleLLMMixin` 如何把一个基础 LLM 函数按角色（extract/keyword/query/vlm）拆成可独立配置、独立并发队列、独立超时的包装器注册表；
> 实际 LLM provider 绑定见 `../../llm/llm-openai-family/llm-openai-family.md` 与 `../../llm/llm-provider-matrix/llm-provider-matrix.md`。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 角色注册表 | 4 个静态 `RoleSpec`（extract/keyword/query/vlm），新增角色只改一行 | `lightrag/llm_roles.py:52` `ROLES` |
| 角色规范化 | 大小写/空白归一，非法角色名抛 ValueError | `llm_roles.py:124` `_normalize_llm_role` |
| 角色 LLM 配置 | `RoleLLMConfig`：func/kwargs/max_async/timeout/metadata，None 回退基础设置 | `llm_roles.py:62` |
| 包装器构建 | `_wrap_llm_role_func` 给每角色套优先级限并发异步队列 | `llm_roles.py:180` |
| 全量重建 | `_rebuild_role_llm_funcs` 按注册 builder 重建所有角色 | `llm_roles.py:202` |
| 单角色重建 | `_rebuild_single_role_llm_func` | `llm_roles.py:212` |
| 运行时配置更新 | `update_llm_role_config` / `aupdate_llm_role_config` 热更新单角色 | `llm_roles.py:393/428` |
| 旧队列清理 | 退役包装器的队列后台排空，避免悬挂任务 | `llm_roles.py:228/265` |
| 密钥脱敏 | 元数据导出时按 `_SECRET_MARKERS` 脱敏 | `llm_roles.py:482/488` |
| 队列状态观测 | 各角色 embedding/rerank 队列状态查询 | `llm_roles.py:594/611/617` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `RoleSpec` | `llm_roles.py:31` | 冻结描述符：name/env_prefix/queue_name |
| `ROLES` / `ROLES_BY_NAME` | `llm_roles.py:52/59` | 角色注册表与名称索引 |
| `RoleLLMConfig` | `llm_roles.py:62` | 初始化期每角色覆盖配置 |
| `_RoleLLMState` | `llm_roles.py:82` | 单角色运行时状态（raw_func/kwargs/wrapped） |
| `_RoleLLMMixin` | `llm_roles.py:93` | 混入 LightRAG 的角色 LLM 运行时 |
| `role_llm_funcs` | `llm_roles.py:149` | 对外只读映射：role → 可调用包装器 |

## 3. 关键调用链

**调用链 A：包装器构建（`llm_roles.py:202` `_rebuild_role_llm_funcs`）**
1. 遍历 `ROLES`，对每角色取 `_get_effective_role_llm_kwargs`（`llm_roles.py:162`）与 timeout/max_async。
2. `_wrap_llm_role_func`（`llm_roles.py:180`）用 `priority_limit_async_func_call` 套优先级限并发队列，queue_name 来自 `RoleSpec.queue_name`。
3. 旧 wrapped 经 `_schedule_retired_llm_queue_cleanup`（`llm_roles.py:228`）后台排空。

**调用链 B：运行时热更新（`llm_roles.py:428` `aupdate_llm_role_config`）**
1. 校验角色名（`_normalize_llm_role`）。
2. `_apply_llm_role_config_update`（`llm_roles.py:309`）写入新配置。
3. `_rebuild_single_role_llm_func`（`llm_roles.py:212`）只重建该角色，退役旧队列。

**调用链 C：消费方取函数（`llm_roles.py:149` `role_llm_funcs`）**
- operate 域经 `global_config["role_llm_funcs"]["extract"/"query"]` 取包装器调用；查询路径还带 `_priority` 参数（见 operate-retrieval `kg_query`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `role_llm_configs` | 每角色 RoleLLMConfig 覆盖；None 字段回退 llm_model_func 等基础设置 | `llm_roles.py:62` |
| `{ROLE}_LLM_BINDING` / `{ROLE}_MAX_ASYNC_LLM` / `{ROLE}_LLM_TIMEOUT` | API env 层按 env_prefix 生成 | `llm_roles.py:44` |
| `max_async` | 每角色并发上限；未配置时 seed 自 `{ROLE}_MAX_ASYNC_LLM` | `llm_roles.py:76` |

## 5. 错误与重试语义

- **非法角色**：`_normalize_llm_role` 对不在 `ROLE_NAMES` 的角色名立即 `ValueError`。
- **旧队列排空**：重建时旧包装器不立即丢弃，后台任务 `wait_for_retired_llm_queues` 等在飞请求完成，避免悬挂。
- **密钥安全**：`get_llm_role_config` 导出前按 `_SECRET_MARKERS` 脱敏，`thinking_token_budget` 等含 token 子串但非密钥的键列入 `_SAFE_OPTION_KEYS` 白名单。

## 6. 并发细节

- 每角色独立优先级限并发队列（`priority_limit_async_func_call`），extract/keyword/query/vlm 互不阻塞。
- 查询 LLM 以 `DEFAULT_QUERY_PRIORITY` 更高优先级进入队列。
- 退役队列清理是后台 asyncio.Task，`finalize` 时 `_finalize_retired_llm_queue_cleanup` 收口。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/llm_roles.py` 角色注册表、包装器构建、热更新、队列观测

**Out-of-Scope（不在本仓库源码内）**
- 各 provider 实际 HTTP 客户端（OpenAI/Ollama/Gemini…）——见 llm 域
- 优先级限并发队列底层实现——在 `lightrag/utils.py`

## 8. 与相邻子系统交互

- 上游：`LightRAG.__post_init__` 装配本 mixin；`register_role_llm_builder` 注册 provider builder
- 下游：本 mixin → `role_llm_funcs` → operate 域（extract/query）与 pipeline 域（vlm/extract）
- 横切：llm 域各 provider builder 把具体函数注册进来

## 9. 语言专项适配口径（纯 Python 框架）

- **注册表即扩展点**：`ROLES` 是冻结注册表；新增角色=加一行 RoleSpec，env 循环/队列观测/更新流全部自动迭代，是典型"注册表驱动"能力缝。
- **配置驱动分派**：同一基础 `llm_model_func` 经每角色 `RoleLLMConfig` 切出不同并发/超时/kwargs 包装器。
- **可观测钩子**：队列状态、密钥脱敏、metadata 注入是框架级横切缝。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 角色 LLM 注册表与包装器架构 | `llm-roles-architecture.html` | architecture | standard（"退役"标签压包装器组件；已加 labelDy） |
| 包装器重建时序 | `llm-roles-sequence.html` | sequence | standard |

本叶子不补 dataflow/lifecycle（无数据管道、无单实体状态机；角色是静态注册表）。JSON IR 位于 `json/`。两图均 standard 渲染通过、HTML >800KB。
