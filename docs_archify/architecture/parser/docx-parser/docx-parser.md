# DOCX 原生解析器（docx-parser）

> 本文是 `parser` 域下的叶子子系统文档。域级总览见 `../parser.md`。本文只展开 **native 引擎
> 对 `.docx` 的结构切分与 smart_heading 标题流发现**；引擎路由/注册表见 `parser-engine`（L16），
> 解析产物 sidecar 落盘见 `sidecar`（L20），markdown 原生解析见 `markdown-parser`（L19）。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| native DOCX 适配器 | `NativeDocxParser` 实现 `NativeParserBase` 钩子；`extract`→`extract_docx_blocks`，`build_ir`→`NativeDocxIRBuilder` | `lightrag/parser/docx/parser.py:26` |
| 解压预算防护 | `enforce_docx_decompression_budget` 在打开压缩包前读中央目录，拒绝 zip-bomb（数十万成员包）；刻意放在 worker 线程而非事件循环 | `lightrag/parser/docx/zip_budget.py`、`parser.py:46` |
| 结构切分主入口 | `extract_docx_blocks`：python-docx 打开→NumberingResolver→styles outline→单遍读记录→按模式组装；**一标题一块**，块大小交给下游 paragraph-semantic 分块器 | `lightrag/parser/docx/parse_document.py:1076` |
| 单遍读记录 | `_read_document_records` 同时供 legacy/smart 两种组装复用，产出 `ParagraphRecord` 序列 | `parse_document.py:684`、`:627` |
| 自动编号还原 | `NumberingResolver` 捕获列表标签（编号前缀），消除 OOXML 编号与文本分离问题 | `numbering_resolver.py`（703 行） |
| 样式大纲级别 | `parse_styles_outline_levels` 解析 `styles.xml` 的 outlineLvl | `parse_document.py:296` |
| 表格转 JSON | 表格抽为二维数组并以 `<table>` 占位符发射；上标/下标转 `<sup>`/`<sub>` | `table_extractor.py`、`parse_document.py:433` |
| 绘图/图片抽取 | `DrawingExtractionContext`+`load_relationships` 抽取绘图关系并导出资产 | `drawing_image_extractor.py`（635 行）、`parser.py:80` |
| smart_heading 物理特征 | `parse_styles_attributes`（沿 basedOn 继承链解析有效字体/对齐）+ `extract_paragraph_physical_features`（字重/对齐/前导空格/分页符/TOC 证据），字号按 0.5pt 网格精确比较 | `smart_heading/features.py` |
| 护栏纯函数层 | 每条规则投"反对标题"票（do-no-harm，平局算非标题）；I1 内容保全/I2 基线标题保留/I3 锚点语义/输出长度四项 landing 校验 | `smart_heading/guardrails.py` |
| 标题块发现 | 启发式找候选窗口（缺强正文特征、含视觉主导行），LLM 经同步 judge 确认并拆分主/副标题；输出严格 locate-back 校验 | `smart_heading/title_block.py` |
| 标题流编排 | `run_smart_heading` 全流程：TOC 识别与保留→CB4 整文档门槛→标题块→候选门控→分级→合并/降级→编号骨架纠正 | `smart_heading/heading_flow.py:2942` |
| 审计台账 | smart 产出写 `<base>.smart_audit.json`（无时间戳+sort_keys 保证重解析字节一致，I4 确定性） | `parser.py:120`（`finalize_parse_warnings`） |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `NativeDocxParser` | `parser.py:26` | native docx 引擎适配器；`wants_llm_bridge` 仅在 `smart_heading` 开启时为真 |
| `extract_docx_blocks` | `parse_document.py:1076` | 结构切分主入口；smart 关闭走 legacy 组装，开启走 heading_flow+护栏 |
| `ParagraphRecord` | `parse_document.py:627` | 单遍读产出的物理记录（段落/表格/标题级别/paraId/物理特征） |
| `DocxContentError` | `parse_document.py:52` | 损坏/伪 docx 的可操作错误，pipeline 据此只标本文档 FAILED |
| `HeadingDecision` | `heading_flow.py:137` | 单候选标题的决策载体（级别/锚点 delta/规则备注） |
| `SmartHeadingResult` | `heading_flow.py:2717` | heading_flow 产物：decisions/toc_indices/audit/fallback 子文档 |
| `GateResult` | `heading_flow.py:319` | 候选门控结果 |
| `TitleBlockLLMError` | `title_block.py` | LLM 输出畸形（缺字段/越界/同索引双投）时响亮抛错，绝不静默降级 |
| `NativeDocxIRBuilder` | `ir_builder.py` | blocks → `IRDoc`（sidecar 契约） |
| `NumberingResolver` | `numbering_resolver.py` | OOXML 自动编号还原 |

