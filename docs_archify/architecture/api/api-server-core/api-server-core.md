# API 服务端核心（api-server-core）

> 本文是 `api` 域下的叶子子系统文档。域级总览见 `../api.md`。
> 本文展开 FastAPI 应用组装、生命周期、Gunicorn 多 worker 进程模型与运行时校验；
> 不展开 5 个路由模块（见 `../api-routers/api-routers.md`）、认证鉴权链（见 `../api-auth/api-auth.md`）
> 与 env 配置面（见 `../api-config/api-config.md`）。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| FastAPI 应用工厂 | `create_app(args)`：装配 FastAPI 实例、异常处理器、中间件链、路由、静态资源挂载；fail-fast 启动校验（embedding model 名、前端构建、UI 定制 bundle、chunker、smart_heading 依赖） | `lightrag/api/lightrag_server.py:1436` |
| 生命周期管理 | `lifespan` 异步上下文：启动调 `rag.initialize_storages()` + `check_and_migrate_data()` + 准入容量探针；关闭先取消并 join 后台任务，再 `finalize_storages()` | `lightrag/api/lightrag_server.py:1640` |
| LLM/Embedding 函数工厂 | `create_llm_model_func(binding)` 按 binding 字符串路由到 openai/azure/gemini/ollama/bedrock/lollms；OpenAI 系走预配置缓存的 optimized 闭包，其他 lazy import | `lightrag/api/lightrag_server.py:2005` |
| LLM 配置缓存 | `LLMConfigCache`：启动时一次性解析 provider options，避免每请求重复解析 | `lightrag/api/lightrag_server.py:560` |
| LightRAG 实例组装 | 以约 40 个构造参数实例化 `LightRAG(...)`，并 `register_role_llm_builder` 注入角色级 LLM 重建钩子 | `lightrag/api/lightrag_server.py:2513`、`:2585` |
| 中间件装配 | 按添加顺序（后加者居外层）：`_RootPathNormalizationMiddleware` → `AdmissionMiddleware` → `BodyLimitMiddleware` → `CORSMiddleware` | `lightrag/api/lightrag_server.py:1796-1856` |
| WebUI 静态资源挂载 | `SmartStaticFiles` 子类化 StaticFiles，做哈希资源强缓存 + index.html 不缓存 + 运行时配置注入；`/webui` 固定挂载点，`/workspace` 可选 | `lightrag/api/lightrag_server.py:3111`、`:3231` |
| Swagger UI 离线化 | 自定义 `/docs` 端点指向打包在 `static/swagger-ui/` 的本地静态资源，支持 theme 注入 | `lightrag/api/lightrag_server.py:2612` |
| Uvicorn 单进程入口 | `main()`：Windows 下切 SelectorEventLoop、`initialize_config()`、`check_env_file()`、日志配置后 `uvicorn.run(app)` | `lightrag/api/lightrag_server.py:3406` |
| Gunicorn 多 worker 入口 | `run_with_gunicorn.py::main()`：设置 `LIGHTRAG_GUNICORN_MODE=1`、macOS fork-safety 校验、构造跨 worker 全局并发限额、派 gunicorn 进程 | `lightrag/api/run_with_gunicorn.py:62` |
| Gunicorn 配置 | `preload_app=True` + 自定义 worker 类；`on_exit` 在 master 进程 `finalize_share_data()`；`post_fork` 重置各 worker 日志 | `lightrag/api/gunicorn_config.py:32`、`:125`、`:141` |
| 孤儿 worker 检测 | `LightRAGUvicornWorker` 继承 UvicornWorker，在 `callback_notify` 节拍检查 ppid 变化，master 死亡时自投 SIGTERM；并把 `graceful_timeout` 转发给 uvicorn | `lightrag/api/gunicorn_worker.py:42` |
| 纯 ASGI 中间件原语 | `header_value`/`bearer_token`/`send_json`/`limited_receive`：在请求体读取前作答的中间件共用的裸 ASGI 工具，错误响应形状与 FastAPI HTTPException 一致 | `lightrag/api/asgi_helpers.py:16` |
| 运行时环境检测 | `detect_runtime_environment()`：cgroup + `/proc/1/cgroup` + `.dockerenv` + K8s serviceaccount 判定 host/Docker/K8s；`validate_runtime_target` 校验 `.env` 声明与实际环境一致 | `lightrag/api/runtime_validation.py:42`、`:82` |
| 错误 ID 与 500 脱敏 | `internal_server_error()` 生成带 error_id 的 HTTPException；全局 `unhandled_exception_handler` 保证原始异常文本不回客户端（CWE-209） | `lightrag/api/utils_api.py:52`；`lightrag_server.py:1764` |
| Token 续期 | `_renew_token_if_needed` 在响应头写 `X-New-Token`，浏览器跨域读取需 CORS expose_headers | `lightrag/api/utils_api.py:140` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `create_app(args)` | `lightrag_server.py:1436` | 应用工厂，返回装配完成的 FastAPI 实例；同时是启动 fail-fast 的集中地 |
| `lifespan(app)` | `lightrag_server.py:1640` | 异步上下文管理器：存储初始化→迁移→准入探针→yield→关闭任务→finalize |
| `LLMConfigCache` | `lightrag_server.py:560` | 启动期解析 LLM provider options 为缓存 dict |
| `LightRAGUvicornWorker` | `gunicorn_worker.py:42` | 带孤儿检测的 uvicorn worker；`exit_if_orphaned()` 可单测 |
| `BodyLimitExceeded` | `asgi_helpers.py:64` | 在被包装的 `receive()` 内抛出，由中间件捕获，绝不泄漏到路由 |
| `limited_receive(receive, limit)` | `asgi_helpers.py:75` | O(1) 内存包裹 receive，逐 chunk 计数，超限即抛 |
| `RuntimeEnvironment` (frozen dataclass) | `runtime_validation.py:15` | host/docker/k8s 三态不可变描述 |
| `_RootPathNormalizationMiddleware` | `lightrag_server.py:427` | 反向代理 strip 路径后规范化 `scope["path"]` |
| `SmartStaticFiles` | `lightrag_server.py:3112` | 子类化 StaticFiles：哈希资源强缓存、index.html 不缓存、注入运行时配置 |
| `get_combined_auth_dependency(api_key)` | `utils_api.py:392` | 生成认证 FastAPI 依赖（详见 api-auth 叶子） |

