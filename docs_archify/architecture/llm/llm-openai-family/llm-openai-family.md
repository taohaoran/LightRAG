# OpenAI 家族 LLM 绑定（llm-openai-family）

> 本文是 `llm` 域下的叶子子系统文档。域级总览见 `../llm.md`。
> 本文展开 OpenAI 兼容协议族绑定：openai.py 主实现、azure_openai、nvidia_openai 与 binding_options；
> 其余 provider（ollama/gemini/anthropic 等）见 `../llm-provider-matrix/llm-provider-matrix.md`。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 带缓存补全 | `openai_complete_if_cache` 核心：查缓存→调模型→写缓存 | `lightrag/llm/openai.py:305` |
| OpenAI 补全 | 同步包装与 gpt-4o/gpt-4o-mini 便捷函数 | `openai.py:937/961/985` |
| OpenAI 嵌入 | `openai_embed` 向量嵌入 | `openai.py:1054` |
| 异步客户端工厂 | `create_openai_async_client` 构造带超时/重试的客户端 | `openai.py:177` |
| Azure 变体 | `azure_openai_complete_if_cache` / embed | `openai.py:1216/1307` |
| NVIDIA 变体 | `nvidia_openai_complete` / embed | `openai.py:1009`、`nvidia_openai.py:45` |
| 可重试错误判定 | transient status / retryable rate-limit | `openai.py:109/117` |
| 绑定选项 | 各 provider env 选项类（OpenAILLMOptions 等） | `lightrag/llm/binding_options.py:132/719` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `openai_complete_if_cache` | `openai.py:305` | 带 LLM 缓存的补全核心 |
| `create_openai_async_client` | `openai.py:177` | OpenAI 异步客户端工厂 |
| `BindingOptions` | `binding_options.py:132` | env 选项解析基类 |
| `OpenAILLMOptions` | `binding_options.py:719` | OpenAI 绑定选项 |
| `InvalidResponseError` / `TransientBadRequestError` | `openai.py:80/86` | 错误类型 |

## 3. 关键调用链

**调用链 A：带缓存补全（`openai.py:305` `openai_complete_if_cache`）**
1. 查 LLM 响应缓存（hashing_kv），命中直接返回。
2. `create_openai_async_client`（`openai.py:177`）构造客户端。
3. 调 OpenAI chat completions，支持流式。
4. 写缓存返回。

**调用链 B：错误重试（`openai.py:109/117`）**
- `_is_transient_status_error` / `_is_retryable_rate_limit_error` 判定可重试性，配合客户端退避。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `OPENAI_API_KEY` / `OPENAI_BASE_URL` / `OPENAI_MODEL` | OpenAI 连接 | env.example |
| 绑定选项 | `OpenAILLMOptions` 从 env 解析 | `binding_options.py:719` |
| 超时/重试 | 客户端工厂注入 | `openai.py:177` |

## 5. 错误与重试语义

- **瞬时错误**：`_is_transient_status_error` 命中走客户端退避重试。
- **可重试限流**：`_is_retryable_rate_limit_error` 区分永久/可重试限流。
- **响应格式校验**：`_validate_openai_response_format`（`openai.py:133`）提前拒绝非法 response_format。

## 6. 并发细节

- 异步客户端单次调用；并发由外层角色队列（L03）控制。
- tiktoken 编码按模型缓存（`_get_tiktoken_encoding_for_model`，`openai.py:157`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/llm/openai.py`、`azure_openai.py`、`nvidia_openai.py`、`binding_options.py`

**Out-of-Scope（不在本仓库源码内）**
- OpenAI/Azure/NVIDIA 远端服务实际推理
- openai 官方 SDK 本身（外部依赖）

## 8. 与相邻子系统交互

- 上游：L03 llm-roles 把本叶子函数注册为 extract/query 等角色
- 下游：本叶子 → OpenAI 远端服务
- 横切：L24 llm-common 的图像归一化与错误判定

## 9. 语言专项适配口径（纯 Python 框架）

- **配置驱动绑定**：`BindingOptions` 子类把 env 解析为类型化选项，是框架级配置缝。
- **双轨变体**：OpenAI/Azure/NVIDIA 共享 OpenAI 协议，以薄封装区分 base_url 与认证。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| OpenAI 家族绑定架构 | `llm-openai-family-architecture.html` | architecture | standard（"构造"标签压核心；已加 labelDy） |
| 带缓存补全时序 | `llm-openai-family-sequence.html` | sequence | standard（末消息 y 越界；已压缩时间轴） |

本叶子不补 dataflow/lifecycle（绑定层无数据管道与状态机）。JSON IR 位于 `json/`。两图均 standard 渲染通过、HTML >799KB。
