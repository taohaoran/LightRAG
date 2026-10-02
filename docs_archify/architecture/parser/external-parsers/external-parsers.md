# 外部解析适配器（external-parsers）

> 本文是 `parser` 域下的叶子子系统文档。域级总览见 `../parser.md`。本文只展开 **MinerU / Docling
> 两个外部 HTTP 解析服务的客户端适配器**：模板方法复用、raw-bundle 缓存、IR 构建。
> **MinerU / Docling 服务端本身不在本仓库源码内**，本仓库只持有客户端。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 外部引擎模板方法 | `ExternalParserBase.parse` 固化统一流程：resolve→raw_dir→force-reparse→缓存命中跳过/否则下载→build_ir→write_sidecar→持久化 full_docs→归档源文件 | `lightrag/parser/external/_base.py:83` |
| 三个引擎私有钩子 | 子类实现 `is_bundle_valid`（廉价缓存命中判定）/`download_into`（拉取 raw bundle）/`build_ir`（bundle→IRDoc），并设 `raw_dir_suffix`/`force_reparse_env` | `_base.py:42/59/75` |
| MinerU 适配器 | `MinerUParser`：raw 目录后缀、force-reparse env、三钩子委托给 mineru 子包 | `lightrag/parser/external/mineru/parser.py:11` |
| Docling 适配器 | `DoclingParser`：额外 `validate_ir`——零块即响亮报错；`upload_name` 映射为 docling-serve multipart 文件名 | `lightrag/parser/external/docling/parser.py:11` |
| 原始 bundle 客户端 | `MinerURawClient`（742 行）/`DoclingRawClient`（537 行）：上传文件、轮询/等待、下载 raw bundle | `mineru/client.py`、`docling/client.py` |
| bundle 缓存 | `is_bundle_valid` 按 raw 目录+源文件+engine_params 判定缓存命中；命中时纯本地，外部端点临时不可用仍可重解析 | `_base.py:124-130`、`mineru/cache.py`、`docling/cache.py` |
| IR 构建 | `MinerUIRBuilder`（885 行）/`DoclingIRBuilder`（1068 行）：raw bundle workdir → `IRDoc` | `mineru/ir_builder.py`、`docling/ir_builder.py` |
| manifest/zip 共用件 | `_manifest.py`（bundle 清单）、`_zip.py`（bundle zip 解包）、`_common.py`（clear_dir_contents/env_bool/raw_dir_for_parsed_dir） | `external/_*.py` |
| 引擎参数编解码 | 每文件 engine_params（如 `mineru(page_range=1-3)`）经 `decode_parse_engine` 解出，同一 dict 贯穿缓存判定与下载，防"参数变了却命中旧 bundle"；畸形指令响亮失败 | `_base.py:102-112` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `ExternalParserBase` | `_base.py:32` | 外部引擎模板方法基类，继承 `BaseParser` |
| `parse` 模板 | `_base.py:83` | 固化 resolve→缓存/下载→IR→sidecar→持久化→归档全流程 |
| `MinerUParser` / `DoclingParser` | `mineru/parser.py:11` / `docling/parser.py:11` | 钩子实现 + 常量配置 |
| `MinerURawClient` / `DoclingRawClient` | `mineru/client.py` / `docling/client.py` | HTTP 上传与 raw bundle 下载 |
| `MinerUIRBuilder` / `DoclingIRBuilder` | `mineru/ir_builder.py` / `docling/ir_builder.py` | raw bundle → `IRDoc` |
| `raw_dir_for_parsed_dir` | `_common.py` | 由 sidecar 目录派生 raw bundle 目录 |

## 3. 关键调用链

### 调用链：外部引擎一次解析（模板方法）

1. pipeline 经 `get_parser("mineru"/"docling")` 取实例（见 L16），调 `parse(ctx)`
   （`_base.py:83`）。先 `decode_parse_engine` 解出持久化指令里的每文件参数
   （`_base.py:102`）；畸形指令抛 `ValueError` 响亮失败。
2. `ctx.resolve(engine_name)` 派生 `(source_path, document_name, parsed_dir)`
   （`_base.py:114`）；`raw_dir = raw_dir_for_parsed_dir(parsed_dir, suffix=raw_dir_suffix)`（`:120`）。
3. 缓存分支（`_base.py:124`）：未设 force-reparse 且 `is_bundle_valid(raw_dir, source, engine_params)`
   为真 → `parse_stage_skipped=True`，纯本地不碰外部端点。
4. 缓存未命中（`_base.py:131`）：force-reparse 时记录并丢弃旧 bundle，`mkdir`+`clear_dir_contents`
   （`:141-142`）后 `await download_into(...)`（`:149`）从外部服务拉取 raw bundle。
