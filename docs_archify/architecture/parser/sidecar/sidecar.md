# Sidecar 写入器（sidecar）

> 本文是 `parser` 域下的叶子子系统文档。域级总览见 `../parser.md`。本文只展开 **Sidecar 文件格式的
> 唯一可执行规范**：引擎适配器产出 `IRDoc`，`write_sidecar` 把它落成 `*.parsed/` 目录。
> 格式规范见 `docs/LightRAGSidecarFormat-zh.md`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| IR 数据模型 | `IRDoc`/`IRBlock`/`IRTable`/`IRDrawing`/`IREquation`/`IRPosition`/`AssetSpec` dataclass；引擎适配器只翻译，不感知磁盘格式 | `lightrag/sidecar/ir.py:219`（`IRDoc`） |
| 占位符约定 | `content_template` 里 `{{TBL:k}}`/`{{IMG:k}}`/`{{EQ:k}}`/`{{EQI:k}}`，每个 item 恰好一个占位符 | `ir.py:17-27` |
| 唯一可执行规范 | `write_sidecar` 是格式规范唯一落点：id 分配、占位符渲染、blockid、asset_dir 判定 | `lightrag/sidecar/writer.py:60` |
| id 分配 | `tb-/im-/eq-<doc_hash>-NNNN`（4 位补零，文档级全局序列） | `writer.py` 模块 docstring |
| 占位符渲染 | `render_template` 把 token 渲染为规范 XML 风格标签；`xml_attr_escape` 属性转义 | `lightrag/sidecar/placeholders.py` |
| blockid 计算 | `md5(doc_id:block_index:heading:content)` | `writer.py` docstring |
| 资产物化 | `_materialize_assets`：Path 复制 / bytes 落盘 / None 已就位仅记大小；唯一命名 `_allocate_unique_name`、安全文件名 `_safe_asset_filename` | `writer.py:397/465/480` |
| 条件写 | `tables.json`/`drawings.json`/`equations.json` 仅在对应字典非空时出现 | `writer.py` docstring |
| chunk 来源回填 | `backfill_chunk_sidecars`：F/R/V 策略按 chunk 的 `_source_span` 反查 blocks.jsonl，补 `sidecar` 溯源 | `lightrag/sidecar/backfill.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `IRDoc` | `ir.py:219` | 顶层 IR，write_sidecar 的输入 |
| `IRBlock` | `ir.py:171` | 一个内容块（content_template/heading/level/positions/tables/drawings/equations） |
| `IRTable`/`IRDrawing`/`IREquation` | `ir.py:73/128/151` | 多模态对象，带 `placeholder_key` 与 `self_ref` 溯源 |
| `AssetSpec` | `ir.py:196` | 待落盘资产（Path/bytes/None） |
| `write_sidecar` | `writer.py:60` | IRDoc → `*.parsed/` 目录主入口 |
| `render_template` | `placeholders.py` | 占位符 token → 规范标签 |

## 3. 关键调用链

### 调用链：IRDoc 落为 sidecar 目录

1. 任一引擎适配器（native docx/markdown、mineru/docling）构建 `IRDoc`，调
   `write_sidecar(ir, parsed_dir=..., doc_id=..., engine=...)`（`writer.py:60`）。
2. writer 做 id 分配（`tb-/im-/eq-<doc_hash>-NNNN`），随后 `_render_block_content`（`writer.py:487`）
   经 `render_template` 展开每个块的 `content_template`：`_table`/`_drawing`/`_equation`/`_inline_equation`
   四个闭包（`:503/520/540/551`）把占位符换成规范标签。
3. `_materialize_assets`（`writer.py:397`）把 `AssetSpec` 落进 `<base>.blocks.assets/`：
   Path 复制、bytes 落盘、None 已就位仅记大小；`asset_dir` 标志由"目录存在且非空"派生。
4. 写 `blocks.jsonl`（meta 行 + 块行），条件写 `tables.json`/`drawings.json`/`equations.json`，
   产出 `merged_text` 与 `document_hash`；返回 `{content, blocks_path, ...}` 供 pipeline 持久化。

### 调用链：chunk 来源回填（F/R/V 策略）

- `backfill_chunk_sidecars`（`backfill.py`）从 `blocks.jsonl` 精确重建 merged text
  （`"\n\n".join` 非空 content 行），记录每个块字符 span，把 chunk 的 `_source_span` 映射到重叠块，
  给 chunk 补 `sidecar` 溯源；V 策略按空白剥离后匹配（句子重排仅差空白）。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| 占位符格式 | `{{TBL:k}}/{{IMG:k}}/{{EQ:k}}/{{EQI:k}}` | `placeholders.py` `_TOKEN_RE` |
| id 格式 | `tb-/im-/eq-<doc_hash>-NNNN` | `writer.py` |
| blockid | `md5(doc_id:block_index:heading:content)` | `writer.py` |
| inline 公式 | `{{EQI:k}}` 不分配 id、不进 equations.json | `ir.py:152` |

## 5. 错误与重试语义

- 引擎适配器不写规范 JSON，全部经 writer；占位符必须每 item 恰好一个（多占位符同 key 不支持，`ir.py:25`）。
- 条件写避免空文件；writer 是纯本地磁盘写，不含网络/重试。
- 块 span 校验：字节精确或空白剥离匹配（V 策略），不匹配即不补溯源。

## 6. 并发细节

- writer 单文档一次性同步写；id 序列文档级全局、按文档隔离，无跨文档锁。
- `AssetSpec.source=None` 情形（native docx 抽取时已写资产）仅记大小不触碰文件，避免重复写竞争。

## 7. 系统边界

**In-Scope（本仓库源码内）**：`lightrag/sidecar/` 全部（IR 模型、writer、占位符、chunk 回填）。

**Out-of-Scope（不在本仓库源码内）**：格式规范文档 `docs/LightRAGSidecarFormat-zh.md` 是规范本身；
chunk 策略消费方见 chunker 域（L21–L23）；引擎适配器见 L16–L19。

## 8. 与相邻子系统交互

- **上游 L16–L19**：各引擎适配器 → `write_sidecar(IRDoc)`。
- 本叶子 → **下游 chunker（L21–L23）**：`blocks.jsonl` 是分块输入；`backfill` 为 F/R/V chunk 补溯源。
- 本叶子 → **pipeline**：返回 `content`/`blocks_path` 供 `_persist_parsed_full_docs`。

## 9. 语言专项适配口径（纯 Python 框架）

- **单一可执行规范缝**：IR 是进程内中间表示（非序列化中间件），writer 是唯一懂磁盘格式处；
  引擎适配器只翻译不嵌格式知识——"规范即代码"。
- **extras 逃生舱**：item 级 `extras` dict 透传引擎特有信号（rowspan/OCR 置信），避免规范频繁改动。
- **图型侧重**：architecture（IR 模型→writer→输出目录）+ dataflow（IRDoc 经 id 分配/渲染/物化到 sidecar 文件）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Sidecar 写入器架构图 | `sidecar-architecture.html` | architecture | showcase |
| IRDoc 落盘数据流图 | `sidecar-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/` 目录。

**省略说明**：未生成 sequence/lifecycle 图——write_sidecar 是单趟同步落盘，无多方消息时序；
无实体状态机（id 分配是单调序列而非状态迁移），按资源节省原则省略。
