# parser 域（文档解析域）

> 本文是 LightRAG 架构分析中 `parser` 域的域级总览。域内 5 个叶子子系统负责：把任意来源文档
> （docx / markdown / pdf / 图片等）解析为统一的 sidecar 中间产物。本域是 pipeline 摄入管线（L04）
> 的下游——pipeline 决定"何时解析"，本域决定"如何解析"。

## 域职责

- **四引擎路由**：legacy / native / mineru / docling 由 `routing.py` 统一路由，文件名 `[hint]`、
  `LIGHTRAG_PARSER` 规则、后缀默认、legacy 兜底四级优先级。
- **统一契约**：所有引擎实现 `BaseParser.parse(ctx) -> ParseResult`，能力元数据在注册表 `ParserSpec`，
  实现惰性装载。
- **统一产物**：各引擎产出 `IRDoc`，由 sidecar writer 落为 `*.parsed/` 目录（`blocks.jsonl` +
  条件多模态 json + assets/），供下游分块器消费。
- **外部服务编排**：MinerU / Docling 是外部 HTTP 服务，本域仅持有带端点门禁与 raw-bundle 缓存的客户端。

## 叶子索引

| 叶子 | 职责 | 关键源码 | 架构图 | 时序/数据流图 |
|---|---|---|---|---|
| [L16 parser-engine](parser-engine/parser-engine.md) | 解析路由/注册表/分发/调试 CLI/LLM 桥/取消族 | `routing.py`、`registry.py`、`native_dispatch.py` | [architecture](parser-engine/parser-engine-architecture.html) | [sequence](parser-engine/parser-engine-sequence.html) |
| [L17 docx-parser](docx-parser/docx-parser.md) | native docx 结构切分 + smart_heading 标题流 | `docx/parse_document.py`、`smart_heading/*` | [architecture](docx-parser/docx-parser-architecture.html)（standard） | [sequence](docx-parser/docx-parser-sequence.html) |
| [L18 external-parsers](external-parsers/external-parsers.md) | MinerU/Docling 外部 HTTP 适配器 + raw 缓存 | `external/_base.py`、`mineru/`、`docling/` | [architecture](external-parsers/external-parsers-architecture.html) | [sequence](external-parsers/external-parsers-sequence.html) |
| [L19 markdown-parser](markdown-parser/markdown-parser.md) | native markdown/textpack 抽取 + 图片 SSRF 防护 | `markdown/parser.py`、`extract.py` | [architecture](markdown-parser/markdown-parser-architecture.html) | [sequence](markdown-parser/markdown-parser-sequence.html) |
| [L20 sidecar](sidecar/sidecar.md) | sidecar 格式唯一可执行规范：IRDoc→`*.parsed/` | `sidecar/writer.py`、`ir.py` | [architecture](sidecar/sidecar-architecture.html) | [dataflow](sidecar/sidecar-dataflow.html) |

## 域级机制细节

### 四引擎路由（配置驱动静态分派）

路由优先级（`resolve_parser_directives`，`routing.py:1420`）：
1. 文件名 `[engine(options)]` 提示；
2. `LIGHTRAG_PARSER` 规则串首个匹配；
3. 后缀默认引擎；
4. legacy 兜底。

注册表（`registry.py:266`）装此模块零重依赖：native（docx/markdown 内分发）、legacy（多格式本地）、
mineru/docling（外部，带端点门禁）、reuse/passthrough（内部空格式）。`get_parser` 惰性 import 实现。

### 解析→sidecar 管道

```
源文件 → routing 选引擎 → BaseParser.parse → IRDoc → write_sidecar → *.parsed/
                                                          ↓
                                          blocks.jsonl（分块器输入）+ assets/
```

- native docx/markdown：本地解析；smart_heading 经 `SyncLLMBridge` 跨线程调 LLM。
- mineru/docling：上传→下载 raw bundle→IR；命中 raw 缓存则纯本地。

### 外部边界

MinerU 服务、Docling-serve 服务、python-docx/lxml/spaCy、httpx、LLM 服务——均不在本仓库源码内。

## 域级图

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| 四引擎路由架构图 | `parser-domain-architecture.html` | architecture | showcase |
| 解析→sidecar 管道数据流图 | `parser-domain-dataflow.html` | dataflow | showcase |
| 解析分发时序图 | `parser-domain-sequence.html` | sequence | showcase |

JSON IR 位于 `json/` 目录。
