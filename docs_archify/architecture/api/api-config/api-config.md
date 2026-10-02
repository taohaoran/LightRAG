# API 配置面（api-config）

> 本文是 `api` 域下的叶子子系统文档。域级总览见 `../api.md`。
> 本文展开 env/CLI 配置加载、校验链与 UI 定制 bundle 加载；
> 不展开应用如何消费配置（见 `../api-server-core/api-server-core.md`）。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 参数解析 | `parse_args()`：argparse 定义全部 CLI/env 参数（约 90 个），返回 Namespace | `lightrag/api/config.py:346` |
| 配置初始化 | `initialize_config(args, force)`：解析 → 四组校验 → 缓存为全局；幂等 | `config.py:1011` |
| 全局配置代理 | `_GlobalArgsProxy`：懒初始化代理，未 init 时自动 `initialize_config()` | `config.py:1064`、`:1115` `global_args` |
| 认证配置校验 | `validate_auth_configuration`：AUTH_ACCOUNTS 与 TOKEN_SECRET 一致性 | `config.py:166` |
| 扫描批处理校验 | `validate_scan_batch_configuration`：扫描批次参数合法性 | `config.py:177` |
| 准入配置校验 | `validate_admission_configuration`：MAX_PENDING_DOCUMENTS 与后端能力 | `config.py:205` |
| Bedrock 校验 | `validate_bedrock_auth_configuration`：Bedrock 凭据完整性 | `config.py:250` |
| binding 默认 host | `get_default_host(binding_type)`：按 LLM binding 返回默认 base_url | `config.py:78` |
| Embedding 前缀开关 | `resolve_asymmetric_embedding_opt_in` / `get_embedding_prefix_config`：区分对称/非对称 embedding binding 的前缀拼接 | `config.py:96`、`:148` |
| API 前缀规范化 | `normalize_api_prefix`：strip 首尾斜杠 | `config.py:322` |
| Uvicorn 模式强制单 worker | `update_uvicorn_mode_config`：uvicorn 模式下 workers>1 强制 1 | `config.py:995` |
| UI 定制 bundle 加载 | `load_ui_customization_snapshot(bundle_dir)`：读 manifest.json、校验键、加载本地化文案与 logo 资产 | `lightrag/api/ui_customization.py:647` |
| UI 定制解析入口 | `resolve_ui_customization_snapshot`：目录空→返回 None；损坏→`UICustomizationError` 启动失败 | `ui_customization.py:915` |
| SVG logo 安全校验 | `_has_svg_root`/`_sniff_logo_mime` 等：防止 SVG 内嵌脚本/外部引用 | `ui_customization.py:428`、`:478` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `parse_args()` | `config.py:346` | 定义全部 CLI 参数与 env 默认值 |
| `initialize_config()` | `config.py:1011` | 配置入口，幂等 |
| `_GlobalArgsProxy` | `config.py:1064` | 懒加载代理，`global_args` 实例 |
| `UICustomizationSnapshot` (frozen dataclass) | `ui_customization.py:557` | 不可变 UI 定制快照（bundle_revision/locales/assets） |
| `UILocaleContent` / `UIAsset` | `ui_customization.py:509`、`:501` | 单语言文案 / 单个静态资产 |
| `UICustomizationError` | `ui_customization.py:106` | bundle 损坏时抛出，导致启动失败 |
| `DefaultRAGStorageConfig` | `config.py:71` | 各存储后端默认参数 |

## 3. 关键调用链

### 3.1 启动配置加载

1. `main()`（`lightrag_server.py:3406`）→ `initialize_config()`（`config.py:1011`）。
2. 未传 args 时 `parse_args()`（`config.py:346`）从 sys.argv 与环境变量读入，生成 Namespace。
3. 顺序跑四个校验：`validate_auth_configuration`（`:166`）→ `validate_bedrock_auth_configuration`（`:250`）→ `validate_scan_batch_configuration`（`:177`）→ `validate_admission_configuration`（`:205`）；任一失败抛异常终止启动。
4. 缓存到 `_global_args`，`_initialized=True`；后续 `global_args.xxx` 经 `_GlobalArgsProxy` 透明访问。

### 3.2 UI 定制 bundle 加载

