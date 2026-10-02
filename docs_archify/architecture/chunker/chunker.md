# chunker 域（文本分块域）

> 本文是 LightRAG 架构分析中 `chunker` 域的域级总览。域内 3 个叶子子系统负责：把解析后的
> 合并文本切成带溯源的 chunk，供实体抽取与向量化。本域位于 parser（L16–L20）之后、KG 构建之前。

## 域职责

- **注册表/工厂**：F/R/V/P/C 选择器路由，实现惰性装载，第三方插件经 `lightrag.chunkers` entry point。
- **双契约兼容**：legacy 六位置参数回调保护外部 `chunking_func`；文件管线标准签名 `(tokenizer, content, chunk_token_size, **kw)`。
- **四种策略**：固定词窗（F）、递归字符（R）、语义向量（V）、标题感知段落语义（P）。
- **溯源**：每个 chunk 带原文 `_source_span`，P 直接挂 sidecar 行；F/R/V 经 L20 backfill 补 sidecar。

## 叶子索引

| 叶子 | 职责 | 关键源码 | 架构图 | 第二张图 |
|---|---|---|---|---|
| [L21 chunker-registry](chunker-registry/chunker-registry.md) | 注册表/工厂/双契约/插件发现 | `registry.py`、`plugins.py`、`__init__.py` | [architecture](chunker-registry/chunker-registry-architecture.html) | [sequence](chunker-registry/chunker-registry-sequence.html) |
| [L22 token-recursive-chunkers](token-recursive-chunkers/token-recursive-chunkers.md) | F 固定词窗 + R 递归字符 | `token_size.py`、`recursive_character.py` | [architecture](token-recursive-chunkers/token-recursive-chunkers-architecture.html) | [dataflow](token-recursive-chunkers/token-recursive-chunkers-dataflow.html) |
| [L23 semantic-chunkers](semantic-chunkers/semantic-chunkers.md) | V 语义向量 + P 段落语义 | `semantic_vector.py`、`paragraph_semantic.py` | [architecture](semantic-chunkers/semantic-chunkers-architecture.html) | [dataflow](semantic-chunkers/semantic-chunkers-dataflow.html) |

## 域级机制细节

### 选择器与契约

- `process_options.chunking` ∈ {F,R,V,P,C}；未指定走 legacy 六参回调。
- C 是双契约桥：用 legacy 回调签名，同时保留文件管线的选择器快照与可观测性。
- `ChunkingContext`（doc_id/sidecar_location/parse_engine…）经 `accepts_chunking_context` 显式 opt-in 传入。

### 分块管道

```
合并文本(blocks.jsonl) → 注册表选策略 → F/R/V/P → chunk(带 span) → 实体抽取/向量化
                                              ↓
                              P 直挂 sidecar 行；F/R/V 经 backfill 补溯源
```

## 域级图

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| 分块域架构图 | `chunker-domain-architecture.html` | architecture | showcase |
| 文本分块管道数据流 | `chunker-domain-dataflow.html` | dataflow | showcase |
| 双契约调用时序 | `chunker-domain-sequence.html` | sequence | showcase |

JSON IR 位于 `json/` 目录。
