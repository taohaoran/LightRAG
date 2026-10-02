# LLM 公共工具（llm-common）

> 本文是 `llm` 域下的叶子子系统文档。域级总览见 `../llm.md`。
> 本文展开 llm 包的公共辅助：错误判定、图像输入归一化、已弃用 provider；
> OpenAI 家族 provider 见 `../llm-openai-family/llm-openai-family.md`，其余 provider 矩阵见 `../llm-provider-matrix/llm-provider-matrix.md`。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 包入口 | `lightrag/llm/__init__.py` 聚合导出各 provider 绑定函数 | `lightrag/llm/__init__.py` |
| 永久限流错误判定 | 识别不可重试的 rate-limit 错误 | `lightrag/llm/_error_utils.py:34` `is_permanent_rate_limit_error` |
| 图像输入归一化 | 把 URL/base64/bytes/路径归一为 `NormalizedImage` | `lightrag/llm/_vision_utils.py:93` `normalize_image_inputs` |
| 图像维度读取 | PNG/JPEG/GIF/WebP 维度探测 | `_vision_utils.py:264` `read_image_dimensions` |
| 图像缓存/审计元数据 | 生成图像缓存与审计元数据 | `_vision_utils.py:133/155` |
| 已弃用 provider | siliconcloud（保留兼容） | `lightrag/llm/deprecated/siliconcloud.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `NormalizedImage` | `_vision_utils.py:42` | 归一化图像数据类 |
| `normalize_image_inputs` | `_vision_utils.py:93` | 多格式图像输入归一 |
| `is_permanent_rate_limit_error` | `_error_utils.py:34` | 限流错误可重试性判定 |

## 3. 关键调用链

**调用链 A：图像归一（`_vision_utils.py:93`）**
1. `_coerce_item`（`_vision_utils.py:78`）把输入项归一为 dict。
2. `_decode_base64` / `_detect_mime` 解析为 bytes 与 MIME。
3. `read_image_dimensions`（`_vision_utils.py:264`）按格式读维度，产出 `NormalizedImage`。

**调用链 B：错误分类（`_error_utils.py:34`）**
- provider 调用失败后，`is_permanent_rate_limit_error` 判定是否永久限流（不重试）vs 可重试。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 无独立配置 | 本叶子为公共工具，配置在各 provider | — |

## 5. 错误与重试语义

- **永久限流**：`is_permanent_rate_limit_error` 返回 True 的错误不进入重试退避，直接上抛。
- **图像解码失败**：归一化失败时跳过该图像，不阻断整批。

## 6. 并发细节

- 本叶子无自有并发原语；被各 provider 函数在调用 LLM 前同步调用。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/llm/__init__.py`、`_error_utils.py`、`_vision_utils.py`、`deprecated/`

**Out-of-Scope（不在本仓库源码内）**
- 各 LLM 服务实际 HTTP 调用——见 L25/L26
- 图像文件本身——外部输入

## 8. 与相邻子系统交互

- 上游：各 provider 函数（L25/L26）调用本叶子的归一化与错误判定
- 下游：归一化图像喂入 VLM 角色调用

## 9. 语言专项适配口径（纯 Python 框架）

- **公共工具缝**：跨 provider 复用的纯函数集中于此，避免各 provider 重复实现。
- **可选依赖**：图像维度读取对未知格式返回 None 而非抛错，优雅降级。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| LLM 公共工具架构 | `llm-common-architecture.html` | architecture | standard（一次渲染通过） |
| 图像归一化数据流 | `llm-common-dataflow.html` | dataflow | standard（一次渲染通过） |

本叶子不补 sequence/lifecycle（公共工具无多方交互与状态机）。JSON IR 位于 `json/`。两图均 standard 渲染通过、HTML >797KB。
