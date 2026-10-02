# API 认证与准入（api-auth）

> 本文是 `api` 域下的叶子子系统文档。域级总览见 `../api.md`。
> 本文展开 JWT 认证、密码哈希、登录限流、请求体上限与摄入准入控制；
> 不展开应用装配（见 `../api-server-core/api-server-core.md`）与路由端点（见 `../api-routers/api-routers.md`）。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| JWT 认证处理器 | `AuthHandler`：解析 `AUTH_ACCOUNTS`、签发/校验 JWT、密码校验；模块级单例 `auth_handler` | `lightrag/api/auth.py:50`、`:263` |
| 密码哈希 | `hash_password`/`verify_password`：`{bcrypt}` 前缀走 bcrypt，明文走 `hmac.compare_digest` 常量时间比较 | `lightrag/api/passwords.py:8`、`:15` |
| 登录时序攻击缓解 | 未知用户名/明文账户也强制跑一次 dummy bcrypt，使响应时间不暴露账户是否存在（CWE-208） | `auth.py:29` `_DUMMY_VERIFY_SPEC`、`:116` |
| JWT 声明校验 | `validate_token`：显式排除 `none` 算法、限制 `sub` 长度 256、校验 `exp` 范围、过期拒绝 | `auth.py:185`、`:40` `MAX_TOKEN_SUBJECT_LENGTH` |
| 登录暴力破解限流 | `LoginRateLimiter`：滑动窗口 + pending/confirmed 双状态，按 IP+username 限流，防 TOCTOU 并发绕过 | `lightrag/api/login_rate_limit.py:69` |
| 纯 ASGI 准入中间件 | `AdmissionMiddleware`：在**读 body 之前**完成认证 + 容量预留，拒绝时不传输 body（连 100-continue 都不发） | `lightrag/api/admission_middleware.py:52` |
| 准入票据交接 | `AdmissionTicket`：跨三层（中间件→路由→后台任务）交接一次容量预留，任一时刻单所有者 | `lightrag/api/admission.py:38` |
| 请求体上限中间件 | `BodyLimitMiddleware`：逐 chunk 计数 body 字节数，超限抛 `BodyLimitExceeded` 转 413，O(1) 内存 | `lightrag/api/body_limit_middleware.py`（见 L27 asgi_helpers） |
| 对话输入计量 | `count_conversation_input_chars`：统一统计 query+instruction+history JSON 序列化后的字符数，供输入预算 | `lightrag/api/input_limits.py:10` |
| 组合认证依赖工厂 | `get_combined_auth_dependency(api_key)` 生成 FastAPI 依赖；`get_auth_status_dependency` 非强制版供 /health | `lightrag/api/utils_api.py:392`、`:546` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AuthHandler` | `auth.py:50` | JWT 签发/校验 + 账户解析 + 密码校验；模块单例 `auth_handler` |
| `TokenPayload` | `auth.py:43` | JWT 载荷 Pydantic 模型：sub/exp/role/metadata |
| `verify_password` | `passwords.py:15` | bcrypt 或常量时间明文比较 |
| `LoginRateLimiter` | `login_rate_limit.py:69` | 滑动窗口限流；`reserve_attempt`/`commit_failure`/`reset`/`release` 四阶段 |
| `AdmissionMiddleware` | `admission_middleware.py:52` | 纯 ASGI：预认证 + 容量预留 + 票据发布到 scope["state"] |
| `AdmissionTicket` | `admission.py:38` | 数据类：token/weight/adopted；`adopt_admission_ticket` 路由接管 |
| `ADMISSION_PATHS` | `admission_middleware.py:45` | 受控摄入路径元组：upload/text/texts（scan 故意豁免） |
| `_DUMMY_VERIFY_SPEC` | `auth.py:29` | 恒不匹配的 bcrypt 规格，用于均衡登录时序 |

## 3. 关键调用链

### 3.1 登录 `/login`

1. 客户端 `POST /login`（`lightrag_server.py:2767`）→ `LoginRateLimiter.reserve_attempt` 在 bcrypt 前登记一个 pending 名额（`login_rate_limit.py:69` 文档）。
2. 路由调 `auth_handler.verify_password(username, password)`（`auth.py:116`）：
   - 已知 bcrypt 账户：一次真实 bcrypt；
   - 已知明文账户：真实常量时间比较 + 一次 dummy bcrypt 对齐耗时；
   - 未知用户：一次 dummy bcrypt 后返回 False。
3. 成功 → `auth_handler.create_token`（`auth.py:148`）按 role 选过期时长（guest 单独配置）签 JWT；`commit_failure`/`reset` 更新限流状态，`release` 删 pending。
4. 失败 → `commit_failure` 记录时间戳；超窗累计超限返回 429。

### 3.2 已认证请求携带 JWT

1. 请求头 `Authorization: Bearer <jwt>` → FastAPI 依赖 `combined_auth` 提取 token（`utils_api.py:392`）。
2. `auth_handler.validate_token(token)`（`auth.py:185`）：`jwt.decode` 显式单算法 → 校验 `sub` 类型与长度 ≤256 → 校验 `exp` 数值范围与过期。
3. 通过则注入 `request.state`；`_renew_token_if_needed`（`utils_api.py:140`）在响应头写 `X-New-Token` 滑动续期。

### 3.3 摄入请求准入

1. `POST /documents/upload` 命中 `AdmissionMiddleware`（`admission_middleware.py:52`）：
   - **先不调 `receive()`**，从 scope 头里取 bearer token 预认证（`asgi_helpers.py:27`）；
   - 认证通过则尝试占用一个待处理文档容量；
   - 拒绝时 `send_json`（`asgi_helpers.py:38`）直接写 401/423 响应，body 一字节不读。
2. 通过则构造 `AdmissionTicket` 发布到 `scope["state"]`（`admission.py:46`），再调用下游 app。
3. 路由内 `adopt_admission_ticket(request)`（`admission.py:51`）接管票据并标记 `adopted=True`，中间件 finally 不再释放。
4. 后台任务完成文档可见后释放；任一环节失败由对应层 finally 兜底释放。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `AUTH_ACCOUNTS` | 逗号分隔 `user:pass`；配置时必须显式设 `TOKEN_SECRET` | `auth.py:52`、`:77` |
| `LIGHTRAG_API_KEY` | 静态 API key；存在时启用认证 | `lightrag_server.py:1635` |
| `TOKEN_SECRET` | JWT 签名密钥；未配 AUTH_ACCOUNTS 时回退默认 guest 模式并告警 | `auth.py:54` |
| `JWT_ALGORITHM` | 必须为安全算法（HS256），`none` 拒绝 | `auth.py:64` |
| `TOKEN_EXPIRE_HOURS` / `GUEST_TOKEN_EXPIRE_HOURS` | 普通/guest token 过期时长 | `auth.py:71-72` |
| `MAX_PENDING_DOCUMENTS` | >0 时启用 AdmissionMiddleware；启动时探测 doc_status 严格计数 | `lightrag_server.py:1811` |
| 登录限流 | 默认 5 次 / 300 秒窗口，最多跟踪 10000 个 key | `login_rate_limit.py:46`、`:72` |

## 5. 错误与重试语义

- **认证失败**：401，detail 统一 "Invalid token"，不区分签名错误/过期/伪造。
- **容量满**：摄入请求在 body 读取前返回 423/429，客户端可稍后重试。
- **登录限流**：pending 占满时返回 `Retry-After: 1` 秒（bcrypt 远不到 1 秒）；confirmed 失败超限按窗口拒绝。
- **body 超限**：流式接收中抛 `BodyLimitExceeded`，中间件转 413，绝不泄漏到路由变 500。
- **账户配置错误**：`AUTH_ACCOUNTS` 格式错误或用户名超长在启动期 `ValueError` 拒绝，不拖到运行时。

## 6. 并发细节

- **登录限流无锁**：单 worker 事件循环线程访问；多 worker（gunicorn N）下有效限额 = N × max_attempts，仍可挡字典暴力。
- **pending/confirmed 分离**：限流判定用 `confirmed+pending` 防 TOCTOU；锁定告警只用 `confirmed`，正确密码不误报。
- **准入票据单所有者**：中间件→路由→后台任务三层显式交接，`adopted` 标记防止重复释放；未被采纳时中间件 finally 释放。
- **内存有界**：限流表最多 10000 key，满时只淘汰非活跃 key；活跃记录永不淘汰。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- JWT 签发/校验、bcrypt 密码哈希、登录限流、纯 ASGI 准入与 body 上限中间件、票据交接

**Out-of-Scope（不在本仓库源码内）**
- bcrypt/PyJWT 库本身（第三方依赖，`不在本仓库源码内`）
- 跨 worker 分布式限流（需外部 Redis，本仓库仅进程内）
- 上游 WAF/反向代理的大流量防护（仅做进程内基本界）

## 8. 与相邻子系统交互

- 上游：FastAPI 中间件栈（L27）安装；客户端携带 JWT/API key。
- 本叶子 → 下游：
  - `auth_handler` 被登录路由（L27 内 `/login`）与各路由 `Depends(combined_auth)` 使用；
  - `AdmissionMiddleware` 依赖 `rag.doc_status` 严格计数（storage 域）；
  - 票据交接给 document 路由（L28）与摄入后台任务（pipeline 域）。

## 9. 语言专项适配口径（纯 Python 框架口径）

按 `language-python.md` 落写：

- **能力缝**：认证是横切关注点，通过 FastAPI `Depends` 依赖注入缝入每个路由，而非路由内重复判断；中间件则是 ASGI 协议层的另一道缝（body 读取前 vs 路由参数解析后）。
- **配置驱动**：`AuthHandler.__init__` 在模块 import 时解析 `global_args.auth_accounts`，启动期拒绝坏配置（fail-fast），运行时无分支判断。
- **可选依赖懒加载**：bcrypt 为可选依赖，登录路径才真正调用；`psutil` 等同理。
- **安全即控制流**：时序攻击缓解（dummy bcrypt）、常量时间比较、声明长度上限都是把安全要求编码进正常控制流而非事后补丁。
- **图型侧重**：architecture（认证组件分层）+ sequence（登录请求时序，含限流/密码校验/token 签发）；准入票据交接是跨层状态传递而非单实体状态机，故不补 lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 认证准入架构图 | `api-auth-architecture.html` | architecture | standard（showcase 报 edge-through-node，把密码哈希节点移到外列后 standard 通过） |
| 登录请求时序 | `api-auth-sequence.html` | sequence | showcase |

本叶子未补 dataflow：认证不产生数据流管道。未补 lifecycle：AdmissionTicket 虽是状态对象但其状态迁移（未采纳/已采纳/已释放）由三层 finally 保证，非业务状态机，且只有 3 相位，不值得单独状态机图。

JSON IR 源文件位于 `json/` 目录。
