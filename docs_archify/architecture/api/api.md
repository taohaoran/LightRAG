# api 域（API 服务域）

> 本域是 LightRAG 的 HTTP 服务层：把 `LightRAG` 核心库包装成 FastAPI 服务，
> 提供 REST API、Ollama 兼容 API、WebUI 静态资源与认证准入。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`。

## 域职责

- **应用装配与进程模型**：FastAPI 工厂 `create_app`、生命周期、Uvicorn/Gunicorn 双启动路径、多 worker 并发限额。
- **路由表面**：文档管理（17 端点）、图谱读写（12 端点）、检索查询（3 端点）、Ollama 兼容（5 端点）、UI 定制（2 端点）。
- **安全横切**：JWT 认证、bcrypt 密码、登录限流、纯 ASGI 准入与请求体上限中间件。
- **配置面**：约 90 个 env/CLI 参数的启动期解析与 fail-fast 校验。

## 叶子索引

| 叶子 | 职责 | 文档 |
|---|---|---|
| L27 api-server-core | FastAPI 应用工厂、生命周期、Gunicorn/Uvicorn 进程模型、纯 ASGI 原语、运行时校验 | [api-server-core/api-server-core.md](api-server-core/api-server-core.md) |
| L28 api-routers | 5 个路由模块：文档/图谱/查询/Ollama 兼容/UI 定制；pipeline 忙预检 | [api-routers/api-routers.md](api-routers/api-routers.md) |
| L29 api-auth | JWT 认证、bcrypt、登录限流、准入票据、body 上限中间件 | [api-auth/api-auth.md](api-auth/api-auth.md) |
| L30 api-config | env/CLI 参数解析、四组启动校验、UI 定制 bundle 加载 | [api-config/api-config.md](api-config/api-config.md) |

## 域级机制细节

### 中间件栈顺序（后加者居外）

请求从外到内：`CORS` → `BodyLimit` → `Admission` → `RootPathNormalization` → 路由认证依赖。
拒绝类中间件在**读 body 之前**用裸 ASGI 作答，401/413/423 响应形状与 FastAPI HTTPException 一致。

### 启动 fail-fast 链

`parse_args` → 四组 validate_* → `create_app` 内 EMBEDDING_MODEL 检查 → 前端构建检查 → UI bundle 加载 → chunker/smart_heading 依赖校验 → LLM binding 合法性。任一失败即拒绝启动。

### Gunicorn 多 worker

`preload_app=True`：master 进程构建 app 后 fork；`LightRAGUvicornWorker` 带孤儿检测；跨 worker 并发限额把 `*_max_async` 同时作为本地限与全局信号量分组。

## 图表清单

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| api 域架构图 | `api-architecture.html` | architecture | showcase |
| 请求生命周期时序 | `api-sequence.html` | sequence | showcase |
| 请求处理数据流 | `api-dataflow.html` | dataflow | standard（竖向流 label 重叠，加 labelDy 后通过） |

域图省略 lifecycle：单实体状态机（文档状态 pending/processing/processed/failed）属于 pipeline 域，本域只做状态查询透传。省略 workflow：本域无多角色审批流。

JSON IR 位于 `json/` 目录。