## 3. 关键调用链

### 3.1 启动装配链（uvicorn 路径）

1. `__main__` → `main()`（`lightrag_server.py:3406`）：Windows 下 `WindowsSelectorEventLoopPolicy` → `initialize_config()` 加载 `.env` 为 `global_args`。
2. `check_env_file()`（`utils_api.py:225`）校验 `.env` 存在性与 `LIGHTRAG_RUNTIME_TARGET` 与实际容器环境一致（`runtime_validation.py:121`）。
3. `create_app(global_args)`（`lightrag_server.py:1436`）：先 fail-fast 检查 `EMBEDDING_MODEL` 非空（`:1457`），再 `check_frontend_build()`（`:1146`）、解析 UI 定制 bundle（`:1495`）、`load_third_party_parsers()`（`:1544`）、`load_and_resolve_chunker`（`:1551`）、`validate_smart_heading_dependencies`（`:1571`）。
4. 装配中间件（`:1796-1856`），构造 `LightRAG` 实例（`:2513`），`include_router` 五个路由（`:2595-2608`），挂载 Swagger 静态目录与 WebUI 静态目录（`:3231`、`:3243`）。
5. 返回 app 后 `uvicorn.run(app=app, host, port)`（`:3474`）；首个请求进入时触发 `lifespan` 启动段（`:1641`）完成存储初始化。

### 3.2 Gunicorn 多 worker 启动链

1. `lightrag-gunicorn` → `run_with_gunicorn.py::main()`（`:62`）：`initialize_config()` → 置 `LIGHTRAG_GUNICORN_MODE=1`（`:76`）→ macOS 多 worker 校验 `OBJC_DISABLE_INITIALIZE_FORK_SAFETY`（`:83`）。
2. `_build_global_concurrency_limits(args)`（`:32`）把每个 role 的 `*_llm_max_async` 映射为跨 worker 信号量分组 `llm:{role}`/`embedding`/`rerank`。
3. 派 gunicorn 进程，使用 `gunicorn_config.py`：`preload_app=True`（`:32`）先 import 并 `get_application()` 在 master 进程构建 app，再 fork workers；worker 类为 `LightRAGUvicornWorker`（`:36`）。
4. worker fork 后 `post_fork`（`gunicorn_config.py:141`）重置日志；每个 worker 各自跑 `lifespan` 启动段初始化自己的存储连接。
5. master 退出时 `on_exit`（`gunicorn_config.py:125`）调 `finalize_share_data()`；worker 端 lifespan 因 `LIGHTRAG_GUNICORN_MODE` 存在而跳过共享数据 finalize（`lightrag_server.py:1690`）。

### 3.3 单请求穿过中间件栈