5. `build_ir(raw_dir, document_name)`（`:156`）→ `validate_ir`（`:157`，docling 零块报错）→
   `write_sidecar(ir, ...)`（`:158`，见 L20）→ `_persist_parsed_full_docs` 写 `lightrag` 行
   （`:165`，重新 `encode_parse_engine` 保留每文件参数）→ `archive_source`（`:180`）。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `MINERU_API_MODE` / `MINERU_LOCAL_ENDPOINT` / `MINERU_API_TOKEN` | MinerU 端点门禁（见 L16） | `registry.py:45` |
| `DOCLING_ENDPOINT` | Docling 端点门禁 | `registry.py:298` |
| `LIGHTRAG_FORCE_REPARSE_MINERU` / `..._DOCLING` | 置位则丢弃旧 bundle 强制重解析 | `mineru/parser.py`、`docling/parser.py` |
| `MINERU_ADDITIONAL_SUFFIXES` / `DOCLING_ADDITIONAL_SUFFIXES` | 部署期扩展可解析后缀 | `registry.py:102` |
| 每文件引擎参数 | `mineru(page_range=..., language=...)` / `docling(force_ocr=...)` | `_base.py:102` |

## 5. 错误与重试语义

- **畸形 parse_engine 指令**：`decode_parse_engine` 报错即抛 `ValueError`，本文档 FAILED，
  不静默退化为无参数解析（`_base.py:107-111`）。
- **源文件缺失**：`source.is_file()` 为假抛 `FileNotFoundError`（`_base.py:116-119`）。
- **Docling 零块**：`validate_ir` 对空 blocks 抛 `ValueError`（`docling/parser.py:validate_ir`）。
- **缓存容错**：命中缓存时纯本地，外部端点临时不可用仍能完成重解析（`_base.py:127-128`）。
- 网络重试/超时归属 HTTP 客户端与 role 层；本模板不含退避，失败由 pipeline 一次性手动重试处理。

## 6. 并发细节

- **队列分组隔离**：mineru/docling 各自独立 queue_group（`max_parallel_parse_mineru/docling`），
  与本地 native/legacy 池隔离（见 L16）。
- **异步下载**：`download_into` 是协程，在 parse worker 中 await；模板方法整体为 async。
- **缓存一致性**：engine_params 同时喂给缓存判定与下载，保证"参数不同不命中旧 bundle"。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/parser/external/`：模板方法基类、MinerU/Docling 适配器、HTTP 客户端、IR 构建、
  bundle 缓存与 manifest/zip 共用件。

**Out-of-Scope（不在本仓库源码内）**
- **MinerU 服务**（官方 API 与本地 mineru-server）——外部 HTTP 服务，本仓库仅客户端。
- **Docling-serve 服务**——外部 HTTP 服务，本仓库仅客户端。
- httpx 等 HTTP 库——第三方，不在本仓库源码内。
- sidecar 落盘格式——见 L20。

## 8. 与相邻子系统交互

- **上游 L16**：注册表 `_REGISTRY` 按 `impl` 串惰性装载 `MinerUParser`/`DoclingParser`；
  端点门禁闭包决定是否出现在 `available_engine_suffixes`。
- 本叶子 → **外部服务**：`download_into` 经 HTTP 上传源文件、下载 raw bundle。
- 本叶子 → **下游 L20 sidecar**：`build_ir` 产出 `IRDoc`，`write_sidecar` 落 `*.parsed/`。
- 本叶子 → **pipeline**：`_persist_parsed_full_docs` 写 `lightrag` 行，`archive_source` 归档源文件。

## 9. 语言专项适配口径（纯 Python 框架）

- **模板方法缝**：`ExternalParserBase.parse` 固化共性流程，三钩子（is_bundle_valid/download_into/build_ir）
  是扩展缝，MinerU/Docling 是缝下两个实现方——"目录即实例库"型双实现矩阵。
- **编排层定位**：本叶子不做实际解析，只编排"上传→下载→bundle 转 IR"，实际文档理解在外部服务完成
  （编排层，外部计算方全部标注不在本仓库源码内）。
- **图型侧重**：architecture（模板基类+双适配器+外部边界）+ sequence（缓存/下载分支调用链）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 外部解析适配器架构图 | `external-parsers-architecture.html` | architecture | showcase |
| 缓存命中/下载分支时序图 | `external-parsers-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。

**省略说明**：本叶子未生成 dataflow/lifecycle 图——raw bundle 的数据管道在 parser 域级 dataflow 表达；
外部解析任务无独立于 pipeline 的状态机（缓存命中/下载是分支判断而非实体状态迁移），按资源节省原则省略。