1. `create_app`（`lightrag_server.py:1486`）读 `args.ui_templates_dir`。
2. 非空则 `resolve_ui_customization_snapshot(dir)`（`ui_customization.py:915`）→ `load_ui_customization_snapshot`（`:647`）读 `manifest.json`。
3. 校验键集合（`:615`）、本地化文案（`:509`）、logo 资产（`:501`）；SVG 经 `_has_svg_root`/`_sniff_logo_mime`（`:428`/`:478`）防脚本注入。
4. 损坏 → `UICustomizationError`（`:106`）→ 启动 fail-fast；目录空 → 返回 None 静默用默认品牌。

### 3.3 消费配置

- 应用装配（L27）读 `global_args.embedding_model`/`llm_binding`/`working_dir` 等构造 LightRAG；
- 中间件（L29）读 `max_pending_documents`/`cors_origins`；
- 路由（L28）读 `top_k`。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 约 90 个 CLI/env 参数 | host/port/workers/working_dir/workspace、LLM/embedding binding+host+key+model、四类存储选择、top_k/chunk_size、CORS、AUTH_ACCOUNTS、TOKEN_SECRET 等 | `config.py:346` |
| `DEFAULT_TOKEN_SECRET` | 未配 AUTH_ACCOUNTS 时的 guest 模式兜底密钥（告警） | `config.py:65` |
| `LIGHTRAG_RUNTIME_TARGET` | host/compose/docker，与实际容器环境不符则拒绝（见 L27 runtime_validation） | `runtime_validation.py:82` |
| `UI_TEMPLATES_DIR` | UI 定制 bundle 目录；配置但损坏则启动失败 | `ui_customization.py:915` |
| `EMBEDDING_MODEL` | **必填**，空则 create_app 拒绝启动（见 L27） | `lightrag_server.py:1457` |

## 5. 错误与重试语义

- **配置校验失败**：启动期直接抛异常/SystemExit，不进入服务态；错误信息带修复提示（如 chunker 选择、spacy 依赖安装命令）。
- **UI bundle 损坏**：`UICustomizationError` 启动失败，不静默降级（运维以为客户品牌生效，实际是默认 LightRAG 品牌）。
- **目录空 bundle**：唯一豁免——返回 None，用内置品牌并打 warning 提示挂载未到位。
- **运行时无重试**：配置是启动期一次性决策，运行时不变更。

## 6. 并发细节

- **全局单例**：`_global_args` 在启动期单线程构造，之后只读；`_GlobalArgsProxy` 无锁读取。
- **多 worker**：gunicorn preload_app 在 master 进程加载配置后 fork，worker 继承同一份只读配置。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- CLI/env 参数定义、四组启动校验、UI 定制 bundle 加载与 SVG 安全校验

**Out-of-Scope（不在本仓库源码内）**
- `.env` 文件本身（部署产物）
- UI 定制 bundle 的具体内容（部署挂载，不在本仓库源码内）
- argparse/dotenv/pydantic 第三方库

## 8. 与相邻子系统交互

- 上游：CLI 入口（L27）与部署侧 `.env`。
- 本叶子 → 下游：
  - `global_args` 被 L27 装配 LightRAG、L29 中间件、L28 路由广泛读取；
  - `UICustomizationSnapshot` 传给 `create_ui_customization_routes`（L28）暴露给前端。

## 9. 语言专项适配口径（纯 Python 框架口径）

按 `language-python.md` 落写：

- **配置面工程**：约 90 个参数的 argparse Namespace 是本叶子的"dataclass 配置体系"等价物；配置在启动期一次性解析+校验，运行时只读。
- **配置驱动静态分派**：`llm_binding`/`kv_storage` 等字符串经 `get_storage_class`（storage 域）与 `create_llm_model_func`（L27）路由到具体后端——本叶子负责把 env 翻译成 Namespace，路由发生在消费方。
- **fail-fast 校验链**：四组 validate_* 函数是纯 Python 框架"宁可启动失败也不静默错误"风格的集中体现。
- **图型侧重**：architecture（配置组件分层）+ dataflow（env→argparse→校验→global_args→消费者 的配置管道）；无状态机。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 配置面架构图 | `api-config-architecture.html` | architecture | showcase |
| 配置加载数据流 | `api-config-dataflow.html` | dataflow | standard（showcase 报 label 重叠与 stage 宽度，加宽 viewBox 后 standard 通过） |

本叶子未补 sequence：配置加载是一次性启动流程，非多请求方消息交互；未补 lifecycle：无单实体状态机。

JSON IR 源文件位于 `json/` 目录。
