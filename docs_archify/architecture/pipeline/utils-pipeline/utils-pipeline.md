# 管线工具与文档身份（utils-pipeline）

> 本文是 `pipeline` 域下的叶子子系统文档。域级总览见 `../pipeline.md`。
> 本文展开 `utils_pipeline.py` 中的纯函数工具：doc-status 字段访问、文档身份（source_key/content_hash）、解析产物路径、载荷归一化；
> 它们被 pipeline-mixin 与 storage 域共享，不持有状态。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| doc-status 字段访问 | 统一读 doc_status 记录字段，容忍后端差异 | `lightrag/utils_pipeline.py:273` `doc_status_field` |
| 文档身份 | source_key / content_hash 计算与去重查询 | `utils_pipeline.py:954/969/979` `normalize_merged_text_for_hash`/`compute_text_content_hash`/`compute_file_content_hash` |
| 重复文档查询 | 按 content_hash 找已存在/重复文档 | `utils_pipeline.py:1091/1116` |
| 解析产物路径 | `parsed_dir()` / `parsed_artifact_dir_for` / `sidecar_uri_for` | `utils_pipeline.py:1187/1192/1239` |
| 文件路径解析 | `resolve_doc_file_path` / `normalize_document_file_path` | `utils_pipeline.py:226/248` |
| 载荷归一化 | `make_lightrag_doc_content` / `strip_lightrag_doc_prefix` | `utils_pipeline.py:1142/1155` |
| kg_write_state/kg_purge 元数据 | 读 doc_status 中的 purge 锚点与写状态 | `utils_pipeline.py:476/493` |
| 元数据携带键 | `_DOC_STATUS_METADATA_CARRY_OVER_KEYS` 等（purge 契约） | `utils_pipeline.py:543` |
| 严格存储能力 | `enforce_strict_storage_capabilities` 判定后端读能力 | `utils_pipeline.py:872` |
| 句切正则安全覆盖 | `apply_trusted_sentence_split_regex`（GHSA 修复） | `utils_pipeline.py:70` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `doc_status_field` | `utils_pipeline.py:273` | 后端无关的 doc_status 字段读取 |
| `compute_text_content_hash` | `utils_pipeline.py:969` | 文本内容 MD5 去重键 |
| `resolve_doc_file_path` | `utils_pipeline.py:226` | 从 status/content 解析文件路径 |
| `make_lightrag_doc_content` | `utils_pipeline.py:1142` | 归一化为 LightRAG 文档载荷 |
| `enforce_strict_storage_capabilities` | `utils_pipeline.py:872` | strict 读能力判定 |

## 3. 关键调用链

**调用链 A：文档去重（`utils_pipeline.py:1091` `get_existing_doc_by_content_hash`）**
1. `normalize_merged_text_for_hash`（`utils_pipeline.py:954`）归一化合并文本。
2. `compute_text_content_hash`（`utils_pipeline.py:969`）算 MD5。
3. `source_candidate_set_lock`（`utils_pipeline.py:1044`）按 source_key 加锁，`get_duplicate_doc_by_content_hash`（`utils_pipeline.py:1116`）查重。

**调用链 B：purge 元数据读写（`utils_pipeline.py:476`）**
- `doc_status_kg_write_state` 读 `kg_write_state`（只写一次、单调）；`doc_status_kg_purge_journal` 读 purge 日志。二者必须同时在 `_DOC_STATUS_METADATA_CARRY_OVER_KEYS` 与 `_DOC_STATUS_METADATA_DIRECTIVE_KEYS` 中（purge 恢复契约）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `INPUT_DIR` / input_dir_path | 输入文件目录 | `utils_pipeline.py:1183/1012` |
| 句切正则 | 只从 addon_params 实时读，不信任快照（GHSA-32jh） | `utils_pipeline.py:70` |

## 5. 错误与重试语义

- **句切正则安全**：`semantic_vector.sentence_split_regex` 从快照读会被丢弃并 WARNING，改为实时读 addon_params，防注入。
- **strict 读**：`enforce_strict_storage_capabilities(require=True)` 在后端读能力不足时抛错，不静默降级。

## 6. 并发细节

- `source_candidate_set_lock` 按 canonical_source_key 加命名锁，防止并发入队对同一 source 产生竞争。
- 本模块全部是纯函数，无自身线程/队列。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/utils_pipeline.py` 管线共享纯函数

**Out-of-Scope（不在本仓库源码内）**
- doc_status 存储后端——见 storage 域
- 解析器——见 parser 域

## 8. 与相邻子系统交互

- 上游：pipeline-mixin 入队/处理、storage 域
- 下游：被 pipeline-mixin 与 purge 恢复逻辑（utils_graph）调用

## 9. 语言专项适配口径（纯 Python 框架）

- **纯函数工具缝**：无状态、可测试，集中处理后端差异（doc_status 字段访问容忍不同存储实现）。
- **安全契约集中点**：句切正则、purge 元数据键等安全敏感逻辑集中在此模块。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 管线工具分组架构 | `utils-pipeline-architecture.html` | architecture | standard（一次渲染通过） |
| 文档身份去重数据流 | `utils-pipeline-dataflow.html` | dataflow | standard（"加锁"标签压节点；已加 labelDy） |

本叶子不补 sequence/lifecycle（纯函数工具，无多方交互与状态机）。JSON IR 位于 `json/`。两图均 standard 渲染通过、HTML >799KB。
