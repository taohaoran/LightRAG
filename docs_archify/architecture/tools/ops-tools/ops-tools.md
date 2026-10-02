# 运维与评估工具集（ops-tools）

> 本文是 `tools` 域下唯一的叶子子系统文档。域级总览见 `../tools.md`。本文覆盖 `lightrag/tools/`
> 的 9 个运维修复工具与 `lightrag/evaluation/` 的 2 个评估脚本。它们是**离线诊断/修复 CLI**，
> 不在摄入/查询热路径上。

## 1. 功能清单

| 工具 | 说明 | 源码路径 |
|---|---|---|
| check_initialization | `check_lightrag_setup` 校验 LightRAG 实例存储连通性 | `tools/check_initialization.py:30` |
| chunk_tracking_repair | chunk↔图节点追踪修复计划（`_DiskPlan`/`ChunkTrackingRepairReport`） | `tools/chunk_tracking_repair.py:66/278` |
| clean_llm_query_cache | LLM 查询缓存清理（`CleanupTool`/`CleanupStats`） | `tools/clean_llm_query_cache.py:122/81` |
| download_cache | 下载 spaCy 模型与 tiktoken 缓存 | `tools/download_cache.py:32/110` |
| hash_password | 口令哈希命令行 | `tools/hash_password.py:23` |
| kg_integrity_repair | KG 完整性审计：孤儿文档、缺锚点（`audit_kg_integrity`） | `tools/kg_integrity_repair.py:70/219` |
| migrate_graph_storage | 图存储迁移：边规范化、源节点校验 | `tools/migrate_graph_storage.py:83/157` |
| migrate_llm_cache | LLM 缓存键迁移（`MigrationTool`/`MigrationStats`） | `tools/migrate_llm_cache.py:129/103` |
| rebuild_vdb | 向量库重建：先拒清空、按 chunk 重嵌 | `tools/rebuild_vdb.py:145/152` |
| source_conflict_repair | 源键冲突修复：收集冲突、逐个修复加锁 | `tools/source_conflict_repair.py:301/326` |
| eval_rag_quality | RAG 质量评估（RAGAS） | `evaluation/eval_rag_quality.py` |
| offline_retrieval_check | 离线检索正确性校验 | `evaluation/offline_retrieval_check.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `audit_kg_integrity` | `kg_integrity_repair.py:70` | 审计 KG 与 chunk/文档一致性 |
| `ChunkTrackingRepairPlan` | `chunk_tracking_repair.py:306` | chunk 追踪修复计划 |
| `MigrationTool` | `migrate_llm_cache.py:129` | LLM 缓存迁移器 |
| `repair_one_conflict` | `source_conflict_repair.py:326` | 单个源键冲突修复 |

## 3. 关键调用链

### 调用链一：KG 完整性审计

1. `audit_kg_integrity(rag)`（`kg_integrity_repair.py:70`）`_map_chunks_to_docs`（`:55`）把 chunk 映射回文档。
2. `_find_anchorless_docs`（`:219`）找出在图/KV 中无归属锚点的文档，产出报告；与 PurgeRecoveryContract
   的"锚点缺失即 fail-closed"口径一致。

### 调用链二：向量库重建

1. `rebuild_vdb` 先 `_ensure_vector_rebuild_supported`（`rebuild_vdb.py:135`），
   `clear_vector_space_refusal`（`:152`）拒绝误清空。
2. `_drop_vdb`（`:145`）后按 chunk 重新嵌入写回。

### 调用链三：源键冲突修复

1. `collect_source_conflicts`（`source_conflict_repair.py:301`）扫描冲突；
2. `repair_one_conflict`（`:326`）经 `source_conflict_repair_lock`（`:240`）加锁逐个修复。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| workspace | 修复目标工作区 | 各工具 CLI 参数 |
| --apply / --dry-run | 多数修复工具默认演练后执行 | 各 README_*.md |

## 5. 错误与重试语义

- 修复类工具默认**先报告后修改**（dry-run），符合 PurgeRecoveryContract 的 fail-closed：锚点缺失拒绝删除。
- 重建/迁移失败上抛，不静默吞错；冲突修复持锁避免并发写。

## 6. 并发细节

- 离线工具，单次同步/异步运行；冲突修复经 `source_conflict_repair_lock` 串行化。
- 不常驻事件循环；由人手动触发。

## 7. 系统边界

**In-Scope**：`lightrag/tools/`、`lightrag/evaluation/`。

**Out-of-Scope**：被修复的存储后端（KV/vector/graph/doc-status）见 storage 域；
评估依赖 RAGAS 等第三方不在本仓库源码内。

## 8. 与相邻子系统交互

- 本叶子 → 下游：直接读写四类存储（K/V/Graph/DocStatus）与 LLM 缓存。
- 上游：运维人员经 CLI；不被 pipeline/查询调用。

## 9. 语言专项适配口径（纯 Python 框架）

- **CLI 脚本族**：每个工具一个可执行模块 + 独立 README；argparse 入口。
- **图型侧重**：architecture（工具清单→四类存储）+ workflow（修复演练→应用流程）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 运维工具架构图 | `ops-tools-architecture.html` | architecture | standard |
| 修复工具数据流 | `ops-tools-dataflow.html` | dataflow | showcase |

> 架构图降 **standard**：9 工具映射到 KV/向量/图三类存储时出现边穿节点（`clean-flow/edge-through-node`），
> 经合并节点仍未达 showcase，按规则降档并记录（检查名：edge-through-node / endpoint-side-direction）。
> workflow 图因当前 archify schema 要求 `lanes` 泳道（与出图指南模板不符）改用 dataflow 表达修复管道。

**省略说明**：未生成 sequence/workflow/lifecycle——工具是独立 CLI；workflow/lifecycle 的 lanes schema
与本版 archify 不兼容，修复管道语义由 dataflow 承担。
