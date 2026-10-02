# 词窗与递归字符分块（token-recursive-chunkers）

> 本文是 `chunker` 域下的叶子子系统文档。域级总览见 `../chunker.md`。本文展开 F（fixed_token）
> 与 R（recursive_character）两种**确定性、无需向量**的分块策略。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| legacy 词窗分块 | `chunking_by_token_size` 六位置参数历史签名，外部 `chunking_func` 兼容 | `token_size.py:133` |
| F 策略固定词窗 | `chunking_by_fixed_token`：同算法新契约签名 | `token_size.py:278` |
| 重叠步长 | `_window_step` 按 chunk/overlap token 计算前进量；`_make_chunk` | `token_size.py:114/96` |
| 源 span 记录 | `_source_span`/`_token_window_source_span` 记 chunk 在原文 `[start,end)` | `token_size.py:43/50` |
| R 递归字符 | `chunking_by_recursive_character`，包 LangChain `RecursiveCharacterTextSplitter` | `recursive_character.py:436` |
| 分隔符归一化 | `normalize_r_separators`、`inspect_r_separators`、`log_r_separator_normalization` | `recursive_character.py:310/232/285` |
| span 切分 | `_split_text_with_regex_spans`/`_merge_splits_with_spans` 保 chunk 溯源 | `recursive_character.py:61/175` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `chunking_by_token_size` | `token_size.py:133` | legacy 词窗分块 |
| `chunking_by_fixed_token` | `token_size.py:278` | F 策略标准签名 |
| `chunking_by_recursive_character` | `recursive_character.py:436` | R 策略 |
| `normalize_r_separators` | `recursive_character.py:310` | 分隔符级联归一化 |

## 3. 关键调用链

### 调用链一：F 词窗分块

1. `chunking_by_fixed_token(tokenizer, content, chunk_token_size, *, chunk_overlap_token_size, ...)`
   （`token_size.py:278`）按 tokenizer 编码全文。
2. `_window_step`（`token_size.py:114`）算步长，滑动窗口切分；`_make_chunk`（`:96`）产出 chunk dict，
   `_source_span`（`:43`）记录原文 `[start,end)`，供 sidecar 回填（L20 backfill）。

### 调用链二：R 递归字符分块

1. `chunking_by_recursive_character`（`recursive_character.py:436`）先 `normalize_r_separators`（`:310`）
   归一化分隔符级联。
2. `_split_text_with_regex_spans`（`:61`）按分隔符递归切，`_merge_splits_with_spans`（`:175`）
   合并到 token 预算内，每段附 span。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| chunk_token_size | 目标 chunk token 数 | 两策略入参 |
| chunk_overlap_token_size | 相邻 chunk 重叠 token | `_window_step` |
| separators | R 分隔符级联，可被 `chunking` 选项覆盖 | `normalize_r_separators` |

## 5. 错误与重试语义

- 纯本地确定性函数，无网络/重试；tokenizer 编码失败直接上抛。
- R 分隔符非法时归一化并记录（`log_r_separator_normalization`），不静默。

## 6. 并发细节

- 同步纯函数；`executor_safe=True` 时可进有界分块执行器（见 L21）。
- 无共享可变状态；tokenizer 由调用方传入。

## 7. 系统边界

**In-Scope**：`lightrag/chunker/token_size.py`、`recursive_character.py`。

**Out-of-Scope**：LangChain `RecursiveCharacterTextSplitter`（R 的底层）、tokenizer 实现不在本仓库源码内；
V/P 策略见 L23。

## 8. 与相邻子系统交互

- 上游 L21 `ChunkerBinding` 装载本模块实现；pipeline 传入 tokenizer/content/context。
- 本叶子 → 产出 chunk 列表（含 `_source_span`），经 L20 `backfill_chunk_sidecars` 补 sidecar 溯源。

## 9. 语言专项适配口径（纯 Python 框架）

- **双轨/兼容**：F 与 legacy `chunking_by_token_size` 同算法不同签名，保护外部 `chunking_func`。
- **图型侧重**：dataflow（文本→切分→带 span chunk）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| F/R 分块架构图 | `token-recursive-chunkers-architecture.html` | architecture | showcase |
| 词窗切分数据流 | `token-recursive-chunkers-dataflow.html` | dataflow | showcase |

**省略说明**：未生成 sequence/lifecycle——单函数确定性切分，无多方消息时序与状态机。
