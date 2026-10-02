# 语义向量化分块（semantic-chunkers）

> 本文是 `chunker` 域下的叶子子系统文档。域级总览见 `../chunker.md`。本文展开 V（semantic_vector）
> 与 P（paragraph_semantic）两种**语义感知**分块策略。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| V 语义向量分块 | `chunking_by_semantic_vector`（async），句向量相似度找断点 | `semantic_vector.py:464` |
| 句切分与 span | `_sentence_spans`、`_semantic_groups_with_spans`、`_trim_span` | `semantic_vector.py:361/389/379` |
| 超界重切 | 超过 `chunk_token_size` 的片段回退 `chunking_by_recursive_character` | `semantic_vector.py:464` docstring |
| P 段落语义分块 | `chunking_by_paragraph_semantic`，标题感知，消费 docx `.blocks.jsonl` sidecar | `paragraph_semantic.py` |
| sidecar 读入 | `_load_blocks_from_jsonl` 按序读 `type=="content"` 行 | `paragraph_semantic.py:193` |
| 表格侧车 | `_tables_sidecar_path`/表头映射，`<base>.blocks.jsonl→<base>.tables.json` | `paragraph_semantic.py:306/320` |
| R 兜底 | sidecar 缺失/不可读时回退 R 策略 | `paragraph_semantic.py:23` docstring |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `chunking_by_semantic_vector` | `semantic_vector.py:464` | V 策略（async） |
| `chunking_by_paragraph_semantic` | `paragraph_semantic.py` | P 策略 |
| `_load_blocks_from_jsonl` | `paragraph_semantic.py:193` | 读 blocks sidecar |
| EmbeddingFunc | 外部传入 | V 策略句向量来源 |

## 3. 关键调用链

### 调用链一：V 语义向量分块

1. `chunking_by_semantic_vector(tokenizer, content, chunk_token_size, *, embedding_func, ...)`
   （`semantic_vector.py:464`）按 `sentence_split_regex` 切句，`_sentence_spans`（`:361`）定位句 span。
2. `_semantic_groups_with_spans`（`:389`）用 `embedding_func` 算句向量，按 percentile 断点阈值分组。
3. 超过 `chunk_token_size` 的组用 `chunking_by_recursive_character` 重切后再产出 chunk。

### 调用链二：P 段落语义分块

1. `chunking_by_paragraph_semantic` 经 `_load_blocks_from_jsonl`（`paragraph_semantic.py:193`）读 sidecar
   块；`_tables_sidecar_path`（`:306`）派生 `<base>.tables.json` 补重复表头。
2. 按标题层级与段落语义归并到目标 token；sidecar 缺失/不可读时回退 R（`:23` docstring）。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| breakpoint_threshold_type | percentile（默认） | `semantic_vector.py:464` |
| buffer_size / number_of_chunks / min_chunk_size | 语义分组调节 | `semantic_vector.py:464` |
| chunk_token_size | 硬上界，超限回退 R | 两策略入参 |

## 5. 错误与重试语义

- V：embedding_func 失败上抛；超界片段回退 R 而非丢弃。
- P：sidecar 缺失/坏→降级 R，不静默产出畸形 slice（`paragraph_semantic.py:416`）。

## 6. 并发细节

- V 是 async，`embedding_max_async`/`embedding_batch_num` 控制嵌入并发。
- P 同步读本地 sidecar；无外部锁。

## 7. 系统边界

**In-Scope**：`lightrag/chunker/semantic_vector.py`、`paragraph_semantic.py`。

**Out-of-Scope**：LangChain `SemanticChunker`、EmbeddingFunc 后端（OpenAI/Ollama 等）不在本仓库源码内；
sidecar 产出方见 L20。

## 8. 与相邻子系统交互

- 上游 L21 装载；P 消费 L20 产出的 `.blocks.jsonl`/`.tables.json`。
- 本叶子 → 下游：chunk 列表；V 回退依赖 L22 R 策略。

## 9. 语言专项适配口径（纯 Python 框架）

- **隐藏状态机/循环**：V 的句分组与超界重切是处理循环，已用 dataflow 表达。
- **图型侧重**：dataflow（文本→句切→嵌入分组→chunk）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| V/P 语义分块架构图 | `semantic-chunkers-architecture.html` | architecture | showcase |
| 向量分块数据流 | `semantic-chunkers-dataflow.html` | dataflow | showcase |

**省略说明**：未生成 sequence/lifecycle——单管线处理，无多方消息时序与独立状态机。