## 3. 关键调用链

### 调用链一：native 模板方法到结构切分（基础路径）

1. `NativeParser.parse`（`native_dispatch.py:53`）按后缀选中 docx，调用 `NativeDocxParser.parse`
   （经 `NativeParserBase.parse` 模板，`native_base.py:226`）：先 `validate_source`（`parser.py:38`）
   校验 `.docx`，worker 线程再 `validate_source_blocking`→`enforce_docx_decompression_budget`（`parser.py:57`）。
2. 模板方法调 `extract`（`parser.py:62`）→ `extract_docx_blocks`（`parse_document.py:1076`）：
   `Document(file_path)` 失败时 `_diagnose_invalid_docx` 抛 `DocxContentError`；构造 `NumberingResolver`
   与 `parse_styles_outline_levels`，再 `_read_document_records`（`parse_document.py:1158`）单遍读记录。
3. smart 关闭时 `_assemble_blocks_legacy`（`parse_document.py:1168`，特征锁定重构前行为）；
   随后 loop 上 `build_ir`→`NativeDocxIRBuilder.normalize`（`parser.py:103`）产出 `IRDoc`。

### 调用链二：smart_heading 标题流与 landing 护栏

1. smart 开启时 `extract_docx_blocks` 先 `parse_styles_attributes`（`parse_document.py:1152`）补物理特征，
   再 `run_smart_heading(records, llm_judge=...)`（`parse_document.py:1177`、`heading_flow.py:2942`）。
2. `run_smart_heading` 内：`g.detect_toc_records`+`plan_toc_output` 识别并保留前 N 行 TOC
   （`heading_flow.py:2999-3011`）；CB4 整文档门槛 `_estimate_record_tokens < min_tokens` 直接 `return None`
   ——零 LLM 调用、回退基线（`heading_flow.py:3030`）；随后 `tb.detect_imprint_regions`（公文印发区，`:3057`）。
3. `_assemble_blocks_smart` 产出 smart 块后做四项 landing 校验（`parse_document.py:1197-1215`）：
   I1 `verify_content_preservation`、I2 `verify_baseline_heading_retention`、I3 `verify_anchor_semantics`、
   `smart_output_length_ok`；任一违例即 `_assemble_blocks_legacy` 回退（同一批 records 内存合成影子基线，
   无二次 IO，`:1195`）。
4. 接受后把 `smart_audit` 写进 `parse_metadata`，由 `finalize_parse_warnings`（`parser.py:120`）落
   `<base>.smart_audit.json`；smart_* 告警进 sidecar，其余进 `doc_status.metadata`。

### 调用链三：LLM 判定的跨线程桥

- `wants_llm_bridge(engine_params)`（`parser.py:59`）为真时，`NativeParserBase._build_llm_submit`
  （`native_base.py:127`）构造 `SyncLLMBridge`；title_block 的 LLM judge 经它在 worker 线程同步等待，
  短轮询响应 `pipeline_cancel_event`（见 L16 调用链三）。LLM 输出畸形抛 `TitleBlockLLMError` 响亮失败，
  欠规约分区则按"弃权"恢复而非整文档失败。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `DOCX_SMART_HEADING` | smart_heading 总开关；播种进持久化 parse_engine | `routing.py:1226`、`parser.py:60` |
| `DOCX_SMART_MIN_TOKENS` | CB4 整文档门槛，低于则零 LLM 回退基线 | `heading_flow.py:2984` |
| `DOCX_SMART_SUBDOC_MIN_TOKENS` | 子文档门槛，默认随 min_tokens 下调 | `heading_flow.py:2985` |
| `DOCX_SMART_TOC_KEEP_LINES` | TOC 保留前 N 可见行作正文，余者折为"……" | `heading_flow.py:3008` |
| `DOCX_SMART_HEADING_MAX_CHARS` | 标题最大长度硬上限，入口与启动双重校验 | `guardrails.py`、`heading_flow.py:2972` |
| 解压预算 | 成员数/解压比阈值，worker 线程读中央目录即拦截 | `zip_budget.py` |

## 5. 错误与重试语义

- **损坏 docx**：`PackageNotFoundError` 被改写为带诊断（magic bytes）与修复建议的 `DocxContentError`，
  pipeline 只把本文档标 FAILED，不影响批次（`parse_document.py:1128-1138`）。
