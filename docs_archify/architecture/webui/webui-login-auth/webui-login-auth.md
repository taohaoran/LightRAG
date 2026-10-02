# 登录与鉴权（webui-login-auth）

> 本文是 `webui` 域下的叶子子系统文档。域级总览见 `../webui.md`。
> 本文展开登录页、登录同意与登录身份逻辑；不展开后端 JWT 签发（api 域 L29）。
>
> 源码基准：`lightrag-hku` main 分支，commit `453dce83`；前端 `lightrag_webui/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 登录页组件 | LoginPage：用户名/密码表单、错误展示、品牌 | `src/features/LoginPage.tsx` |
| 登录同意 | loginConsent：首次登录同意文案 | `features/loginConsent.ts` |
| 登录身份 | loginIdentity：身份态判定 | `lib/loginIdentity.ts` |
| 登录请求 | loginToServer：POST /login 拿 token | L32 `api/lightrag.ts:1147` |
| 认证状态 | getAuthStatus：启动时查认证态 | L32 `api/lightrag.ts:1068` |
| 令牌存储 | localStorage `LIGHTRAG-API-TOKEN` | L32 `lightrag.ts:423` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `LoginPage` | `features/LoginPage.tsx` | 登录表单组件 |
| `loginConsent` | `features/loginConsent.ts` | 登录同意逻辑 |
| `loginIdentity` | `lib/loginIdentity.ts` | 身份判定 |
| `loginToServer` | L32 `lightrag.ts:1147` | 登录请求 |

## 3. 关键调用链

### 3.1 登录提交

1. 用户在 LoginPage 提交用户名/密码 → 调 `loginToServer`（L32 `lightrag.ts:1147`）。
2. 后端签发 JWT → 写 `localStorage['LIGHTRAG-API-TOKEN']`（`:423`）。
3. 更新 auth store → 跳转工作区（L35）。

### 3.2 启动认证探测

1. L35 authBootstrap 调 `getAuthStatus`（L32 `:1068`）。
2. 未配置认证 → guest token；已配置但无有效 token → 渲染 LoginPage。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 认证模式 | 由后端 auth_configured 决定 | L32 AuthStatusResponse |
| 令牌存储 | localStorage | L32 |

## 5. 错误与重试语义

- **凭据错误**：后端 401 → 表单展示错误。
- **无自动重试**：登录失败即展示。

## 6. 并发细节

- 登录按钮提交后禁用，防重复提交。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 登录表单、同意、身份逻辑

**Out-of-Scope（不在本仓库源码内）**
- 后端 JWT 签发/校验（api 域 L29）、bcrypt 密码哈希

## 8. 与相邻子系统交互

- 上游：L35 workspace 在未认证时渲染 LoginPage。
- 本叶子 → 下游：调 L32 loginToServer/getAuthStatus。

## 9. 语言专项适配口径（TypeScript/React 口径）

按 `language-ts.md` 落写：

- **capability seam**：Consumer——消费 L32 认证 API，提供登录表单 UI。
- **异步调用链**：提交→loginToServer→存 token→跳转 是 Promise 链。
- **外部边界**：后端 /login、localStorage 标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 登录架构图 | `webui-login-auth-architecture.html` | architecture | showcase |
| 登录时序 | `webui-login-auth-sequence.html` | sequence | showcase（消息 y 上限调整后通过） |

本叶子补 sequence（登录提交调用链）；未补 lifecycle：认证态由条件渲染表达；未补 dataflow：无数据管道。

JSON IR 源文件位于 `json/` 目录。
