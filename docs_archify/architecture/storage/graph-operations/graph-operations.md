# 图写操作与 Purge 恢复（graph-operations）

> 本文是 `storage` 域下的叶子子系统文档。域级总览见 `../storage.md`。
> 本文展开 LightRAG 的图库管理员操作（建/改/删/合并实体与关系）、关系权重契约、提交失败闭合、
> 以及 Purge 恢复锚点机制。权威契约见 `docs/design/PurgeRecoveryContract.md`。
>
> 源码基准：`lightrag` main 分支，commit `453dce83`。utils_graph.py 约 3735 行。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 建实体/关系 | 管理员手动建图对象，先校验后写 | `utils_graph.py:2293`（`acreate_entity`）、`:2487`（`acreate_relation`） |
| 改实体/关系 | 改属性/描述，保持追踪一致 | `utils_graph.py:1550`（`aedit_entity`）、`:1850`（`aedit_relation`） |
| 删实体/关系 | 删图对象并清扫孤儿追踪行 | `utils_graph.py:539`（`adelete_by_entity`）、`:770`（`adelete_by_relation`） |
| 合并实体 | 合并节点、边、属性 | `utils_graph.py:3440`（`amerge_entities`）、`:2703`（`_merge_entities_impl`） |
| 关系权重契约 | weight≥去重真实 source_id 数 | `utils_graph.py:165`（`relation_evidence_source_ids`）、`:192`（`validate_relation_weight`）、`:220`（`apply_relation_weight_floor`） |
| 提交失败闭合 | 图 store 拒绝提交时抛错而非半写 | `utils_graph.py:278`（`_declined_commit_error`）、`:474`（`_commit_graph_or_raise`） |
| 锚点写入 | 每文档 write-ahead 锚点 | 消费方见 `docs/design/PurgeRecoveryContract.md` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `acreate_entity`/`aedit_entity` 等 | utils_graph.py | admin 写操作入口，全部在 `LightRAG._admin_write_gate` 内运行 |
| `_commit_graph_or_raise` | `utils_graph.py:474` | 提交图变更，失败即抛 |
| `_declined_commit_error` | `utils_graph.py:278` | 图 store 声明"快照被越过"时的错误 |
| `_entity_chunk_tracking_update` | `utils_graph.py:910` | chunk 追踪更新（优先于图 source_id） |
| `_merge_entities_impl` | `utils_graph.py:2703` | 合并内部：边归并、权重取 max |
| `_merge_attributes` | `utils_graph.py:3579` | 属性合并策略 |

## 3. 关键调用链

**调用链一：admin 写操作必经门**
1. REST 路由 → `acreate_*`/`aedit_*`/`adelete_by_*`/`amerge_*`。
2. 每个入口 docstring 注明"调用方在 `LightRAG._admin_write_gate` 内运行"（如 :597/:1612/:3475）。
3. 工作区 admin 锁（等待）→ `busy` 预留（kind=admin，拒 busy/scanning）→ 在逐实体 key 锁外。

**调用链二：关系权重校验（写前）**
1. `create_relation`/`edit_relation`/`insert_custom_kg` 在**首次存储变更前**校验完整关系。
2. `relation_evidence_source_ids`(:165) 去重真实 source_id（空串与 `manual_creation`/`UNKNOWN` 不算证据）。
3. `apply_relation_weight_floor`(:220) 保证 `weight ≥ 证据数`。

**调用链三：提交失败闭合**
1. `_commit_graph_or_raise`(:474) 提交；图 store 声明拒绝时 `_declined_commit_error`(:278)。
2. **Purge 失败闭合**：purge 抛 `RecoveryAnchorMissingError`（HTTP 409）、什么都不删，除非四证之一——anchors / pre_graph / journal / empty_scope。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `_admin_write_gate` | admin 写必经串行门 | LightRAG 类（lightrag.py） |
| 关系权重 | `weight ≥ 去重真实 source 数` | `apply_relation_weight_floor`(:220) |

## 5. 错误与重试语义

- **失败 loud**：图 store 拒绝提交即抛 `_declined_commit_error`，不静默半写。
- **Purge 失败闭合**：缺四证则 `RecoveryAnchorMissingError` → HTTP 409，零删除。
- **`kg_write_state` 单调**：入队时写一次，禁止回填/推断——回填会重现旧的静默跳过缺陷。
- **LLM 缓存可达性**：缓存行只能经所属 chunk 的 `llm_cache_list` 到达；引用先于行写、提交先于行。

## 6. 并发细节

- admin 写在 `_admin_write_gate` 下：先工作区 admin 锁（等待），再 `busy` 预留（kind=admin）。
- 顺序固定：admin 锁 → busy 预留，且在逐实体 key 锁**之外**。
- 门持有期间的流水线启动被推迟到 ingress 邮箱，释放后驱动。

## 7. 系统边界

**In-Scope**：`lightrag/utils_graph.py`。

**Out-of-Scope（不在本仓库源码内）**：具体图存储后端（见各叶子）；purge 的 `_purge_kg_contributions` 实现本体在管线/lightrag 侧，本文只覆盖其调用的写操作与锚点契约。

## 8. 与相邻子系统交互

- **上游**：REST 路由 `/graphs/*`、`ainsert_custom_kg`。
- **下游**：图存储后端（L10 NetworkX 声明单写者才过门；L11–L14 服务端并发不必）。
- **跨层**：chunk 追踪（`entity_chunks`/`relation_chunks`）优先于图 `source_id`。

## 9. 语言专项适配口径

按 `language-python.md`：本叶子是图存储的**操作编排层**（不直接存数据），是 admin 写面的扩展点；合并/改名的失败模型按 PurgeRecoveryContract 的 ordering invariants 执行。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| admin 图写操作矩阵与门 | `graph-operations-architecture.html` | architecture | showcase |
| admin 写门与提交时序 | `graph-operations-admin-sequence.html` | sequence | showcase |

两图均一次通过 showcase（时序图末条消息 y 微越界后上移通过）。

省略说明：不补 lifecycle——purge 失败闭合是"四证分支判定"而非单实体状态机，已写入本 MD 第 3/5 节。

JSON IR 源文件位于 `json/` 目录。
