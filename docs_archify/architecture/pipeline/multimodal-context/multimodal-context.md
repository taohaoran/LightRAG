# 多模态上下文增强（multimodal-context）

> 本文是 `pipeline` 域下的叶子子系统文档。域级总览见 `../pipeline.md`。
> 本文展开多模态实体增强的上下文装配、多模态提示词与分块 schema 归一化；
> 多模态分析 worker 的调度见 `../pipeline-mixin/pipeline-mixin.md`。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 文本原子化 | 把含表格/图/公式标记的文本切成原子段 | `lightrag/multimodal_context.py:105` `_atomize` |
| 目标 span 定位 | 在分块中定位 drawing/table/equation 的 span | `multimodal_context.py:162` `find_target_span` |
| 表格修剪 | 按 token 预算前/后修剪表格行 | `multimodal_context.py:297/359` `_row_trim_table_*` |
| 文本首尾装配 | 围绕目标元素拼上下文首尾 | `multimodal_context.py:505/629` `_build_leading/trailing` |
| 内容内修剪 | 超预算时修剪中间内容 | `multimodal_context.py:851` `_trim_content_inner` |
| 分块分隔符 | 加载块间分隔符缓存 | `multimodal_context.py:760` `load_chunk_separators` |
| 块内容加载 | 按 blockid 加载 sidecar 块内容 | `multimodal_context.py:769` `load_content_rows_by_blockid` |
| 多模态提示词 | 图片类型回退与表格格式标签 | `lightrag/prompt_multimodal.py:49/65` |
| 分块 heading 归一化 | heading 面包屑清理与截断 | `lightrag/chunk_schema.py:41/148` |
| sidecar 分块归一化 | block/drawing/table/equation 类型归一 | `chunk_schema.py:201` `normalize_chunk_sidecar` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `find_target_span` | `multimodal_context.py:162` | 定位目标多模态元素在块中的位置 |
| `_atomize` | `multimodal_context.py:105` | 文本切分为原子段 |
| `normalize_chunk_sidecar` | `chunk_schema.py:201` | sidecar 块归一化 |
| `format_parent_headings` | `chunk_schema.py:148` | heading 面包屑格式化 |
| `IMAGE_TYPE_FALLBACK` | `prompt_multimodal.py:49` | 图片类型未知时回退 "Other" |

## 3. 关键调用链

**调用链 A：多模态上下文装配（`multimodal_context.py`）**
1. `_atomize` 把文本切成原子段，`find_target_span` 定位 drawing/table/equation。
2. `_build_leading`（`multimodal_context.py:505`）拼前文，`_build_trailing`（`multimodal_context.py:629`）拼后文。
3. 超 token 预算时 `_row_trim_table_*` 修剪表格行，`_trim_content_inner`（`multimodal_context.py:851`）修剪中间。

**调用链 B：sidecar 分块归一（`chunk_schema.py:201`）**
1. `normalize_chunk_sidecar` 按 `_SIDECAR_TYPES`（block/drawing/table/equation）归一。
2. `normalize_chunk_heading` 清理 heading 文本，`format_parent_headings` 拼面包屑。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 块分隔符 | `load_chunk_separators` 加载并缓存 | `multimodal_context.py:760` |
| heading 长度上限 | `_truncate_heading_level` 截断 | `chunk_schema.py:125` |

## 5. 错误与重试语义

- **表格超预算**：优先按行修剪而非整体丢弃；行修剪失败回退 JSON/HTML 表格兜底（`_char_fallback_*`）。
- **图片类型未知**：回退 `IMAGE_TYPE_FALLBACK="Other"`，不报错。

## 6. 并发细节

- `_cached_surrounding_chunk_separators` 缓存分隔符，避免重复加载。
- 本模块以纯函数为主，无自身并发原语。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/multimodal_context.py`、`prompt_multimodal.py`、`chunk_schema.py`

**Out-of-Scope（不在本仓库源码内）**
- VLM 实际推理——外部 VLM 服务
- sidecar 解析产物生成——见 sidecar 域

## 8. 与相邻子系统交互

- 上游：`_analyze_worker`（pipeline-mixin）调用本模块装配上下文喂 VLM
- 下游：归一化分块喂实体提取（operate-retrieval）

## 9. 语言专项适配口径（纯 Python 框架）

- **纯函数处理缝**：文本修剪/归一化全部是无状态纯函数，便于单测。
- **sidecar schema 驱动**：`_SIDECAR_TYPES` 注册表驱动块类型归一化。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 多模态上下文模块架构 | `multimodal-context-architecture.html` | architecture | standard（垂直边标签压组件；已加 labelDy） |
| 上下文装配数据流 | `multimodal-context-dataflow.html` | dataflow | standard（"超预算"标签压节点；已加 labelDy） |

本叶子不补 sequence/lifecycle（纯函数装配，无多方消息交互与状态机）。JSON IR 位于 `json/`。两图均 standard 渲染通过、HTML >800KB。
