# 解析引擎（parser-engine）

> 本文是 `parser` 域下的叶子子系统文档。域级总览见 `../parser.md`。本文只展开
> **解析引擎的路由、注册、分发与调试入口**，即"一份文件该交给哪个引擎、引擎名如何
> 持久化还原、注册表如何惰性装载实现"；各引擎真正的解析算法分别见相邻叶子
> `docx-parser`（L17）、`external-parsers`（L18）、`markdown-parser`（L19），
> 解析产物的落盘格式见 `sidecar`（L20）。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 统一解析契约 | `BaseParser` 抽象基类，每个引擎实现 `async parse(ctx) -> ParseResult`；`ParseContext` 封装 rag/doc_id/文件句柄与取消事件 | `lightrag/parser/base.py:155`（`BaseParser`）、`:44`（`ParseContext`） |
| 解析结果结构 | `ParseResult.to_dict()` 只发射有语义字段，保持与重构前 `parse_native/parse_mineru/parse_docling` 返回形状逐字节兼容 | `lightrag/parser/base.py:114` |
| 引擎注册表 | 模块级 `ParserSpec` 字面量表，**装此模块不 import 任何解析实现**；能力查询（后缀/端点/可用引擎）零重依赖 | `lightrag/parser/registry.py:142`（`ParserSpec`）、`:266`（`_REGISTRY`） |
| 惰性装载实现 | `get_parser(engine)` 用 `importlib` 按 `"module:Class"` 串惰性 import 并缓存实例； heavy 包（httpx 等）只在真解析时拉入 | `lightrag/parser/registry.py:340` |
| 引擎可用性门禁 | `endpoint_configured()` 闭包读环境变量（MinerU 官方/本地双模式、Docling endpoint）；`available_engine_suffixes` 是上传白名单与扫描的唯一来源 | `lightrag/parser/registry.py:45`、`:368` |
| 部署期后缀扩展 | `extra_suffixes_env` 描述符每次访问实时合并 `MINERU_ADDITIONAL_SUFFIXES` 等，畸形 token 启动期上报 | `lightrag/parser/registry.py:102`（`_EnvExtendedSuffixes`）、`:395` |
| 文件名提示解析 | 从文件名 `[engine(options)]` 括号语法解析引擎、process_options、分块参数与引擎参数 | `lightrag/parser/routing.py:797`（`split_engine_and_options`）、`:1053`（`filename_parser_directives`） |
| 解析规则路由 | `LIGHTRAG_PARSER` 规则串按优先级（文件名提示 > 规则 > 后缀默认 > legacy）解析出最终引擎 | `lightrag/parser/routing.py:1420`（`resolve_parser_directives`）、`:1512`（`resolve_file_parser_engine`） |
| 引擎名编解码 | `encode/decode_parse_engine` 把引擎名+参数编码进持久化 `parse_engine` 字段，重解析免疫后续 env 改动 | `lightrag/parser/routing.py:95`、`:114` |
| 存储态引擎还原 | `resolve_stored_document_parser_engine` 按 `parse_format` 分流 reuse/passthrough/真实引擎 | `lightrag/parser/routing.py:1547` |
| native 内部分发 | `NativeParser` 是注册表里唯一 native 实现，按后缀再分发到 docx / markdown 子解析器 | `lightrag/parser/native_dispatch.py:26` |
| 第三方插件发现 | `lightrag.parsers` entry point 组，每个插件是零参可调用、自行 `register_parser`；失败插件记日志跳过不拖垮启动 | `lightrag/parser/plugins.py:37` |
| 原生解析模板方法 | `NativeParserBase.parse()` 模板：validate→worker 线程 `extract`→`build_ir`→告警收口；`wants_llm_bridge` 决定是否建 LLM 桥 | `lightrag/parser/native_base.py:59`、`:226` |
| 同步 LLM 桥 | `SyncLLMBridge` 在 loop 线程构造、worker 线程调用，`run_coroutine_threadsafe` 跨事件循环边界，短轮询响应取消 | `lightrag/parser/llm_bridge.py:32` |
| 取消异常族 | `ParseCancelled/ParsePipelineCancelled/ParseShutdown`；pipeline 捕获 `ParsePipelineCancelled` 把文档标为取消而非失败 | `lightrag/parser/exceptions.py:15` |
| 空格式处理器 | `ReuseParser`（lightrag 行，复用已有 sidecar）与 `PassthroughParser`（raw 行，逐字透传），均 `parse_stage_skipped=True` | `lightrag/parser/noop.py:24`、`:54` |
| 调试 CLI | `python -m lightrag.parser.cli <file> --engine <e>` 走注册表分发，sidecar 平铺输出供检查；经 `load_third_party_parsers` 后构造 engine 选项 | `lightrag/parser/cli.py`、`:54`（`_build_parser`） |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BaseParser` (ABC) | `base.py:155` | 引擎抽象契约；子类设 `engine_name` 并实现 `parse`；能力元数据刻意不放这里而放注册表 |
| `ParseContext` | `base.py:44` | 解析入参；`resolve()` 派生 `(source_path, document_name, parsed_dir)`，`source_path()` 惰性 import pipeline 层避免循环依赖 |
| `ParseResult` | `base.py:114` | 结构化输出；`smartheading_llm_cache_ids` 是一等字段，pipeline 镜像进 `doc_status.metadata` 以供 purge 时清理解析期缓存 |
| `ParserSpec` (frozen dataclass) | `registry.py:142` | 轻量元数据：`impl` 串、`suffixes`、`queue_group`、`concurrency`、端点闭包、`extra_suffixes_env` |
| `get_parser` | `registry.py:340` | 惰性工厂；缓存键 `(engine, impl)`，换实现不会命中陈旧实例 |
| `parser_specs_snapshot` | `registry.py:326` | pipeline 批次开始时拍快照贯穿队列/路由/worker，防止批次中途 `register_parser` 改引擎集 |
| `resolve_parser_directives` | `routing.py:1420` | 解析路由总入口，产出 `ParserDirectives(engine, process_options, chunk_params, engine_params)` |
| `ParserDirectives` | `routing.py:1401` | 全量解析结果；`process_options` 纯选择串，参数分栏存放 |
| `ProcessOptions` | `routing.py:149` | process_options 串的解码视图；`chunking_explicit` 区分"显式选策略"与"用默认" |
| `NativeParserBase` | `native_base.py:59` | 原生引擎模板方法基类；抽象 `extract`/`build_ir`，钩子 `validate_source`/`wants_llm_bridge`/`surface_warnings` |
| `NativeParser` | `native_dispatch.py:26` | native 引擎后缀内部分发器（docx vs md/textpack） |
| `SyncLLMBridge` | `llm_bridge.py:32` | 同步 worker 线程 → 异步 loop 的 LLM 调用桥；loop 线程内调用直接报错防死锁 |
| `ParseCancelled` 族 | `exceptions.py:15` | 阻塞解析等待的取消信号；`first_cancellation` 按事件优先级取首个已置位事件 |

## 3. 关键调用链

### 调用链一：批次内一份文件的引擎解析（路由侧）

1. pipeline 在入队/处理时调用 `resolve_file_parser_engine(file_path, parser_rules=...)`
   （`routing.py:1512`），它薄封装 `resolve_parser_directives`（`routing.py:1420`）。
2. `resolve_parser_directives` 先 `_validate_filename_hint_for_resolution`（`routing.py:930`）
   校验文件名提示；再取 `filename_parser_directives`（`routing.py:1053`）拆出提示引擎/选项/参数。
3. 若提示引擎 `_engine_is_usable`（`routing.py:843`）不通过（后缀不支持或端点未配），
   置空提示引擎但保留提示选项，转入规则匹配 `_matching_rule_directives`（`routing.py:1352`）。
4. 终选 `engine = hinted_engine or rule_engine or default_engine or legacy`
   （`routing.py:1477`）；分块参数按选择符分栏叠加（规则先、文件名后赢），引擎参数只保留
   归属胜出引擎的部分，最后 `seed_smart_heading_param`（`routing.py:1248`）以最低优先级播种全局默认。

### 调用链二：注册表惰性装载与解析分发（行为侧）

1. pipeline 经 `get_parser(engine)`（`registry.py:340`）取引擎实例：查 `_INSTANCE_CACHE`，
   未命中则 `spec.impl.partition(":")` 拆出模块与类名，`importlib.import_module` 惰性加载。
2. 拿到的若是 `native`，实例是 `NativeParser`（`native_dispatch.py:26`），其 `parse(ctx)`
   按 `Path(ctx.file_path).suffix.lower()`：`.md`/`.textpack` → `NativeMarkdownParser`，
   否则 → `NativeDocxParser`（懒加载各自模块）。
3. 真实引擎的 `parse` 返回 `ParseResult`，pipeline 按 `to_dict()` 形状消费（`.get(...)`）。
   存储态恢复时走 `resolve_stored_document_parser_engine`（`routing.py:1547`）：
   `parse_format == lightrag` → `ReuseParser`，非 `pending_parse` → `PassthroughParser`。

### 调用链三：原生引擎的模板方法与 LLM 桥（native 子树）

1. `NativeParserBase.parse`（`native_base.py:226`）先 `validate_source`，再在 worker 线程跑
   `validate_source_blocking` + `extract`（抽象钩子），随后 loop 上 `build_ir` 建 `IRDoc`。
2. 若 `wants_llm_bridge(engine_params)`（`native_base.py:118`）为真（docx 的 `smart_heading`），
   基类经 `_build_llm_submit`（`native_base.py:127`）构造 `SyncLLMBridge`（`llm_bridge.py:32`）：
   在 loop 线程捕获 loop 与线程 id，worker 线程 `__call__` 用 `run_coroutine_threadsafe`
   提交协程，并以 `poll_interval` 切片轮询 `future.result`——每片之间查取消事件。
3. 取消事件由 `ParseContext.pipeline_cancel_event`（`base.py:60`）传入；`first_cancellation`
   （`exceptions.py:55`）命中即 `future.cancel()` 并抛 `ParseCancelled` 族，pipeline 据此
   把文档标为取消而非失败。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `LIGHTRAG_PARSER` | 解析规则串；文件名提示 > 规则 > 后缀默认 > legacy | `routing.py:1102`（`parser_rules_from_env`） |
| `MINERU_API_MODE` | `local`；`official` 需 `MINERU_API_TOKEN`，`local` 需 `MINERU_LOCAL_ENDPOINT` | `registry.py:45` |
| `MINERU_LOCAL_ENDPOINT` / `MINERU_API_TOKEN` | MinerU 端点门禁闭包 | `registry.py:46-51` |
| `DOCLING_ENDPOINT` | Docling 端点门禁；未配置则 docling 不出现在 `available_engine_suffixes` | `registry.py:298` |
| `MINERU_ADDITIONAL_SUFFIXES` / `DOCLING_ADDITIONAL_SUFFIXES` | 逗号分隔扩展后缀，实时合并进 `spec.suffixes`，畸形 token 启动报错 | `registry.py:102` |
| `DOCX_SMART_HEADING` | smart_heading 全局默认开关；播种进持久化 `parse_engine` 使重解析免疫 env 翻转 | `routing.py:1226`、`:1248` |
| process_options 字符 `i/t/e/!/F/R/V/P/C` | images/tables/equations/skip_kg/分块策略选择；`F/R/V/P/C` 五选一 | `routing.py:194`（`validate_process_options`） |
| entry point 组 `lightrag.parsers` | 第三方引擎注册入口；每进程 `load_third_party_parsers` 一次（幂等） | `plugins.py:30` |

## 5. 错误与重试语义

- **引擎不可用回退**：文件名提示的引擎若后缀不支持或端点未配，不报错，而是置空提示引擎
  转入规则/默认解析（`routing.py:1455-1461`）；`require_external_endpoint=True` 的严格校验路径
  由 `_validate_filename_hint_for_resolution`（`routing.py:930`）先行抛错。
- **畸形环境后缀**：`extra_suffixes_env` 里非 `[a-z0-9]+` 的 token 不静默吞，
  `malformed_env_suffixes`（`registry.py:395`）在启动检查 `validate_parser_suffix_env_vars`
  （`routing.py:1133`）时上报。
- **插件失败隔离**：第三方 entry point 抛异常仅记 error 日志并跳过该插件，不影响内置引擎注册
  （`plugins.py:56-63`）。
- **取消语义**：阻塞解析等待被取消时抛 `ParseCancelled` 族；pipeline 捕获
  `ParsePipelineCancelled` 标文档取消，**不**走 FAILED 重试路径（FAILED 只能经手动重试进入，见 pipeline 域）。
- **LLM 桥死锁防护**：在 loop 线程内调用 `SyncLLMBridge.__call__` 直接 `RuntimeError`
  （`llm_bridge.py:72`），避免"等待自己阻塞的 loop"的必然死锁。
- **不做重试**：解析路由层本身不含退避/重试；失败重试归属 pipeline 域（一次手动尝试）。

## 6. 并发细节

- **快照一致性**：pipeline 批次开始时 `parser_specs_snapshot()`（`registry.py:326`）拍一份
  注册表浅拷贝，贯穿队列构造、路由与 parse worker；并发 `register_parser` 不改本批次引擎集。
- **队列分组**：`ParserSpec.queue_group` 决定引擎落在哪个并发池——native/legacy 共享 native 池
  （本地无网络），mineru/docling 各自独立池（`max_parallel_parse_mineru/docling`）。
- **跨线程边界**：原生 `extract` 是同步钩子，跑在 worker 线程；LLM 调用在异步 loop。
  `SyncLLMBridge` 是唯一穿越点，`run_coroutine_threadsafe` 提交 + 短轮询使取消可及时打断，
  而轮询间隔**不是** LLM 超时（总超时/重试归属 role wrapper）。
- **取消传播**：`pipeline_cancel_event` 由活动批次设置；桥在每次轮询片查事件，命中即取消 future。
- **实例缓存无锁**：`_INSTANCE_CACHE` 按 `(engine, impl)` 写一次后只读；注册表字面量在 import 时定型。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/parser/` 下路由、注册、分发、模板方法、LLM 桥、取消族、noop 处理器、调试 CLI；
  各引擎解析算法（docx 结构切分、外部 HTTP 适配器、markdown 抽取）在相邻叶子。

