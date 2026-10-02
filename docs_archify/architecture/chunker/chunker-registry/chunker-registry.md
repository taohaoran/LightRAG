# 分块注册表（chunker-registry）

> 本文是 `chunker` 域下的叶子子系统文档。域级总览见 `../chunker.md`。本文展开分块策略的
> **注册表 / 工厂 / 双契约路由**，不重复展开四种具体策略算法（见 L22、L23）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 双契约并存 | legacy 六位置参数回调（外部 `chunking_func`）vs 文件管线标准签名 `(tokenizer, content, chunk_token_size, **kw)` | `chunker/__init__.py` 模块 docstring |
| 策略选择器 | F（fixed_token）/R（recursive_character）/V（semantic_vector）/P（paragraph_semantic）/C（custom 回调桥） | `chunker/__init__.py` |
| ChunkingContext | 文档级元数据（doc_id/file_path/sidecar_location/parse_format/parse_engine/process_options）显式传入 | `registry.py:39` |
| 上下文标记 | `accepts_chunking_context` 装饰器打标，`callback_supports_context` 读取 | `registry.py:55/69` |
| 单次调用 | `invoke_chunker` 探测一次即调用，禁止失败重试二次执行 | `registry.py:79` |
| ChunkerSpec | frozen dataclass：name/惰性 impl 引用/version/description/executor_safe/accepts_context | `registry.py:99` |
| 注册与解析 | `register_chunker`（不导入实现）、`resolve_chunker`（按名解析为 ChunkerBinding）、`ChunkerBinding.__call__` | `registry.py:149/242/215` |
| 第三方插件发现 | entry point 组 `lightrag.chunkers`，零参数注册函数；失败日志并跳过 | `plugins.py` |
| 惰性导出 | `__getattr__` 按需 import 实现模块，顶层不加载 | `chunker/__init__.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `ChunkingContext` | `registry.py:39` | 文档级上下文，显式 opt-in 传入 |
| `ChunkerSpec` | `registry.py:99` | 策略元数据 + 惰性 impl 引用 |
| `ChunkerBinding` | `registry.py:204` | spec + 已装载 impl，统一 `__call__` |
| `register_chunker` | `registry.py:149` | 注册（名校验/保留名/冲突报告） |
| `resolve_chunker` | `registry.py:242` | CUSTOM_CHUNKER 字符串→binding |
| `invoke_chunker` | `registry.py:79` | legacy/context-aware 统一单次调用 |

## 3. 关键调用链

### 调用链一：CUSTOM_CHUNKER 选择器解析

1. 构造期 `load_and_resolve_chunker(name)`（`plugins.py`）先 `_discover()` 扫描 entry point，
   每个 ep 加载零参数注册函数并调 `register_chunker(ChunkerSpec(...))`（`plugins.py` `_discover`）。
2. `resolve_chunker(name)`（`registry.py:242`）校验名正则 `[a-z0-9][a-z0-9_-]{0,63}`、拒绝 import 路径、
   报告重复名，再按 spec.impl 惰性装载实现，包成 `ChunkerBinding`（`registry.py:204`）。
3. 分片失败时 `_plugin_registration`（`registry.py:130`）回滚 `_REGISTRY` 到上一快照。

### 调用链二：单次分块调用

1. pipeline 对每文档调 `invoke_chunker(callback, *args, context=ctx)`（`registry.py:79`）。
2. 若是 `ChunkerBinding` 或被 `accepts_chunking_context` 标记，则带 `context=` 调用；否则按 legacy 六参调用
   （`registry.py:91-95`）。**TypeError 视为真实失败，绝不二次重试**。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| CUSTOM_CHUNKER | 注册名；未设则走 legacy 回调 | `registry.py:242` |
| 选择器 F/R/V/P/C | 由 `process_options.chunking` 指定 | `chunker/__init__.py` |
| executor_safe | True 的同步实现进有界分块执行器 | `registry.py:114` |

## 5. 错误与重试语义

- 插件加载失败：`_failures` 收集，日志 `[chunker-plugins] skipped` 并跳过，不阻断启动。
- `invoke_chunker` 故意不做签名探测重试——callback 内 TypeError 是真实失败。
- 重复名注册：报告两个 origin，启动报错。

## 6. 并发细节

- `executor_safe=True` 的实现可进有界分块执行器；默认 legacy 契约在事件循环上同步执行。
- 注册表 `_REGISTRY` 是模块级 dict，发现期写、运行期只读。

## 7. 系统边界

**In-Scope**：`lightrag/chunker/__init__.py`、`registry.py`、`plugins.py`。

**Out-of-Scope**：具体分块算法见 L22/L23；LangChain `RecursiveCharacterTextSplitter`/`SemanticChunker`、
EmbeddingFunc 不在本仓库源码内；pipeline 选择器来源见 L04。

## 8. 与相邻子系统交互

- 上游 pipeline（L04）→ `invoke_chunker` 传入 tokenizer/content/context。
- 本叶子 → 下游 L22（F/R）、L23（V/P）具体实现；`ChunkerBinding` 惰性装载 impl。

## 9. 语言专项适配口径（纯 Python 框架）

- **注册表与工厂**：字符串选择器→ChunkerBinding，与 parser 域 `STORAGE_IMPLEMENTATIONS` 同构；
  实现惰性 import（`__getattr__` / spec.impl），顶层 import 不加载重依赖。
- **配置驱动静态分派**：F/R/V/P/C 选择器决定运行时签名与实现。
- **图型侧重**：architecture（注册表 + 双契约路由）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 分块注册表架构图 | `chunker-registry-architecture.html` | architecture | showcase |
| 策略选择调用链时序 | `chunker-registry-sequence.html` | sequence | showcase |

**省略说明**：未生成 dataflow/lifecycle——注册表是静态路由而非数据管道；无实体状态机。