1. 请求进入后最外层是 `CORSMiddleware`（最后 add，故最外）→ `BodyLimitMiddleware` → `AdmissionMiddleware` → `_RootPathNormalizationMiddleware`（若有 prefix）。
2. 每个中间件在**读 body 之前**用裸 ASGI 原语（`asgi_helpers.py:38` `send_json`）直接作答——401/413/423 响应形状与 FastAPI HTTPException 完全一致，客户端无法区分拒绝来自中间件还是路由。
3. body 超过上限时 `limited_receive`（`asgi_helpers.py:75`）在流式 chunk 计数中抛 `BodyLimitExceeded`，由 BodyLimitMiddleware 转 413；O(1) 内存，不缓冲。
4. 过中间件后进入路由层认证依赖（`Depends(get_combined_auth_dependency)`），再到具体路由处理函数，最终调到 `rag.aquery` / `rag.ainsert` 等核心方法（详见 api-routers 叶子）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `EMBEDDING_MODEL` | **必填**；空则 `SystemExit` 拒绝启动（向量模型名是维度外换模型的唯一检测标记） | `lightrag_server.py:1457` |
| `EMBEDDING_DIM` | 配合 EMBEDDING_DIM 显式声明非默认维度 | 同上注释 |
| `HOST` / `PORT` | uvicorn 监听地址 | `config.py`（L30） |
| `WORKERS` | gunicorn worker 数；>1 时 macOS 强制要求 `OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES` | `run_with_gunicorn.py:83` |
| `PRELOAD_APP` | 恒为 True（gunicorn_config.py:32）：master 先构建 app 再 fork | `gunicorn_config.py:32` |
| `LIGHTRAG_GUNICORN_MODE` | run_with_gunicorn 自置；lifespan 据此跳过共享数据 finalize | `lightrag_server.py:1690` |
| `MAX_PENDING_DOCUMENTS` | >0 时安装 AdmissionMiddleware 并在启动时探测 doc_status 严格计数能力 | `lightrag_server.py:1811`、`:1657` |
| `ENABLE_API_DOCS` | 单开关控制 /docs、/redoc、/openapi.json、/static/swagger-ui 五处 | `lightrag_server.py:1708` |
| `API_PREFIX` | 反代 strip 前缀；触发 `_RootPathNormalizationMiddleware` | `lightrag_server.py:1796` |
| `CORS_ORIGINS` | `*` 时不发 Allow-Credentials（Fetch 规范）；空/逗号尾 fail-closed 为不允许跨域 | `lightrag_server.py:1777` |
| `LIGHTRAG_RUNTIME_TARGET` | host/compose/docker；与实际容器环境不符则拒绝启动 | `runtime_validation.py:82` |
| `LOG_DIR` / `LOG_MAX_BYTES` / `LOG_BACKUP_COUNT` | 轮转日志文件位置与大小 | `lightrag_server.py:3319` |
| `UI_TEMPLATES_DIR` | UI 定制 bundle 目录；配置但损坏则启动失败（fail-fast，不静默降级） | `lightrag_server.py:1486` |

## 5. 错误与重试语义

- **启动即失败（fail-fast）**：embedding model 缺失、chunker 非法、smart_heading 依赖缺失、UI bundle 损坏、runtime target 与实际环境不符——全部在 `create_app`/`main` 阶段直接 `SystemExit`/`sys.exit(1)`/`raise`，不进入服务态。
- **未捕获异常兜底**：`@app.exception_handler(Exception)`（`lightrag_server.py:1764`）生成 12 位 error_id，完整 traceback 落服务端日志，客户端只收到 `Internal server error (error_id: ...)`，杜绝 CWE-209。
- **参数校验**：`validation_exception_handler`（`:1395`）按端点形状格式化 422。
- **准入探针失败**：`MAX_PENDING_DOCUMENTS` 开启但 doc_status 后端不能严格计数时，启动即 `RuntimeError`（`:1663`），而非运行时每次上传 503。
- **Gunicorn 孤儿 worker**：master 被 `kill -9` 后，worker 在下一个 notify 节拍（约 `timeout/2` 秒）自查 ppid，自投 SIGTERM 优雅退出，不再占监听端口（`gunicorn_worker.py:53`）。
- **关闭时后台任务**：`drain_reserved_background_tasks` 先取消并 join 预留后台任务，再 finalize 存储；关闭期产生的取消异常在清理完成后才 re-raise（`lightrag_server.py:1683`）。

## 6. 并发细节