**Out-of-Scope（不在本仓库源码内）**
- **MinerU 服务、Docling-serve 服务本身**（外部 HTTP 解析服务）——本仓库仅持有客户端适配器，
  服务端不在本仓库源码内（见 L18）。
- python-docx、httpx、python-docx 依赖等第三方库——不在本仓库源码内。
- 文档摄入的入队/并发池/状态机——见 pipeline 域（L04/L05）。
- sidecar `*.parsed/` 落盘格式——见 sidecar 叶子（L20）。

## 8. 与相邻子系统交互

- **上游 pipeline（L04/L06）** → 本叶子：pipeline 调 `resolve_file_parser_engine` /
  `get_parser(engine).parse(ctx)`；`ParseContext` 反向惰性 import pipeline 的
  `_call_source_file_resolver`/`_read_source_file`/`archive_docx_source_after_full_docs_sync`。
- 本叶子 → **下游引擎实现**：native → `docx.parser:NativeDocxParser`（L17）/
  `markdown.parser:NativeMarkdownParser`（L19）；mineru/docling → `external/*`（L18）；
  reuse/passthrough → `noop.py`。
- 本叶子 → **sidecar（L20）**：引擎产出的 `IRDoc` 由 sidecar writer 落 `*.parsed/`；
  `ReuseParser` 反向读 `sidecar_blocks_path`。
