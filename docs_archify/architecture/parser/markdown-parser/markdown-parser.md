# Markdown 原生解析器（markdown-parser）

> 本文是 `parser` 域下的叶子子系统文档。域级总览见 `../parser.md`。本文只展开 **native 引擎对
> `.md` / `.textpack` 的抽取、图片解析与 SSRF 防护、IR 构建**；路由/注册表见 L16，docx 见 L17，
> sidecar 落盘见 L20。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| native markdown 适配器 | `NativeMarkdownParser` 实现 `NativeParserBase` 钩子；`.md` 直读，`.textpack`（zip TextBundle）解压 | `lightrag/parser/markdown/parser.py:1548` |
| textpack 解包 | `_open_textpack`/`_find_text_file`/`_extract_text`：安全解包 Bear/Ulysses TextBundle（text.markdown+assets/） | `parser.py:1644/1657/1690` |
| markdown 结构抽取 | `extract_markdown`：标题清洗、pipe 表/HTML 表/公式/绘图占位符识别，产出 MarkdownExtraction | `markdown/extract.py:359` |
| 图片解析器 | `_MarkdownImageResolver.resolve` 按 src 分四路：base64 data-URL、bundle 内相对引用、外部 http(s)、跳过/告警 | `parser.py:1124/1166` |
| SSRF/预算防护 | `_GuardedHTTPConnection`/`_GuardedRedirectHandler` 守卫 socket 与重定向；`_ImageBudget` 限总字节/请求数/总时长 | `parser.py:938/990/412` |
| 下载看门狗 | `_DownloadState` + watchdog 线程：取消事件、超时、主动关停 socket | `parser.py:506/590/656` |
| 越预算降级 | 超预算远程图降级为外链并记告警计数，不中断解析 | `parser.py:1256/1272` |
| IR 构建 | `NativeMarkdownIRBuilder`：抽取结果 → `IRDoc` | `markdown/ir_builder.py`（230 行）、`parser.py:1730` |
| 共享标题渲染 | `strip_heading_markdown_prefix`：跨引擎（docx/mineru/docling）标题行渲染一致，深度钳到 6 级 | `lightrag/parser/_markdown.py` |
| 共享 HTML 表解析 | `HTMLTableInfo`：colspan 感知的行列统计，合并单元格语义跨引擎一致 | `lightrag/parser/_html_table.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `NativeMarkdownParser` | `parser.py:1548` | native markdown 引擎适配器 |
| `_MarkdownImageResolver` | `parser.py:1124` | 图片 src 四路解析与预算降级 |
| `_ImageBudget` | `parser.py:412` | 总字节/请求数/总时长三预算 |
| `_DownloadState` | `parser.py:506` | 下载取消/超时/socket 关停状态机 |
| `ResolvedImage` / `ImageResolver` | `extract.py:80/100` | 解析结果与解析器协议 |
| `extract_markdown` | `extract.py:359` | markdown 结构抽取主函数 |
| `HTMLTableInfo` | `_html_table.py` | HTML 表结构事实 |

## 3. 关键调用链

### 调用链：一次 markdown 解析

1. `NativeParser.parse`（`native_dispatch.py:53`）按后缀选中 markdown，进
   `NativeParserBase.parse` 模板（`native_base.py:226`）→ `validate_source`（`parser.py:1552`）。
2. `extract`（`parser.py:1562`）：`.textpack` 走 `_open_textpack` 安全解包（`:1644`）取文本与资源目录；
   `.md` 直读。随后 `extract_markdown`（`extract.py:359`）逐行切分标题/表/公式。
3. 行内图片经 `_MarkdownImageResolver.resolve`（`parser.py:1166`）：data-URL 解码落地、bundle 内相对引用
   从安全解包目录解析（带路径穿越防护）、外部 http 经 `_GuardedHTTPConnection` 下载（`_resolve_remote`，`:1361`）。
4. `build_ir`（`parser.py:1730`）→ `NativeMarkdownIRBuilder` 产出 `IRDoc`，由模板写 sidecar。

### 调用链：外部图片下载的防护

1. `_resolve_remote`（`parser.py:1361`）→ `_download`（`:1440`）→ `_fetch`（`:1508`），
   经 `_GuardedHTTPConnection.connect`（`:939`）绑 socket 注册进 `_DownloadState.register_socket`（`:564`）。
2. 每图过 `_ImageBudget.charge_bytes`/`charge_request`/`check_time`（`:490/497/483`）；超限抛
   `_ImageByteBudgetExceeded` 等，被 `_drop_over_budget`/`_byte_budget_degraded`（`:1196/1272`）转为外链降级。
3. 取消事件/超时由 watchdog（`:590`）触发 `_shutdown_socket`（`:576`）打断阻塞下载。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `NATIVE_MD_IMAGE_DOWNLOAD_ENABLED` | 默认 true；false 则直接丢弃外部图（不发 drawing） | `parser.py` 模块 docstring |
| `NATIVE_MD_IMAGE_MAX_TOTAL_BYTES` | 单文档图片总字节内存上限 | 同上 |
| `NATIVE_MD_IMAGE_MAX_REQUESTS` | 远程抓取/重定向/未出主机尝试总数上限 | 同上 |
| `NATIVE_MD_IMAGE_DOWNLOAD_TOTAL_TIMEOUT` | 跨所有下载的墙钟总时长 | 同上 |

## 5. 错误与重试语义

- **越预算**：不中断解析，远程图降级为外链 + 告警计数（GHSA-25c3-j78v-83qx 资源边界）。
- **下载失败/取消**：发外链 fallback drawing；取消由 watchdog 关停 socket 打断。
- **bundle 内越界相对引用**：路径穿越防护拒绝，跳过+告警。
- 本叶子不含重试；失败/重试归属 pipeline 域。

## 6. 并发细节

- **worker 线程**：`extract` 同步跑在 worker 线程；下载看门狗在辅助线程，经 socket 关停打断阻塞。
- **取消传播**：`pipeline_cancel_event` 经 `_DownloadState.cancellation`（`:526`）传播到下载循环。
- 无跨文档共享可变状态。

## 7. 系统边界

**In-Scope（本仓库源码内）**：`lightrag/parser/markdown/` 与共享叶模块 `_markdown.py`/`_html_table.py`。

**Out-of-Scope（不在本仓库源码内）**：外部 http(s) 图片服务（本仓库仅带 SSRF 防护的下载客户端）；
python-docx 不涉及本叶子；sidecar 落盘见 L20。

## 8. 与相邻子系统交互

- **上游 L16**：`NativeParser` 按后缀（`.md`/`.textpack`）分发到本叶子。
- 本叶子 → **下游 L20 sidecar**：`IRDoc` 经 `write_sidecar` 落盘，图片资产进 sidecar assets。
- 本叶子 → **外部图片服务**：受 SSRF/预算防护的下载客户端。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝**：`NativeParserBase` 模板方法缝的另一实现方；`ImageResolver` 是 Protocol 缝。
- **可选依赖/安全边界**：图片下载自带 socket/重定向守卫（编排层不裸信外部 URL）。
- **叶模块共享**：`_markdown.py`/`_html_table.py` 零重 import，供多引擎复用一致语义。
- **图型侧重**：architecture（适配器+图片四路解析+防护）+ sequence（下载防护调用链）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Markdown 解析与图片防护架构图 | `markdown-parser-architecture.html` | architecture | showcase |
| 外部图片下载防护时序图 | `markdown-parser-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。

**省略说明**：未生成 dataflow/lifecycle 图——文档→块→IR 管道属域级 dataflow；图片下载预算降级是
分支判断而非单实体状态机，按资源节省原则省略。
