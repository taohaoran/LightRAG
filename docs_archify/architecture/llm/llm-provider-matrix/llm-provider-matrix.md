# 多 provider LLM 矩阵（llm-provider-matrix）

> 本文是 `llm` 域下的叶子子系统文档。域级总览见 `../llm.md`。
> 本文展开 OpenAI 兼容协议族之外的其余 LLM/embedding provider 绑定矩阵；
> OpenAI 家族见 `../llm-openai-family/llm-openai-family.md`，公共工具见 `../llm-common/llm-common.md`。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 1. 功能清单（目录即实例库，深读 ollama/gemini，其余共性）

每个 provider 文件都遵循同一契约：`*_complete_if_cache`（带缓存补全）+ `*_model_complete`（便捷）+ `*_embed`（嵌入）。

| Provider | 代表函数 | 源码路径 | 深度 |
|---|---|---|---|
| Ollama | `_ollama_model_if_cache`/`ollama_model_complete`/`ollama_embed` | `lightrag/llm/ollama.py:209/453/491` | 深读 |
| Gemini | `gemini_complete_if_cache`/`gemini_model_complete` | `lightrag/llm/gemini.py:312/676` | 深读 |
| LMDeploy | lmdeploy 补全 | `lightrag/llm/lmdeploy.py` | 共性 |
| LlamaCpp | lollms 补全 | `lightrag/llm/lollms.py` | 共性 |
| HuggingFace | hf 补全/嵌入 | `lightrag/llm/hf.py` | 共性 |
| Anthropic | claude 补全 | `lightrag/llm/anthropic.py` | 共性 |
| Bedrock | AWS Bedrock 补全 | `lightrag/llm/bedrock.py` | 共性 |
| VoyageAI | voyage 嵌入 | `lightrag/llm/voyageai.py` | 共性 |
| Jina | jina 补全 | `lightrag/llm/jina.py` | 共性 |
| 智谱 Zhipu | zhipu 补全/嵌入 | `lightrag/llm/zhipu.py` | 共性 |
| LlamaIndex | llama_index_impl 适配 | `lightrag/llm/llama_index_impl.py` | 共性 |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `_ollama_model_if_cache` | `ollama.py:209` | Ollama 带缓存补全核心 |
| `_get_gemini_client` | `gemini.py:106` | Gemini 客户端工厂 |
| `_build_generation_config` | `gemini.py:174` | Gemini 生成配置 |
| `BindingOptions` 子类 | `binding_options.py` | 各 provider env 选项 |

## 3. 关键调用链

**调用链 A：Ollama 补全（`ollama.py:209` `_ollama_model_if_cache`）**
1. `ensure_think_supported`（`ollama.py:70`）校验思考模式。
2. 查 LLM 缓存，未命中调 ollama 服务。
3. `_ollama_usage_counts`（`ollama.py:170`）提取用量。

**调用链 B：Gemini 补全（`gemini.py:312` `gemini_complete_if_cache`）**
1. `_get_gemini_client`（`gemini.py:106`）建客户端。
2. `_build_generation_config`（`gemini.py:174`）构生成配置。
3. 调模型，`_extract_response_text`（`gemini.py:255`）取文本。

## 4. 配置项

| 配置项 | 行为 | 位置 |
|---|---|---|
| `OLLAMA_HOST` / `OLLAMA_MODEL` | Ollama 连接 | env.example |
| `GEMINI_API_KEY` | Gemini 认证 | `gemini.py:158` |
| 各 provider env | 经 BindingOptions 解析 | `binding_options.py` |

## 5. 错误与重试语义

- **Gemini 可重试错误**：`_is_retryable_gemini_client_error`（`gemini.py:58`）判定退避。
- 各 provider 对远端错误做各自的重试包装，错误语义与 OpenAI 家族对齐。

## 6. 并发细节

- 与 OpenAI 家族一致：异步客户端单调用，并发由角色队列（L03）控制。
- 无跨 provider 共享状态。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `lightrag/llm/` 下各 provider 绑定文件

**Out-of-Scope（不在本仓库源码内）**
- Ollama/Gemini/Anthropic/Bedrock 等远端服务实际推理
- 各 provider SDK 本身（外部依赖）

## 8. 与相邻子系统交互

- 上游：L03 llm-roles 按配置把某 provider 函数绑定到角色
- 下游：本叶子 → 各 provider 远端服务
- 横切：L24 公共工具、L25 OpenAI 家族共享缓存语义

## 9. 语言专项适配口径（纯 Python 框架）

- **"目录即实例库"**：11 个 provider 文件 = 同一契约的实例库而非各异实现，无法穷举；本叶子深读 ollama/gemini 两代表，其余以共性表格说明。
- **统一契约**：所有 provider 暴露 `*_complete_if_cache` + `*_embed`，是框架级"能力缝"——角色注册表（L03）按字符串选 provider 函数即完成路由。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| provider 矩阵路由架构 | `llm-provider-matrix-architecture.html` | architecture | standard（一次渲染通过） |
| 角色到 provider 数据流 | `llm-provider-matrix-dataflow.html` | dataflow | standard（一次渲染通过） |

本叶子不补 sequence/lifecycle（矩阵绑定无独有多方交互与状态机；调用时序与 OpenAI 家族一致）。
**未深读清单（如实披露）**：lmdeploy/lollms/hf/anthropic/bedrock/voyageai/jina/zhipu/llama_index_impl 仅以共性契约归纳，未逐函数精读，因其与 ollama/gemini 同构。JSON IR 位于 `json/`。两图均 standard 渲染通过、HTML >798KB。
