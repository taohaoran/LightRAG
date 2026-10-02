# llm 域总览

> 本文是 llm 域的域级总览，链接到本域全部叶子子系统文档。
> 输出根：`docs_archify/architecture/`。
> 源码基准：`lightrag-hku` main，commit `453dce83d6d0354a06e46c8d4029a0895c4e054b`。

## 域职责

llm 域是 LLM 与 embedding 的绑定/适配层：把各家模型服务统一为 `*_complete_if_cache` + `*_embed` 契约，供 core 域的角色注册表路由。它是纯编排/适配层，实际推理在外部服务。

关键机制（纯 Python 框架口径）：
- **统一契约**：每个 provider 暴露带缓存补全 + 嵌入，角色注册表按字符串选函数即完成路由。
- **配置驱动**：`BindingOptions` 子类把 env 解析为类型化选项。
- **目录即实例库**：11 家 provider 同构，深读代表 + 共性归纳。
- **缓存语义**：补全结果写入 LLM 响应缓存 KV。

## 叶子索引

| 叶子 | 职责 | 文档 |
|---|---|---|
| L24 llm-common | 包入口、错误判定、图像输入归一化、已弃用 provider | [llm-common.md](llm-common/llm-common.md) |
| L25 llm-openai-family | openai.py 主实现、azure/nvidia 变体、binding_options | [llm-openai-family.md](llm-openai-family/llm-openai-family.md) |
| L26 llm-provider-matrix | ollama/gemini 深读代表 + 其余 9 家共性归纳 | [llm-provider-matrix.md](llm-provider-matrix/llm-provider-matrix.md) |

## 域级机制细节

- **L24 ↔ L25/L26**：各 provider 复用 L24 的图像归一化与错误判定。
- **L25 ↔ L26**：同构契约，OpenAI 协议族为薄封装，其余 provider 独立实现。
- **llm 域 ↔ core**：core 域 L03 角色注册表从本域取函数。

## 域级图表

| 图 | 文件 | 类型 | 说明 |
|---|---|---|---|
| llm 域组件架构 | `llm-architecture.html` | architecture | 三叶子与外部服务边界 |
| 带缓存补全数据流 | `llm-dataflow.html` | dataflow | 角色→provider→缓存→远端 |
| 错误分类时序 | `llm-sequence.html` | sequence | 调用失败→可重试/永久 |

本域不补 lifecycle（provider 绑定无独有状态机）。JSON IR 位于 `json/`。
