# tools 域（运维与评估域）

> 本文是 LightRAG 架构分析中 `tools` 域的域级总览。域内仅 1 个叶子子系统（L31 ops-tools），
> 覆盖 `lightrag/tools/` 的 9 个离线运维/修复工具与 `lightrag/evaluation/` 的 2 个评估脚本。
> 它们**不在摄入/查询热路径上**，由运维人员手动经 CLI 触发。

## 域职责

- **健康检查**：`check_initialization` 校验存储连通。
- **修复/重建**：KG 完整性审计、chunk 追踪修复、向量库重建、图/LLM 缓存迁移、源键冲突修复。
- **缓存清理/下载**：LLM 查询缓存清理、spaCy/tiktoken 缓存下载、口令哈希。
- **质量评估**：RAGAS RAG 质量评估、离线检索正确性校验。

## 叶子索引

| 叶子 | 职责 | 关键源码 | 图 |
|---|---|---|---|
| [L31 ops-tools](ops-tools/ops-tools.md) | 9 运维工具 + 2 评估脚本 | `tools/*.py`、`evaluation/*.py` | [architecture](ops-tools/ops-tools-architecture.html)（standard）· [dataflow](ops-tools/ops-tools-dataflow.html)（showcase） |

## 域级机制细节

- **fail-closed 修复**：修复类工具先审计/dry-run 后应用，锚点缺失拒绝删除（与 PurgeRecoveryContract 一致）。
- **四类存储直连**：工具直接读写 KV / vector / graph / doc-status，不经 pipeline。
- **每工具一 README**：`tools/README_*.md` 记录用法与安全边界。

## 域级图

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| 工具分组架构图 | `tools-domain-architecture.html` | architecture | showcase |
| 运维工具架构图（叶子） | `ops-tools/ops-tools-architecture.html` | architecture | standard |
| 修复工具数据流（叶子） | `ops-tools/ops-tools-dataflow.html` | dataflow | showcase |

JSON IR 位于 `json/` 与叶子 `json/` 目录。