- **smart 护栏回退**：I1/I2/I3/长度任一违例不报错失败，而是**本文档回退基线输出**并记
  `smart_fallback_baseline=1`（`parse_document.py:1216-1235`）——这是 do-no-harm：宁可不要 smart 也不丢内容。
- **CB4 短文档**：不跑 smart、零 LLM，输出基线 TOC 完整（`:1180-1184`）。
- **LLM 输出畸形**：`TitleBlockLLMError` 响亮抛错（不静默降级）；欠规约分区按弃权恢复。
- **缺 paraId**：仅告警计数（`missing_paraid_count`），每文档一行日志，不中断。
- 本叶子不含重试；失败/重试归属 pipeline 域。

## 6. 并发细节

- **worker 线程边界**：`extract` 是同步钩子跑在 worker 线程；`validate_source_blocking`（读数十万成员
  中央目录耗时秒级）刻意不放在事件循环，避免阻塞无关请求（`parser.py:46-57` 注释）。
- **LLM 桥**：仅 smart_heading 开启时跨线程；loop 线程内调用直接报错防死锁（见 L16）。
- **确定性写入**：`smart_audit.json` 无时间戳+`sort_keys`，重解析字节一致（I4）；I4 缓存关闭标记
  `smart_i4_cache_disabled` 一并入账。
- **无共享可变状态**：单文档解析在 records 本地进行，影子基线同内存合成，无跨文档锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/parser/docx/` 全部：适配器、结构切分、smart_heading 四件套、编号/表格/绘图抽取、
  IR 构建、解压预算。

**Out-of-Scope（不在本仓库源码内）**
- **python-docx、lxml、spaCy**（smart_heading 可选 NLP 依赖）——第三方库，不在本仓库源码内；
  smart-off 路径刻意不 import smart_heading 包（`:1150`）。
- LLM 服务本身——本叶子只经 `SyncLLMBridge` 调用 role wrapper，模型服务不在本仓库源码内。
- 块大小切分（长块锚点拆分/小块合并）——**故意不做**，归属 paragraph-semantic 分块器（L23）。

## 8. 与相邻子系统交互

- **上游 L16**：`NativeParser`（`native_dispatch.py`）按后缀选中本叶子；`NativeParserBase` 模板驱动。
- 本叶子 → **下游 L23 分块器**：产出"一标题一块"的 sidecar 契约，块大小留给 paragraph-semantic。
- 本叶子 → **下游 L20 sidecar**：`NativeDocxIRBuilder` 产出 `IRDoc`；`smart_audit.json` 落 `*.parsed/`。
- 本叶子 → **LLM 桥（L16）**：title_block 经 `SyncLLMBridge` 同步调 LLM judge。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝**：`NativeParserBase` 模板方法缝（extract/build_ir 抽象钩子 + validate/warnings 收口钩子），
  docx 是缝下的具体实现方；行为与"do-no-harm 护栏"分离——护栏纯函数层不反向依赖 heading_flow。
- **双轨实现**：同一 `extract_docx_blocks` 有 legacy（特征锁定基线）与 smart（LLM+物理特征）双路径，
  由 `smart_heading` 配置**静态分派**；smart-off 路径零 smart 包依赖（可选依赖懒加载）。
- **配置驱动分派**：运行时是否走 smart、门槛阈值全由 env/engine_params 决定。
- **图型侧重**：architecture（四件套分层+外部边界）+ sequence（smart 标题流与护栏回退调用链）；
  数据管道（文档→块→IR）属域级 dataflow，单文档状态机属 pipeline 域，故不补 lifecycle/dataflow。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| DOCX 解析与 smart_heading 四件套架构图 | `docx-parser-architecture.html` | architecture | standard |
| smart 标题流与护栏回退时序图 | `docx-parser-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。

**架构图降档说明**：architecture 落 standard，失败检查为 `composition/proper-crossing`——`features→headingflow`
竖边与 `extract→irbuilder` 主路径横边在同一走廊交叉；已两轮调整网格（重排为行列对齐、删除穿节点的
pydocx/c12 边）后仍剩一处不可消除的走廊交叉，按规则拆图/降档，render 退出码 0、HTML 806KB。

**省略说明**：本叶子未生成 dataflow/lifecycle 图——文档→块→IR→sidecar 的数据管道语义在 parser 域级
dataflow 图表达；单文档解析无独立于 pipeline 的状态机（CB4 门槛是分支判断而非实体状态迁移），
按资源节省原则省略。