- **进程模型**：uvicorn 单进程；gunicorn `preload_app=True` 多 worker——master 构建一次 app 后 fork，worker 各自持有独立事件循环与存储连接；共享数据（`shared_storage`）的 finalize 只在 master `on_exit` 执行一次。
- **跨 worker 并发限额**：`_build_global_concurrency_limits`（`run_with_gunicorn.py:32`）把 `*_max_async` 同时作为 worker 本地限与跨 worker 全局信号量分组，避免实际并发 = MAX_ASYNC × workers。
- **后台任务集**：`app.state.background_tasks: set`（`:1644`）持有预留任务引用，关闭时统一取消 join；防止任务在共享状态销毁后才释放其 reservation。
- **中间件无共享可变状态**：AdmissionMiddleware 经 `rag_getter` 懒闭包访问后续才构造的 `rag`；BodyLimitMiddleware 的 `total` 计数是每请求闭包内局部变量。
- **事件循环**：Windows 强制 SelectorEventLoop 规避 ProactorEventLoop 与 uvicorn 端口绑定的竞态（issue #2438）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- FastAPI 应用工厂、生命周期、中间件装配、静态资源挂载、Swagger 离线化
- Uvicorn 单进程与 Gunicorn 多 worker 两套启动入口及配套配置/worker 类
- 纯 ASGI 中间件原语、运行时容器环境检测、错误 ID 与 token 续期辅助

**Out-of-Scope（不在本仓库源码内）**
- Uvicorn / Gunicorn / Starlette / FastAPI 框架本身（第三方依赖，`不在本仓库源码内`）
- LLM/Embedding 远程推理服务（OpenAI/Ollama/Gemini 等远端 HTTP 服务，`不在本仓库源码内`）
- 五类存储后端的实际数据面（见 storage 域叶子）
- 前端 React 产物（由 `lightrag_webui/` 构建后被 SmartStaticFiles 挂载，见 webui 域）

## 8. 与相邻子系统交互

- 上游：CLI 入口 `lightrag-server` / `lightrag-gunicorn`（pyproject entry point）→ 本叶子。
- 本叶子 → 下游：
  - `create_app` 构造 `LightRAG` 实例（core 域 L01）注入五个路由；
  - 中间件链依赖 api-auth（L29）提供的认证依赖与 admission 原语；
  - 配置面来自 api-config（L30）的 `global_args`；
  - 路由层见 api-routers（L28）。
- 横向：SmartStaticFiles 服务 webui 域构建产物；Ollama 兼容路由见 L28。

## 9. 语言专项适配口径（纯 Python 框架口径）

按 `language-python.md` 落写：

- **编排层定位**：本叶子不承载检索/抽取计算，实际计算在 `LightRAG` 核心与远端 LLM 服务；仓库价值 = 把配置对象翻译成可运行的 FastAPI 服务实例。外部计算方（uvicorn/gunicorn/远端 LLM）均标注"不在本仓库源码内"。
- **配置驱动静态分派**：`create_llm_model_func(binding)`（`:2005`）按 `args.llm_binding` 字符串路由到 6 种 LLM 函数构造器；同一服务代码经不同 binding 配置切换后端——是纯 Python 框架的静态多态核心，已在架构图显式表达。Embedding binding 同理（8 种）。
- **注册表与工厂**：路由以 `create_document_routes(rag, ...)` 等工厂函数注入 rag 依赖，而非全局 router 模块；`register_role_llm_builder`（`:2585`）把角色级 LLM 重建钩子回调注册进 LightRAG。
- **可选依赖与懒加载**：每个 binding 的 import 放在分支内（lazy import），顶层不加载全部 provider；`psutil` 缺失仅跳过内存上报不致命。
- **错误即控制流**：fail-fast 启动校验大量使用 `raise`/`SystemExit`/`sys.exit(1)` 而非配置降级——与框架"宁可拒绝启动也不静默错误"的一致风格。
- **图型侧重**：architecture（分层 + binding 路由 + 外部边界）+ sequence（请求穿过中间件栈的单次调用链）；本叶子无独立管道数据流（数据管道属于 pipeline 域）、无单实体状态机（文档状态机见 pipeline 域），故不补 dataflow/lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 服务端核心架构图 | `api-server-core-architecture.html` | architecture | showcase |
| 请求穿过中间件栈时序 | `api-server-core-sequence.html` | sequence | showcase |

本叶子未补 dataflow 图：请求/响应的数据变换属于各路由（L28）与核心检索（L02）职责，本叶子只做装配与中间件旁路，无独立管道语义。未补 lifecycle：单实体状态机（文档状态）在 pipeline 域，本叶子的启动/关闭只是 lifespan 两个相位，不值得单独状态机图。

JSON IR 源文件位于 `json/` 目录。