- 本叶子 → **API server**：`available_engine_suffixes` 是上传白名单；`create_app` 启动时
  `load_third_party_parsers` + `validate_parser_routing_config`。

## 9. 语言专项适配口径（纯 Python 框架）

本叶子是**典型的"能力缝 + 注册表 + 插件体系"**（language-python.md 口径）：

1. **能力缝**：以 `BaseParser` ABC 定义契约缝，6 个引擎是缝下的实现方；行为与元数据分离——
   `BaseParser` 只背 `parse` 行为，能力元数据全在 `ParserSpec`，使能力查询零重依赖 import。
2. **注册表/工厂**：字符串 `engine_name` → `ParserSpec` → `"module:Class"` 惰性 import，
   与存储层 `STORAGE_IMPLEMENTATIONS`/`factory.get_storage_class()` 同构；注册时机为
   import 时字面量定型 + 运行时 `register_parser` 覆盖；实例缓存按 `(engine, impl)` 防陈旧。
3. **可选依赖/懒加载**：heavy 实现包（httpx 等）只在 `get_parser` 真解析时拉入；
   顶层 `import lightrag.parser.registry` 不拉任何解析实现——这是 import 面性能工程。
4. **配置驱动静态分派**：运行时用哪个引擎由文件名提示/`LIGHTRAG_PARSER` 规则/env 端点门禁
   **配置对象**决定，而非显式 `if engine == ...` 链；`process_options` 选择串再静态选分块策略。
5. **插件面**：`lightrag.parsers` entry point 组是宿主方扩展点，第三方包为扩展方；
   宿主（API server / 调试 CLI）在路由校验前调用 `load_third_party_parsers`。
6. **图型侧重**：architecture（注册表路由 + 外部边界）为主，sequence（解析分发调用链）为辅；
   无单实体状态机/数据管道语义，故不补 lifecycle/dataflow（见第 10 节省略说明）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 解析引擎注册表与路由架构图 | `parser-engine-architecture.html` | architecture | showcase |
| 引擎解析分发调用链时序图 | `parser-engine-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。

**省略说明**：本叶子未生成 dataflow 与 lifecycle 图——本叶子的底层语义是"组件路由与单次调用链"，
数据载体流动（文档→解析→IR→sidecar）属于域级 dataflow（见 `../parser.md`）而非本叶子内管道；
解析任务状态机属于 pipeline 域文档状态（见 L04），本叶子无单实体状态变迁，按资源节省原则省略。
