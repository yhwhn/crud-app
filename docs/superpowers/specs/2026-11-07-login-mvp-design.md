# 登录 MVP 设计文档（Serverless / AWS）

- 状态：草稿 v3（Serverless 重写，待用户确认）
- 日期：2026-11-07
- 项目目录：`login-app/`（与现有 CRA 员工管理项目并存，不覆盖旧代码）
- 认证方案：自建认证，不使用 Cognito
- 部署：AWS Serverless
  - API Gateway HTTP API + Lambda（Node.js 22）
  - DynamoDB（4 张表，On-Demand）
  - S3 + CloudFront（React 静态站点）
  - SES + SQS + Lambda（异步发信）
  - CDK（TypeScript）+ GitHub Actions OIDC
- 预算目标：低流量下 **< $5/月**（不含域名年费）

> v3 是架构重写，不是 v2 的局部修补。产品行为（注册、登录、可选邮箱 MFA、找回密码、限流、锁定）保持不变；运行时从 Express + SQLite 改为 API Gateway + Lambda + DynamoDB。本文档尚未获得最终批准，也还没有写任何实现代码。

---

## 0. v3 相对 v2 的变化

| 维度 | v2（本地优先） | v3（Serverless） |
|---|---|---|
| 运行时 | 单台机器上的 Express | API Gateway + Lambda |
| 数据库 | SQLite 文件 | DynamoDB |
| 会话 | SQLite `sessions` 表 | DynamoDB `Sessions` 表 + HttpOnly Cookie |
| 限流 | SQLite 计数 | DynamoDB 原子计数 + TTL |
| 验证码 | SQLite | DynamoDB + TTL |
| 邮件 | 本地 console / SES | SQS + Lambda worker + SES |
| 前端托管 | 本地静态文件 | S3 + CloudFront |
| 基础设施 | 手工/脚本 | CDK |
| 成本模型 | 固定服务器 | 按请求付费 |
| 本地开发 | 直接跑 Express | Express + DynamoDB Local；邮件走 console |
| 公网入口 | EC2 + 公网 IP | CloudFront 域名（无公网 IP） |

**保留不变的底线**：bcrypt 密码哈希、验证码只存 HMAC、HttpOnly + Secure + SameSite=Lax Cookie、统一错误文案、不记录敏感信息、参数化/表达式化数据访问、不把 token 放 localStorage。

---

## 1. 意图与为什么

### 1.1 这个项目是什么

一个“登录优先”的小网站：

- 访客可以注册、验证邮箱、登录、退出、找回密码。
- 登录后是一个极简的 `/app` 占位页；以后可以放博客或项目介绍。
- 注册和找回密码通过 6 位邮箱验证码完成。
- 用户可以在设置页选择开启邮箱码 MFA；默认关闭。
- 登录、注册、发码、MFA 验证都有限流；密码和 MFA 连续失败会短时间锁定。

它不是员工管理系统，也不是博客系统。第一版只把“账号闭环”做完整。

### 1.2 为什么是 Serverless

用户的目标是投云/后端，且明确不能接受每月 $50–90 的固定成本。Serverless 同时满足三件事：

1. **成本**：低流量下按请求付费，没有空闲的 EC2/RDS 账单。
2. **云原生**：API Gateway、Lambda、DynamoDB、SQS、SES、CloudFront、CDK、IAM 都是真实的托管服务，不是“把单机搬到云上”。
3. **可交付**：不需要 VPC、NAT、RDS Proxy、负载均衡器，第一版能在较短时间内上线。

代价也要说清楚：

- Lambda 有冷启动；登录接口的 p95 可能比常驻进程高。
- DynamoDB 需要先设计访问模式，不能像 SQL 那样随时 join。
- 本地开发要跑 DynamoDB Local，调试链路比 SQLite 长。
- 异步发信失败、TTL 延迟删除、最终一致性都需要显式处理。

这些代价是项目要展示的一部分，不是要藏起来的问题。

### 1.3 受众

- 想注册/登录这个网站的人。
- 想看项目代码和部署方式的人。
- 面试时评估“是否真的理解云上认证”的人。

---

## 2. 目标、非目标、反目标

### 2.1 目标

- 完成注册 → 邮箱验证 → 登录 → 会话 → 退出 → 找回密码的闭环。
- 支持可选的邮箱码 MFA，默认关闭。
- 登录、注册、发码、MFA 验证有限流；密码和 MFA 失败有锁定。
- 部署在 AWS，使用 API Gateway、Lambda、DynamoDB、S3、CloudFront、SES、SQS、CloudWatch、IAM、CDK。
- 前端与 API 同源（CloudFront 的 `/api/*` 转发到 API Gateway），Cookie 不需要跨站。
- 有自动化测试、CDK 部署、GitHub Actions CI/CD、基本告警和成本告警。
- 低流量月成本 < $5（不含域名）。

### 2.2 非目标

- 博客/项目介绍的内容本身。
- 管理员后台、用户管理、审计日志查询页。
- 第三方登录（Google/GitHub）、Passkey、TOTP。
- 多区域、高可用、自动扩缩容压测到很高并发。
- 完整的 WAF/反欺诈体系；第一版只做 API Gateway 粗限流 + 应用层限流。
- 自定义邮件模板系统；第一版纯文本邮件。
- 把参考项目 `Two_Factor_Auth` 原样搬过来。

### 2.3 反目标（做到这些算失败）

- 密码、验证码、会话 token 明文落库或进日志。
- 用 `Scan` 做登录/验证码查询。
- S3 桶公开，或 IAM 用 `*`。
- 把 JWT 放 localStorage。
- 为了“显得用了 AWS”而硬拆微服务，导致项目讲不清为什么拆。
- 为了省事把限流/锁定做成进程内内存计数（Lambda 多实例下无效）。
- 部署后没有成本告警，月底账单失控。

---

## 3. 约束

- 新代码放在 `login-app/`，不改造旧 CRA 的 `src/`；旧项目保留作为参考。
- 前端：Vite + React（JavaScript，不上 TypeScript）、React Router、纯 CSS；不引入 UI 组件库。
- 后端：Node.js，Express 应用通过 `serverless-http` 跑在 Lambda 上。
- Lambda Runtime：`nodejs22.x`；本地开发机是 Node 24，代码保持 22/24 兼容。
- 数据库：DynamoDB On-Demand；4 张表；第一版不建 GSI；不使用 `Scan`。
- 邮件：SES；开发环境用 console adapter，不真的发信。
- 基础设施：AWS CDK（TypeScript，只用于 `infra/`）。
- 区域：默认 `ap-southeast-1`（新加坡）；CloudFront 证书在 `us-east-1`。
- 域名：自定义域名 + ACM + Route 53；没有域名时先用 CloudFront 默认域名。
- 预算：低流量 < $5/月；设置 AWS Budgets 告警。
- 验证码规则：6 位数字、10 分钟有效、最多 5 次尝试、60 秒重发、单次使用。
- 密码：bcryptjs cost 12；最短 8 位；最长 72 字节（bcrypt 限制）。
- 会话：7 天；HttpOnly + Secure + SameSite=Lax；数据库只存 token 的 SHA-256。
- MFA：可选、默认关闭；只做邮箱码；不做 TOTP/恢复码。

---

## 4. 方案选择

| 方案 | 认证逻辑 | 基础设施 | 月成本（低流量） | 学习价值 | 主要风险 |
|---|---|---|---|---|---|
| A. Cognito + 自定义 UI | Cognito 管密码/会话/验证 | Cognito + API Gateway + Lambda | 低 | 中（学 Cognito 周边） | 黑盒、锁定/限流行为受托管服务限制 |
| B. Cognito Hosted UI | Cognito 全包 | Amplify/Cognito | 低 | 低（代码量太少） | 页面不可控、项目显得薄 |
| C. 自建 + EC2 + Express + SQLite | 自己写 | EC2 + Caddy + SQLite | $10–15 | 高（但偏传统运维） | 单点、固定成本、云原生含量低 |
| D. 自建 + Serverless（选中） | 自己写 | API Gateway + Lambda + DynamoDB + S3/CloudFront + SES/SQS + CDK | $1–5 | 高（认证 + 云原生） | 冷启动、DynamoDB 建模、异步链路复杂度 |

**为什么选 D：**

- 用户要投云/后端，D 能展示“用托管服务解决具体问题”，而不是“把 Express 放进 EC2”。
- 成本符合“不能接受 $50–90/月”的硬约束。
- 认证逻辑仍然是自己的：密码哈希、验证码、会话、限流、锁定都能讲清楚。
- 不需要 VPC/NAT/RDS，第一版范围可控。

**为什么不用 Cognito：** 这个项目的学习目标就是亲手实现认证；Cognito 会把最想讲的部分变成配置。

**为什么不用 ECS Fargate + RDS：** 架构更接近生产，但固定成本约 $50–90/月，用户明确不接受；可以作为以后有预算时的迁移方向。

---

## 5. 第一版范围

### 5.1 做

- 注册（邮箱 + 密码），发送 6 位验证码。
- 邮箱验证；未验证邮箱不能登录。
- 登录（邮箱 + 密码）。
- 可选邮箱码 MFA：设置页开启/关闭；开启后登录多一步验证码。
- 退出登录。
- 刷新页面保持登录（7 天会话）。
- 忘记密码 / 重置密码（6 位邮箱码）。
- 重发验证码页面 `/resend-verification`。
- `/app` 占位页 + 路由守卫。
- 登录/注册/发码/MFA 限流。
- 密码 5 次失败锁 15 分钟；MFA 10 次失败锁 15 分钟。
- AWS 部署：API Gateway、Lambda、DynamoDB、S3、CloudFront、SES、SQS、CloudWatch、Secrets Manager、IAM、CDK。
- GitHub Actions：测试、构建、CDK synth/deploy、前端发布。
- 基础告警：Lambda 错误、API 5xx、DLQ、DynamoDB 限流、SES 退信。
- 成本告警。

### 5.2 不做

- 博客/项目介绍内容。
- 管理员后台、用户列表、审计日志查询。
- 第三方登录、Passkey、TOTP、短信。
- 多区域、高可用、灾备演练。
- WAF、验证码图片、设备指纹、风控。
- 自定义邮件模板、邮件打开率。
- MFA 恢复码；邮箱丢失只能管理员手动处理。
- 公开首页（待定，见 18.1）。

### 5.3 安全底线（不能砍）

- bcrypt 哈希密码；不存明文。
- 验证码只存 HMAC-SHA256；不存明文、不返回给客户端。
- 会话 token 只存 SHA-256；Cookie 是 HttpOnly/Secure/SameSite=Lax。
- 登录失败统一文案；不暴露“邮箱是否存在”。
- 限流/锁定用 DynamoDB 共享状态，不用进程内内存。
- DynamoDB 只用 GetItem/Query/PutItem/UpdateItem/DeleteItem；不 Scan。
- IAM 最小权限；S3 阻止公共访问；Secrets Manager 存密钥。
- 日志不记录密码、验证码、Cookie、完整 token。
- 所有写接口校验 `Content-Type: application/json` 和 `Origin`（同源）。

---

## 6. 用户流程

### 6.1 注册 + 邮箱验证

1. 用户打开 `/register`，输入邮箱、密码、确认密码。
2. 前端校验格式/长度；提交 `POST /api/auth/register`。
3. 服务端规范化邮箱（trim + lowercase）。
4. 检查 `Users` 是否已存在：
   - 已存在：返回 `409 EMAIL_ALREADY_REGISTERED`，页面提示“如果这是你的邮箱，可以直接登录或重发验证邮件”。
   - 不存在：bcrypt 哈希密码；写入 `Users`（`email_verified_at` 为空）；生成 6 位码；写 `VerificationCodes`；把发信任务放进 SQS。
5. 返回 `201`。前端跳到 `/verify-email?email=...`。
6. 用户收到邮件，输入 6 位码；提交 `POST /api/auth/verify-email`。
7. 服务端查最新码，校验哈希、过期、尝试次数、用途；成功后更新 `email_verified_at`，标记码已消费。
8. 返回 `200`；前端跳到 `/login`，提示“邮箱已验证，可以登录”。

**失败处理：**

- 验证码错误：`400 INVALID_CODE`，剩余尝试次数不告诉前端具体值，只提示“验证码不正确或已过期”。
- 验证码过期：同上。
- 重发：`POST /api/auth/resend-verification`；同一邮箱 60 秒冷却，15 分钟内最多 3 次；返回统一的 `200`，不暴露邮箱是否存在。
- 发信队列失败：注册接口返回 `503 EMAIL_QUEUE_UNAVAILABLE`；用户可稍后重发。用户记录已创建，重发不需要重新注册。

### 6.2 登录

1. 用户打开 `/login`，输入邮箱、密码。
2. 提交 `POST /api/auth/login`。
3. 服务端：
   - 按邮箱取用户；不存在时用一个固定的假 bcrypt 哈希做一次比较，避免时间差。
   - 检查 `locked_until`；未到期返回 `423 ACCOUNT_LOCKED` + `Retry-After`。
   - 校验密码；失败则原子增加 `failed_login_count`，达到 5 次写 `locked_until = now + 15min`，返回 `401 INVALID_CREDENTIALS`。
   - 密码正确但邮箱未验证：返回 `403 EMAIL_NOT_VERIFIED`，前端提示去验证/重发。
   - 密码正确且 `mfa_enabled = false`：创建会话，设置 Cookie，返回 `200 { user }`。
   - 密码正确且 `mfa_enabled = true`：生成 6 位码（用途 `mfa_login`）；签发 `mfaToken`（HMAC，10 分钟）；返回 `200 { mfaRequired: true, mfaToken }`，**不设置会话 Cookie**。
4. 前端把 `mfaToken` 放在内存（React state），跳到 `/mfa`。
5. 用户输入邮箱码；提交 `POST /api/auth/mfa/verify`，body 含 `mfaToken` + `code`。
6. 服务端验签、检查过期、取用户、检查 `session_version` 是否与 token 一致、检查锁定、查最新码、校验；成功后创建会话，设置 Cookie，返回 `200 { user }`。
7. 失败：`401 INVALID_CODE`；码的 `attempts` 增加；达到 5 次该码作废；`failed_mfa_count` 增加；达到 10 次锁 15 分钟，返回 `423`。

**重发 MFA 码：** `POST /api/auth/mfa/resend`，body 含 `mfaToken`；60 秒冷却；每账号 15 分钟最多 3 次；返回 `200`，不返回新码。

### 6.3 刷新页面保持登录

1. 应用启动时 `AuthProvider` 调 `GET /api/auth/me`。
2. 服务端读 Cookie，SHA-256 后查 `Sessions`；不存在/过期 → `401`。
3. 取 `Users`，比较 `session_version`；不一致 → 删除会话，`401`。
4. 返回 `{ user }`；前端进入 `/app`。
5. 任意 `/api/*` 返回 `401`：前端清空用户，跳 `/login`，并带上当前路径作为 `from`。

### 6.4 退出

1. `POST /api/auth/logout`。
2. 服务端删除 `Sessions` 项；清除 Cookie；返回 `204`。
3. 前端清空用户，跳 `/login`。

### 6.5 忘记密码 / 重置密码

1. `/forgot-password` 输入邮箱；`POST /api/auth/forgot-password`。
2. 无论邮箱是否存在，都返回 `200`；存在则生成 `reset_password` 码并入队发信。
3. `/reset-password?email=...` 输入 6 位码 + 新密码；`POST /api/auth/reset-password`。
4. 服务端校验码；成功后：
   - 更新 `password_hash`
   - `session_version += 1`（所有旧会话失效）
   - 清空 `failed_login_count`、`failed_mfa_count`、`locked_until`
   - 标记码已消费
5. 返回 `200`；前端跳 `/login`。
6. 未验证邮箱也可以重置密码，但重置后登录仍会被 `EMAIL_NOT_VERIFIED` 拦住；前端提示先验证邮箱。

### 6.6 MFA 设置

**开启：**

1. `/app/settings` 点“开启邮箱验证码”。
2. 输入当前密码；`POST /api/auth/mfa/enable`。
3. 服务端校验密码；生成 `enable_mfa` 码；入队发信；返回 `202`。
4. 输入验证码；`POST /api/auth/mfa/enable/confirm`。
5. 校验成功后 `mfa_enabled = true`；清空 `failed_mfa_count`；返回 `200`。

**关闭：**

1. 点“关闭邮箱验证码”。
2. 输入当前密码；`POST /api/auth/mfa/disable`。
3. 服务端校验密码；生成 `disable_mfa` 码；入队发信；返回 `202`。
4. 输入验证码；`POST /api/auth/mfa/disable/confirm`（body 含 password + code）。
5. 校验成功后 `mfa_enabled = false`；清空 `failed_mfa_count`；返回 `200`。

**规则：**

- 开启和关闭都要当前密码；开启还要一次邮箱码，关闭还要当前邮箱码。
- 不做恢复码；邮箱丢失只能管理员手动处理（第一版没有管理员界面，用 AWS 控制台/DynamoDB 操作）。
- MFA 开关不强制踢出其他会话；如需“关闭 MFA 后所有设备重新登录”，后续可加 `session_version += 1`。

### 6.7 路由守卫

- 公开路由：`/login`、`/register`、`/verify-email`、`/resend-verification`、`/forgot-password`、`/reset-password`、`/mfa`。
- 受保护路由：`/app`、`/app/settings`。
- `/`：第一版 `Navigate to="/app"`；若加公开首页，改为公开 Landing（见 18.1）。
- `RequireAuth` 在 `loading` 时显示骨架；未登录 `<Navigate to="/login" state={{ from: location }} />`；登录成功回跳 `from`，只接受站内路径。

### 6.8 限流/锁定时的用户体验

- `429`：显示“操作太频繁，请稍后再试”，带 `Retry-After` 秒数；按钮倒计时。
- `423`：显示“尝试次数过多，请 15 分钟后再试”；不告诉用户是密码还是 MFA 触发。
- 验证码错误不显示剩余次数；只显示统一错误。

---

## 7. 技术架构

### 7.1 技术栈

| 层 | 选择 | 说明 |
|---|---|---|
| 前端 | Vite + React（JS） | React Router；纯 CSS；无 UI 库 |
| 前端托管 | S3 + CloudFront | 私有桶 + OAC；自定义域名 + ACM |
| API | API Gateway HTTP API | 便宜、够用；`/api/*` 走 Lambda |
| 计算 | Lambda（Node.js 22） | Express 通过 `serverless-http` 适配 |
| 数据库 | DynamoDB On-Demand | 4 张表；TTL；条件写；无 GSI |
| 邮件 | SES + SQS + Lambda worker | 异步、可重试、有 DLQ |
| 密钥 | Secrets Manager | 一个 JSON secret 存两个 HMAC 密钥 |
| 日志/追踪 | CloudWatch + X-Ray | 结构化日志；基础告警 |
| IaC | AWS CDK（TypeScript） | `infra/`；多 stack |
| CI/CD | GitHub Actions + OIDC | 不用长期 AWS 密钥 |
| 本地 DB | DynamoDB Local（Docker） | 与线上 API 兼容 |

### 7.2 请求路径

```text
Browser
  │  https://app.example.com
  ▼
CloudFront (ACM, HTTPS, 自定义域名)
  ├── /*      → S3 (React build, OAC, 私有桶)
  └── /api/*  → API Gateway HTTP API
                   │
                   ▼
                Lambda (Express via serverless-http)
                   ├── DynamoDB: Users / Sessions / VerificationCodes / RateLimits
                   ├── SQS: 发信任务
                   └── CloudWatch / X-Ray
                          │
                          ▼
                   Lambda (email worker) → SES → 用户邮箱
```

- HTTP API 用 `$default` 路由（或 `ANY /api/{proxy+}`）把请求交给同一个 Lambda；Express 负责 `/api/*` 内部路由。
- **没有 EC2、没有公网 IP、没有 ALB、没有 NAT、没有 RDS。** 公网入口是 CloudFront 的域名；自定义域名让推广链接更可信，也让 Cookie 的 `Secure` 生效。

### 7.3 仓库结构

```text
login-app/
  package.json                 # workspaces + scripts
  README.md
  .env.example
  .gitignore
  web/
    index.html
    vite.config.js
    src/
      main.jsx
      App.jsx
      router.jsx
      lib/api.js
      auth/AuthProvider.jsx
      auth/RequireAuth.jsx
      pages/
        Login.jsx
        Register.jsx
        VerifyEmail.jsx
        ResendVerification.jsx
        ForgotPassword.jsx
        ResetPassword.jsx
        MfaVerify.jsx
        AppHome.jsx
        Settings.jsx
      styles.css
  api/
    package.json
    src/
      app.js                   # Express app
      server.js                # 本地 listen
      lambda.js                # export handler
      config.js
      db/
        client.js              # DynamoDB DocumentClient
        users.js
        sessions.js
        codes.js
        rateLimits.js
      services/
        authService.js
        codeService.js
        emailService.js
        rateLimitService.js
        lockoutService.js
        tokenService.js
      middleware/
        auth.js
        rateLimit.js
        errors.js
      routes/
        auth.js
        mfa.js
        health.js
      emails/
        templates.js
    test/
  infra/
    bin/app.ts
    lib/data-stack.ts
    lib/api-stack.ts
    lib/web-stack.ts
    lib/observability-stack.ts
    cdk.json
    package.json
  tests/
    e2e/
    load/
  docs/
    adr/
    runbook.md
```

### 7.4 本地开发

```bash
# 1. DynamoDB Local
docker run -d -p 8000:8000 amazon/dynamodb-local

# 2. 安装
npm install

# 3. 建本地表
npm run db:local

# 4. 启动 web + api
npm run dev
```

- `web`：Vite `http://localhost:5173`，`/api` 代理到 `http://localhost:3001`。
- `api`：Express `http://localhost:3001`，`AWS_ENDPOINT_URL_DYNAMODB=http://localhost:8000`。
- `EMAIL_DRIVER=console`：验证码直接打印在 API 终端；不连 SES/SQS。
- `AWS_ACCESS_KEY_ID=local`、`AWS_SECRET_ACCESS_KEY=local`、`AWS_REGION=ap-southeast-1` 仅本地使用。
- 本地不启 API Gateway/CloudFront；前端代理模拟同源。

### 7.5 配置与密钥

`.env.example`：

```dotenv
PORT=3001
APP_BASE_URL=http://localhost:5173
AWS_REGION=ap-southeast-1
AWS_ENDPOINT_URL_DYNAMODB=http://localhost:8000
TABLE_USERS=login-app-users-dev
TABLE_SESSIONS=login-app-sessions-dev
TABLE_CODES=login-app-codes-dev
TABLE_RATE_LIMITS=login-app-rate-limits-dev
EMAIL_DRIVER=console
SES_FROM=no-reply@example.com
SECRET_ARN=
CODE_SECRET=dev-only-not-for-production
MFA_TOKEN_SECRET=dev-only-not-for-production
```

生产：

- `SECRET_ARN` 指向 Secrets Manager 的一个 JSON secret：
  ```json
  { "codeSecret": "...", "mfaTokenSecret": "..." }
  ```
- Lambda 冷启动时读取并缓存；不在环境变量里放明文密钥。
- `.env` 不进 git；`.env.example` 只放占位。
- 本地 `CODE_SECRET`/`MFA_TOKEN_SECRET` 只用于开发，部署时拒绝使用默认值。

### 7.6 CDK Stack 划分

| Stack | 资源 |
|---|---|
| `DataStack` | 4 张 DynamoDB 表；Users PITR；TTL；删除保护（prod） |
| `ApiStack` | HTTP API、Lambda、SQS + DLQ、email worker、Secrets、IAM、日志、X-Ray、告警 |
| `WebStack` | S3（版本控制）、CloudFront、OAC、ACM、Route 53、自定义域名 |
| `ObservabilityStack` | Dashboard、alarms、SNS 告警主题、Budget（可选独立） |

- `WebStack` 配置 SPA fallback：CloudFront 403/404 → `/index.html`（200），`defaultRootObject=index.html`；直接打开 `/login`、`/app` 不会 404。
- `ApiStack` 依赖 `DataStack`；`WebStack` 的 `/api/*` origin 指向 `ApiStack` 的 API URL。
- `/api/*` 关闭 CloudFront 缓存；origin request policy 转发 Cookie、Origin、Authorization；`Set-Cookie` 透传，不做缓存。
- 环境用 CDK context：`-c env=dev` / `-c env=prod`；资源名带环境后缀。
- `DataStack` 导出表名；`ApiStack` 通过 `env` 注入 Lambda。
- 第一版一个区域；不做多区域。

### 7.7 CI/CD

- `ci.yml`：`npm ci` → lint → unit/integration → `npm run build` → `cdk synth`。
- `deploy.yml`：手动触发或 push `main`；GitHub OIDC assume role；`cdk deploy --all`；前端 `aws s3 sync` + CloudFront invalidation。
- GitHub OIDC：IAM role 信任 `token.actions.githubusercontent.com`，限定 repo/branch；不存长期 access key。
- 生产部署可加 GitHub Environment 手动批准；第一版 dev 自动、prod 手动。

### 7.8 可观测性

- Lambda 结构化日志：`requestId`、route、status、duration、userId（可选）；不打码/密码/Cookie。
- API Gateway access logs（可选，注意成本）；HTTP API 默认指标够用。
- X-Ray：API Gateway → Lambda → DynamoDB/SQS；看冷启动和下游延迟。
- CloudWatch alarms：
  - Lambda `Errors` > 0（5 分钟）
  - API `5xx` > 1%
  - SQS `ApproximateNumberOfMessagesVisible` 或 DLQ > 0
  - DynamoDB `ThrottledRequests` > 0
  - SES `Bounce`/`Complaint`（若配置）
- 告警发 SNS → 邮箱；第一版不接 PagerDuty。
- 日志保留 14 天，控制成本。

---

## 8. DynamoDB 数据模型

### 8.1 约定

- 时间统一用 **epoch 秒**（DynamoDB TTL 要求秒级数字）。
- 所有表 On-Demand；第一版 **不建 GSI**。
- 不使用 `Scan`；所有查询都必须有分区键。
- 认证相关读使用 `ConsistentRead: true`（用户、会话、验证码），避免刚写完读不到。
- 条件写用于注册、消费验证码、更新计数；需要两个 item 原子更新时用 `TransactWriteItems`。
- TTL 只负责“最终清理”，业务代码必须自己检查 `expires_at`；TTL 删除可能延迟 48 小时。

### 8.2 Users

表名：`login-app-users-<env>`；PK：`email`（S）。

| 属性 | 类型 | 说明 |
|---|---|---|
| `email` | S | PK；trim + lowercase |
| `user_id` | S | `crypto.randomUUID()` |
| `password_hash` | S | bcryptjs cost 12 |
| `email_verified_at` | N | epoch 秒；未验证则不存在 |
| `mfa_enabled` | BOOL | 默认 false |
| `failed_login_count` | N | 连续密码失败 |
| `failed_mfa_count` | N | 连续 MFA 失败 |
| `locked_until` | N | epoch 秒；未锁定则不存在 |
| `session_version` | N | 初始 1；重置密码 +1 |
| `created_at` | N | epoch 秒 |
| `updated_at` | N | epoch 秒 |

访问：

- 登录/注册/查用户：`GetItem { email }`，`ConsistentRead: true`。
- 创建：`PutItem`，条件 `attribute_not_exists(email)`；冲突 → 409。
- 失败计数：`UpdateItem ADD failed_login_count :one`，返回新值；达到阈值再 `SET locked_until`。
- 重置密码：`TransactWriteItems` 更新 Users + 消费验证码。

没有 GSI：会话表存 `email`，查用户直接按 email；不需要“按 user_id 查用户”。

### 8.3 Sessions

表名：`login-app-sessions-<env>`；PK：`token_hash`（S，SHA-256 hex）。

| 属性 | 类型 | 说明 |
|---|---|---|
| `token_hash` | S | PK；Cookie 原 token 的 SHA-256 |
| `user_id` | S | 冗余，便于日志 |
| `email` | S | 查 Users |
| `session_version` | N | 签发时的版本 |
| `created_at` | N | epoch 秒 |
| `expires_at` | N | epoch 秒；TTL 属性 |
| `ip_hash` | S | 可选；SHA-256，不存原始 IP |
| `user_agent` | S | 可选；截断 |

访问：

- 读会话：`GetItem { token_hash }`，`ConsistentRead: true`。
- 创建：`PutItem`。
- 退出：`DeleteItem`。
- 过期：TTL 清理，但读时仍检查 `expires_at`。
- 重置密码：`session_version += 1`；旧会话读到时版本不一致 → 删除并 401。不需要按用户查所有会话（没有 GSI）。

### 8.4 VerificationCodes

表名：`login-app-codes-<env>`；PK：`scope`（S），SK：`sort_key`（S）。

- `scope = ${email}#${purpose}`
- `sort_key = ${created_at}#${code_id}`
- 用途：`verify_email`、`reset_password`、`mfa_login`、`enable_mfa`、`disable_mfa`。

| 属性 | 类型 | 说明 |
|---|---|---|
| `scope` | S | PK |
| `sort_key` | S | SK |
| `code_id` | S | UUID |
| `code_hash` | S | HMAC-SHA256(code) |
| `created_at` | N | epoch 秒 |
| `expires_at` | N | epoch 秒；TTL |
| `attempts` | N | 已尝试次数 |
| `consumed_at` | N | 成功消费时间；未消费则不存在 |
| `last_sent_at` | N | 用于 60 秒冷却 |
| `purpose` | S | 冗余，便于审计 |

访问：

- 取最新码：`Query` PK=`scope`，`ScanIndexForward: false`，`Limit: 1`，`ConsistentRead: true`。
- 创建：`PutItem`；新码成为最新，旧码自然被忽略（只查最新一条）。
- 消费：`UpdateItem` 条件 `attribute_not_exists(consumed_at) AND attempts < :max AND expires_at > :now`；成功后 `SET consumed_at = :now`。
- 失败：`UpdateItem ADD attempts :one`；达到 5 次后不再可用（条件里 `attempts < 5`）。
- 冷却：读最新码的 `last_sent_at`；不足 60 秒返回 429。
- 不做“删除旧码”；TTL 清理，业务只认最新。

### 8.5 RateLimits

表名：`login-app-rate-limits-<env>`；PK：`bucket`（S）。

- `bucket = ${key_hash}#${window_start}`；`window_start = floor(now / window) * window`。
- 每次窗口一个 item；TTL 清理。

| 属性 | 类型 | 说明 |
|---|---|---|
| `bucket` | S | PK |
| `count` | N | 当前窗口计数 |
| `reset_at` | N | epoch 秒；TTL |
| `key_hash` | S | 可选；便于排查（不存原始邮箱/IP） |

访问：

```text
UpdateItem {
  Key: { bucket },
  UpdateExpression: "ADD #count :one SET #reset_at = if_not_exists(#reset_at, :reset_at)",
  ExpressionAttributeNames: { "#count": "count", "#reset_at": "reset_at" },
  ExpressionAttributeValues: { ":one": 1, ":reset_at": windowEnd },
  ReturnValues: "ALL_NEW"
}
```

- 返回 `count > limit` → 429；`Retry-After = reset_at - now`。
- 原子递增，Lambda 并发下不会丢计数。
- 不需要 GSI；限流键只按 bucket 读写。

### 8.6 访问模式

| 场景 | 操作 | 表 |
|---|---|---|
| 按邮箱取用户 | GetItem `email` | Users |
| 创建用户 | PutItem 条件 `attribute_not_exists` | Users |
| 密码失败计数 | UpdateItem `ADD` | Users |
| 锁定 | UpdateItem `SET locked_until` | Users |
| 重置密码 | TransactWriteItems（Users + Codes） | Users/Codes |
| 读会话 | GetItem `token_hash` | Sessions |
| 创建/删除会话 | PutItem / DeleteItem | Sessions |
| 取最新验证码 | Query PK，降序 Limit 1 | Codes |
| 创建验证码 | PutItem | Codes |
| 消费验证码 | UpdateItem 条件 | Codes |
| 限流计数 | UpdateItem `ADD` | RateLimits |

**没有 Scan，没有 GSI。** 这是有意的：第一版访问模式简单，多表 + 明确 PK 比强行单表更好讲。

### 8.7 一致性与原子性

- 注册：`PutItem` 条件写，防并发重复注册。
- 验证码消费：条件写，防同一码并发使用。
- 密码失败计数：`ADD` 原子；锁定的 `SET` 是第二次写，极端并发可能多记一两次，可接受。
- 重置密码：`TransactWriteItems` 同时更新 Users（password_hash、session_version、清锁定）和 Codes（consumed_at），避免“密码改了但码没消费”。
- 验证邮箱：可以 `TransactWriteItems` 更新 Users + Codes；也可以先消费码再更新用户；第一版用事务，语义更清楚。
- 读用户/会话/码用 `ConsistentRead`，避免刚注册就登录读不到。

### 8.8 TTL 注意

- TTL 删除不是即时的，可能延迟最多 48 小时；所有读都要检查 `expires_at`。
- 过期验证码仍可能被 Query 到；校验时先判断 `expires_at > now`。
- 过期会话仍可能被 GetItem 到；校验时判断 `expires_at`。
- 过期限流 item 不影响新窗口，因为 bucket 含 `window_start`。

---

## 9. API 设计

### 9.1 约定

- Base：`/api`。
- JSON：`Content-Type: application/json`；响应 `application/json`。
- 认证：HttpOnly Cookie `app_session`；MFA 登录用 body 里的 `mfaToken`（不是 Cookie）。
- 错误格式：
  ```json
  { "error": { "code": "INVALID_CREDENTIALS", "message": "邮箱或密码不正确" } }
  ```
- 429/423 带 `Retry-After` 响应头。
- 所有 POST 校验 `Origin` 与 `APP_BASE_URL` 同源；不匹配返回 403。
- 不使用 GET 做状态变更。
- API Gateway HTTP API 阶段限流：20 rps / burst 40（应用层限流仍然生效）。

### 9.2 端点

| 方法 | 路径 | 认证 | 成功 | 主要错误 |
|---|---|---|---|---|
| POST | `/api/auth/register` | 无 | 201 | 400, 409, 429, 503 |
| POST | `/api/auth/verify-email` | 无 | 200 | 400, 429 |
| POST | `/api/auth/resend-verification` | 无 | 200 | 400, 429 |
| POST | `/api/auth/login` | 无 | 200（用户或 MFA challenge） | 401, 403, 423, 429 |
| POST | `/api/auth/mfa/verify` | mfaToken | 200 | 401, 423, 429 |
| POST | `/api/auth/mfa/resend` | mfaToken | 200 | 401, 429 |
| POST | `/api/auth/logout` | Cookie | 204 | 401 |
| GET | `/api/auth/me` | Cookie | 200 | 401 |
| POST | `/api/auth/forgot-password` | 无 | 200 | 429 |
| POST | `/api/auth/reset-password` | 无 | 200 | 400, 429 |
| POST | `/api/auth/mfa/enable` | Cookie | 202 | 401, 409, 429 |
| POST | `/api/auth/mfa/enable/confirm` | Cookie | 200 | 400, 401, 429 |
| POST | `/api/auth/mfa/disable` | Cookie | 202 | 401, 429 |
| POST | `/api/auth/mfa/disable/confirm` | Cookie | 200 | 400, 401, 429 |
| GET | `/api/health` | 无 | 200 | — |

### 9.3 关键请求/响应

**注册：**

```json
POST /api/auth/register
{ "email": "a@example.com", "password": "correct horse battery staple" }
→ 201 { "message": "验证码已发送" }
```

**登录（无 MFA）：**

```json
POST /api/auth/login
{ "email": "a@example.com", "password": "..." }
→ 200 { "user": { "id": "...", "email": "a@example.com", "mfaEnabled": false } }
Set-Cookie: app_session=...; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800
```

**登录（有 MFA）：**

```json
→ 200 {
  "mfaRequired": true,
  "mfaToken": "<base64url payload>.<base64url signature>"
}
```

**MFA 验证：**

```json
POST /api/auth/mfa/verify
{ "mfaToken": "...", "code": "123456" }
→ 200 { "user": { ... } }
Set-Cookie: app_session=...
```

**重置密码：**

```json
POST /api/auth/reset-password
{ "email": "a@example.com", "code": "123456", "newPassword": "..." }
→ 200 { "message": "密码已重置" }
```

### 9.4 `mfaToken`

不落库，HMAC 签名：

```text
payload = base64url(JSON.stringify({
  sub: user_id,
  email,
  sv: session_version,
  purpose: "mfa_login",
  iat,
  exp: iat + 600
}))
signature = base64url(HMAC-SHA256(payload, MFA_TOKEN_SECRET))
mfaToken = payload + "." + signature
```

校验：

- 签名用 `crypto.timingSafeEqual`。
- `exp` 未过期；`purpose` 正确。
- 取用户；`user.session_version === sv`；否则 401（重置密码后旧 token 失效）。
- 用户 `mfa_enabled` 仍为 true；否则 401。
- 用户未锁定。

优点：不需要 `mfa_challenges` 表；Lambda 无状态；重置密码自动失效。缺点：签发后无法单独撤销（但 10 分钟过期，且 `session_version` 可失效）。

### 9.5 错误码

| code | HTTP | 说明 |
|---|---|---|
| `VALIDATION_ERROR` | 400 | 字段缺失/格式错 |
| `INVALID_CREDENTIALS` | 401 | 邮箱或密码错误 |
| `EMAIL_NOT_VERIFIED` | 403 | 邮箱未验证 |
| `EMAIL_ALREADY_REGISTERED` | 409 | 注册重复 |
| `INVALID_CODE` | 400 | 验证码错误/过期/已用 |
| `CODE_COOLDOWN` | 429 | 60 秒内重复发码 |
| `RATE_LIMITED` | 429 | 限流 |
| `ACCOUNT_LOCKED` | 423 | 锁定 |
| `MFA_ALREADY_ENABLED` | 409 | 已开启 MFA |
| `MFA_NOT_ENABLED` | 409 | 未开启 MFA |
| `EMAIL_QUEUE_UNAVAILABLE` | 503 | 入队失败 |
| `INTERNAL_ERROR` | 500 | 未预期错误 |

对外文案统一、简短；不返回堆栈、SQL/表名、内部 ID。

---

## 10. 会话与安全

### 10.1 会话 Cookie

- 名称：`app_session`。
- 值：32 字节随机数，base64url；只在 Cookie 里出现，数据库存 SHA-256。
- 属性：`HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800`。
- 生产必须 HTTPS（CloudFront + ACM）；本地 HTTP 不加 `Secure`（用环境判断）。
- 不做滑动续期；7 天后过期，重新登录。
- 退出删除 Sessions 项并清 Cookie。
- 重置密码 `session_version += 1`；旧会话在下一次请求时被拒绝并删除。

### 10.2 密码

- bcryptjs cost 12；异步 `hash`/`compare`。
- 最短 8 位；最长 72 字节；不 trim 密码；不限制特殊字符。
- 注册和重置都校验。
- 登录时邮箱不存在也做一次假哈希比较，降低时间差。
- 不返回“密码错误”与“邮箱不存在”的区别。

### 10.3 验证码

- 6 位数字；`crypto.randomInt(0, 1_000_000)` 补零。
- 只存 `HMAC-SHA256(code, CODE_SECRET)`；不存明文。
- 10 分钟过期；5 次尝试；60 秒重发；单次使用。
- 校验用 `timingSafeEqual`。
- 新码创建后，旧码不再被查询（只查最新）；不需要批量删除。
- 邮件正文只发码，不发链接；降低钓鱼/扫描器误点。
- 不把码返回给客户端；开发环境 console adapter 除外。

### 10.4 限流矩阵

| 动作 | 键 | 限额 | 窗口 |
|---|---|---|---|
| 登录 | IP | 20 | 15 分钟 |
| 注册 | IP | 5 | 1 小时 |
| 重发注册验证码 | 邮箱 | 3 | 15 分钟 |
| 验证邮箱提交 | 邮箱 | 10 | 15 分钟 |
| 验证邮箱提交 | IP | 30 | 15 分钟 |
| 忘记密码 | 邮箱 | 3 | 15 分钟 |
| 忘记密码 | IP | 10 | 1 小时 |
| 重置密码提交 | IP | 10 | 15 分钟 |
| MFA 验证 | IP | 30 | 15 分钟 |
| MFA 重发 | 账号 | 3 | 15 分钟 |
| MFA 设置发码 | 账号 | 5 | 15 分钟 |

- IP 从 API Gateway `requestContext.http.sourceIp` 取；本地从 `req.ip`。
- 邮箱/IP 在写 `RateLimits` 前做 SHA-256，避免表里出现原始值。
- 超限返回 429 + `Retry-After`；不告诉具体还剩多少。
- API Gateway 阶段限流是粗粒度兜底；应用层矩阵才是业务限流。

### 10.5 账号锁定

- 密码连续失败 5 次 → `locked_until = now + 15min`。
- MFA 连续失败 10 次 → `locked_until = now + 15min`。
- 两个计数分开；共用一个 `locked_until`。
- 登录成功（密码 + MFA 都通过）清零两个计数。
- 重置密码清空两个计数和 `locked_until`。
- 锁定期间返回 423 + `Retry-After`；不区分密码/MFA。
- 单次 MFA challenge 错 5 次作废；用户需重新登录获取新 `mfaToken`。
- 锁定是账号级，攻击者可以用错误密码锁别人账号（DoS）；用 IP 限流、15 分钟短锁、统一文案降低；第一版接受这个取舍。

### 10.6 CSRF / CORS

- 前后端同源（CloudFront），不配置 CORS。
- Cookie `SameSite=Lax`；跨站 POST 不带 Cookie。
- 所有写接口检查 `Origin` 与 `APP_BASE_URL`；不匹配 403。
- 要求 `Content-Type: application/json`；简单表单跨站提交不会带这个类型。
- 第一版不做 CSRF token；残余风险记录在 18.3。若以后加跨域前端，再上 token。

### 10.7 IAM 最小权限

Lambda 角色只允许：

- `dynamodb:GetItem`、`PutItem`、`UpdateItem`、`DeleteItem`、`Query`、`ConditionCheckItem`（按表 ARN）；事务写由这些底层动作授权。
- **不允许 `Scan`**。
- `sqs:SendMessage`（API Lambda）；`sqs:ReceiveMessage`、`DeleteMessage`、`ChangeMessageVisibility`（email worker）。
- `ses:SendEmail`（worker；API Lambda 不需要）。
- `secretsmanager:GetSecretValue`（指定 secret ARN）。
- `logs:CreateLogStream`、`PutLogEvents`。
- `xray:PutTraceSegments`、`PutTelemetryRecords`。

Web 栈：S3 桶 `BlockPublicAccess.BLOCK_ALL`；CloudFront OAC；Bucket policy 只允许该 distribution。

### 10.8 日志与隐私

- 结构化 JSON：`level`、`msg`、`requestId`、`route`、`status`、`durationMs`、`userId`（可选）。
- 不记录：密码、验证码、Cookie、Authorization、完整 `mfaToken`、SES 正文。
- 错误日志可带内部错误码；对外只返回统一文案。
- CloudWatch 保留 14 天；生产可调 30 天。
- IP 只存 hash；User-Agent 截断。

### 10.9 其他安全取舍

- 注册 409 会暴露“邮箱已注册”；这是可用性取舍，登录/找回不暴露。
- 锁定 423 会暴露“账号存在”；用 IP 限流和统一文案降低，第一版接受。
- 没有 WAF；API Gateway 限流 + 应用限流 + 锁定是 v1 的防线；WAF 月费 $5+，不符合 <$5 目标。
- 没有审计表；安全事件靠 CloudWatch 日志和告警。后续可加 `AuditEvents` 表。
- API Gateway 的 URL 本身是公网可访问的；CloudFront 是规范入口，但无法完全阻止直连。第一版接受；若要封，需 WAF 或 REST API resource policy。

---

## 11. 邮件与异步

### 11.1 流程

1. API 生成码，写 `VerificationCodes`。
2. API 向 SQS 发消息：
   ```json
   { "to": "a@example.com", "purpose": "verify_email", "code": "123456", "codeId": "..." }
   ```
3. email worker（Lambda）消费：
   - 校验消息字段。
   - 调 SES `SendEmail`（纯文本）。
   - 成功返回；失败抛错，SQS 重试。
4. 重试 3 次后进 DLQ；CloudWatch alarm 通知。

### 11.2 为什么用 SQS

- 登录/注册请求不等待 SES；SES 抖动不影响 API 延迟。
- 自动重试；DLQ 保留失败消息。
- 以后可以换模板、加退信处理，不影响 API。

### 11.3 失败语义

- 入队失败：API 返回 503 `EMAIL_QUEUE_UNAVAILABLE`；用户记录/验证码可能已写入；用户可用“重发”再试。
- 入队成功但 SES 失败：API 仍返回 201/200；用户没收到邮件；可重发；DLQ 告警让运维知道。
- 重复投递：SQS 至少一次；用户可能收到重复邮件；可接受。worker 可用 `codeId` 做幂等标记，但第一版不做强去重。
- 本地：`EMAIL_DRIVER=console` 直接打印，不入 SQS。

### 11.4 SES 配置

- 验证发件域名（或邮箱）；配置 DKIM。
- 沙盒模式只能发给已验证地址；上线前申请生产访问。
- 纯文本邮件；`From: no-reply@<domain>`；`Reply-To` 可留空。
- 退信/投诉：第一版只告警，不做自动封禁。
- 发件额度：SES 默认有速率/日限额；低流量够用。

---

## 12. 前端

### 12.1 路由

```text
/                     → Navigate to /app（或公开 Landing，待定）
/login                → Login
/register             → Register
/verify-email         → VerifyEmail
/resend-verification  → ResendVerification
/forgot-password      → ForgotPassword
/reset-password       → ResetPassword
/mfa                  → MfaVerify（需要 location.state.mfaToken）
/app                  → RequireAuth → AppHome
/app/settings         → RequireAuth → Settings
*                     → NotFound
```

### 12.2 状态

- `AuthProvider`：`user`、`loading`、`refresh()`、`logout()`。
- 启动时 `GET /api/auth/me`；401 视为未登录。
- `login()` 返回 `{ user }` 或 `{ mfaRequired, mfaToken }`；后者由 Login 页 `navigate('/mfa', { state: { mfaToken } })`。
- `mfaToken` 只放 React state / `location.state`，**不放 localStorage/sessionStorage**；刷新 `/mfa` 丢失 token 就回 `/login`。
- 任意 API `401`：`AuthProvider` 清空 user，跳 `/login`，带 `from`。
- 不在前端存密码、验证码、会话 token。

### 12.3 页面

| 页面 | 内容 |
|---|---|
| Login | 邮箱、密码、显示/隐藏、登录、注册链接、忘记密码 |
| Register | 邮箱、密码、确认密码、注册、登录链接 |
| VerifyEmail | 邮箱（只读/可改）、6 位码、验证、重发 |
| ResendVerification | 邮箱、重发；统一提示“如果存在，已发送” |
| ForgotPassword | 邮箱、发送；统一提示 |
| ResetPassword | 邮箱、码、新密码、确认 |
| MfaVerify | 6 位码、验证、重发、回登录 |
| AppHome | 占位：登录成功；以后放博客/项目介绍 |
| Settings | MFA 开关；开启/关闭流程；当前密码 |

### 12.4 视觉

- 参考 write.as 的浅色、窄栏、安静风格；不复制 ccCode 深色登录页。
- 最大宽度 640px；登录卡片居中；正文可用 serif，表单用 sans。
- 颜色：近黑文字、浅灰背景、一个低饱和强调色；不做渐变/玻璃拟态。
- 响应式：手机单列；按钮全宽。
- 可访问性：`label` 绑定输入；错误 `role="alert"`；焦点可见；按钮 loading/disabled；`autocomplete` 正确。

### 12.5 公开首页（待定）

如果目标是推广，纯登录门会让第一次来的人不知道这是什么。建议：

```text
[Logo/名字]
一句话：一个用于演示登录、邮箱验证和可选 MFA 的小项目。
[登录] [注册]
```

- `/` 公开；`/app` 仍受保护。
- 这会改变“第一版纯门”的决定，所以标为待确认（18.1）。

---

## 13. 关键决策与理由

| 决策 | 选择 | 理由 |
|---|---|---|
| 认证 | 自建，不用 Cognito | 项目目标就是亲手实现认证 |
| 计算 | Lambda + API Gateway | 低流量便宜；不用管服务器 |
| 数据库 | DynamoDB 4 表 | 访问模式简单；TTL/原子计数/条件写合适；无 GSI |
| 会话 | 不透明 token + Cookie | 可撤销；不把 JWT 放 localStorage |
| MFA | 可选邮箱码 | 不用装 App；与找回密码共用发码链路 |
| 发信 | SQS + Lambda + SES | 异步、重试、DLQ；API 不被 SES 拖慢 |
| 前端托管 | S3 + CloudFront | 静态、便宜、HTTPS、同源 `/api` |
| 限流 | DynamoDB 原子计数 | Lambda 多实例下共享状态 |
| 锁定 | 5/10 次，15 分钟 | 简单、可解释；短锁降低 DoS |
| IaC | CDK TypeScript | 类型安全；和 AWS 资源贴合 |
| CI/CD | GitHub OIDC | 不存长期 AWS 密钥 |
| 区域 | `ap-southeast-1` | 亚洲延迟；成本适中 |

---

## 14. 验收标准

### 14.1 本地

- [ ] `npm run dev` 同时启动 web + api；DynamoDB Local 可用。
- [ ] 注册 → console 看到验证码 → 验证 → 登录 → `/app`。
- [ ] 未验证邮箱登录被 403 拦截。
- [ ] 重复注册 409；`/resend-verification` 可重发。
- [ ] 忘记密码 → console 码 → 重置 → 旧会话 401 → 新密码登录。
- [ ] 开启 MFA → 退出 → 登录出现 MFA → 输入码进入。
- [ ] 关闭 MFA → 再登录直接进入。
- [ ] 密码错 5 次 → 423；MFA 错 10 次 → 423。
- [ ] 登录/注册/发码超限 → 429 + `Retry-After`。
- [ ] Cookie 有 HttpOnly/SameSite=Lax；生产构建有 Secure。
- [ ] 前端任何 401 都回登录页。
- [ ] `grep` 确认没有 `Scan`、没有明文验证码、没有 localStorage token。

### 14.2 AWS 部署

- [ ] `cdk deploy --all` 成功；输出 CloudFront 域名。
- [ ] 自定义域名 + ACM 证书生效；HTTPS 可访问。
- [ ] S3 桶私有；只能通过 CloudFront 访问。
- [ ] `/api/health` 200；`/api/auth/me` 未登录 401。
- [ ] 注册邮件真的通过 SES 发出（生产访问已批）。
- [ ] CloudWatch 有日志；X-Ray 有 trace。
- [ ] DLQ 告警、Lambda 错误告警、5xx 告警配置。
- [ ] Budget 告警 $5；月底账单在目标内。
- [ ] GitHub Actions 能部署；OIDC 不需要长期密钥。

### 14.3 安全

- [ ] 密码/验证码/会话 token 不明文落库。
- [ ] 日志无敏感信息。
- [ ] IAM 无 `*`，无 `Scan`。
- [ ] 所有写接口校验 Origin/Content-Type。
- [ ] MFA token 过期/签名错/版本错都 401。
- [ ] 重置密码后旧会话全部失效。
- [ ] 限流/锁定在并发下有效（至少 10 并发测试）。

---

## 15. 测试计划

### 15.1 单元

- `tokenService`：签名/验签/过期/篡改/版本不匹配。
- `codeService`：生成、HMAC、过期、尝试次数、单次消费。
- `rateLimitService`：窗口计算、原子计数、超限。
- `lockoutService`：5/10 阈值、成功清零、重置清零。
- `authService`：假哈希、未验证拦截、MFA 分支。

### 15.2 集成（DynamoDB Local + supertest）

1. 注册 → 验证 → 登录 → me → logout。
2. 重复注册 409；重发 200；验证码错误 400。
3. 密码错 5 次 423；锁定期间正确密码也 423。
4. MFA 开启 → 登录 challenge → 验证 → 会话；MFA 错 10 次 423。
5. 重置密码 → 旧会话 401；新密码登录；锁定清零。
6. 限流 429；`Retry-After` 存在。
7. 验证码复用 400；过期 400。
8. Cookie 属性；响应体不含 token。
9. 同一验证码并发消费只有一个成功（条件写）。
10. 注册并发只有一个成功（条件写）。

### 15.3 前端

- Testing Library：登录成功/失败、MFA 流程、401 跳转、表单校验。
- 路由守卫：未登录访问 `/app` 跳 `/login`；登录后回跳。
- 不在 localStorage/sessionStorage 写 token。

### 15.4 E2E（Playwright）

- 注册 → 验证（测试环境读 DB/console）→ 登录 → 退出。
- 开启 MFA → 退出 → 登录 → MFA → 设置关闭。
- 忘记密码 → 重置 → 旧页面下一次请求退出。

### 15.5 负载

- k6：50 VU 登录 1 分钟；观察 p95、429、Lambda 并发。
- 冷启动：部署后首次请求；记录 p95。
- 限流并发：同一 IP 100 并发登录，确认不会超过限额太多。

### 15.6 CDK

- `cdk synth` 无错误；`cdk diff` 审查。
- 可选 `cdk-nag` 检查公开桶/宽 IAM。
- 快照测试资源名/关键属性（可选）。

---

## 16. 部署与运维

### 16.1 前置

- AWS 账号；区域 `ap-southeast-1`；预算告警。
- 域名（Route 53 或外部 registrar）；ACM 证书（CloudFront 用 `us-east-1`）。
- SES 验证发件域名；申请生产访问。
- GitHub 仓库；OIDC provider + deploy role。
- 本地 AWS CLI 配置；不要用 root 账号日常部署。

### 16.2 首次部署

```bash
cd login-app
npm ci
npm run build
cd infra
npx cdk bootstrap
npx cdk deploy --all -c env=dev
```

- 记录 `WebStack` 输出的 CloudFront 域名。
- 在 Route 53 加别名记录；等 ACM 证书 ISSUED。
- 手动验证 SES；发一封测试邮件。
- 打开 `/api/health` 和首页。

### 16.3 前端发布

```bash
npm run build -w web
aws s3 sync web/dist s3://<bucket> --delete
aws cloudfront create-invalidation --distribution-id <id> --paths "/*"
```

- CI 里用同一套命令；S3 桶私有，OAC 读取。
- 静态资源带 hash，`index.html` 不缓存；CloudFront 默认缓存策略可调。

### 16.4 回滚

- Lambda/API：CDK 回滚到上一个 stack；或 `cdk deploy` 指定旧版本。
- 前端：S3 版本控制 + 重新 sync 旧 build；CloudFront invalidation。
- 数据库：DynamoDB PITR（Users）；代码/验证码表不需要回滚。
- 第一版不做蓝绿/灰度；低流量可接受短暂中断。

### 16.5 备份

- Users 开 PITR；保留 7 天（按成本调整）。
- Sessions/Codes/RateLimits 不需要备份；TTL 数据。
- 可选：每日导出 Users 到 S3（cdk 定时 Lambda）；第一版可不做。
- Secrets Manager 密钥备份/轮换：手动；记录在 runbook。

### 16.6 Runbook（最小）

- 登录 5xx 升高：看 Lambda 错误、DynamoDB 限流、API Gateway 5xx。
- 用户收不到邮件：看 SQS 深度、DLQ、SES 退信/沙盒、worker 日志。
- 账单异常：看 Cost Explorer、Lambda 调用、CloudFront 流量、DynamoDB 读写。
- 账号被锁：等 15 分钟，或管理员在 DynamoDB 清 `locked_until`（谨慎）。
- 密钥泄露：轮换 Secrets Manager 密钥；所有 `mfaToken` 失效；必要时 bump `session_version`。

---

## 17. 成本估算

低流量（每月几百注册、几千登录、几万请求）估算，单位美元：

| 服务 | 估算 | 说明 |
|---|---|---|
| Lambda | $0–0.50 | 免费额度覆盖大部分；按请求/GB-s |
| API Gateway HTTP API | $0–0.50 | $1/百万请求；免费额度 12 个月 |
| DynamoDB On-Demand | $0–0.50 | 读写很少；存储 <1 GB |
| S3 | $0–0.20 | 前端几 MB |
| CloudFront | $0–0.50 | 免费额度 1 TB/10M 请求 |
| SES | $0–0.10 | $0.10/千封 |
| SQS | $0–0.10 | 免费额度覆盖 |
| Secrets Manager | $0.40 | 1 个 secret |
| Route 53 | $0.50 | 1 个 hosted zone |
| CloudWatch | $0–0.50 | 日志/告警；保留 14 天 |
| ACM | $0 | 公有证书免费 |
| **合计** | **约 $1–3/月** | 不含域名 |

- 域名：约 $10–15/年。
- WAF：$5/月起 + 请求费；第一版不开；需要时再加。
- 预算告警：AWS Budgets $5/月；第一版 2 个预算免费。
- 价格随区域/汇率变化；部署前用 AWS Pricing Calculator 复核。
- 最大不确定项：CloudFront 流量、CloudWatch 日志量、DynamoDB 意外 Scan（已禁止）。

---

## 18. 风险与未决问题

### 18.1 待用户确认（会影响首屏/推广）

1. **公开首页还是纯登录门？**
   - 现状：`/` 直接跳 `/app`，未登录再跳 `/login`。
   - 推广角度：纯门没有解释，第一次来的人不知道这是什么；建议 `/` 放一句话 + 登录/注册。
   - 这是产品决策，未定。

2. **域名**：有没有已注册域名？用 Route 53 还是外部 DNS？
3. **区域**：默认 `ap-southeast-1`；如果主要用户在中国大陆，延迟/备案/访问需要再评估（AWS 全球区不需要备案，但跨境延迟存在）。
4. **SES 生产访问**：是否已有域名/发件邮箱？沙盒只能发已验证地址。
5. **环境**：只做 `dev`，还是 `dev` + `prod` 两套？
6. **WAF**：第一版不开（$5/月）；是否接受？
7. **GitHub**：仓库地址/分支；是否用 OIDC 部署。

### 18.2 技术风险

- **冷启动**：登录 p95 可能 1–2 秒；用 512MB、ARM、保持包小；必要时 Provisioned Concurrency（成本上升）。
- **DynamoDB 建模**：4 表无 GSI 是刻意的；如果以后要“按用户查会话/审计”，需要加 GSI 或改表。
- **TTL 延迟**：过期项可能还在；代码必须检查时间。
- **SES 沙盒/退信**：上线前必须申请生产；退信率高会被限。
- **CloudFront/API Gateway 超时**：Lambda 10 秒；不要让 API 等 SES。
- **限流键**：IP 可能被 NAT/代理共享；邮箱/IP 双键降低误伤。
- **锁定 DoS**：攻击者可锁别人账号；短锁 + IP 限流缓解，不能根治。
- **无审计表**：安全事件只能看日志；后续可加。
- **无 CSRF token**：同源 + SameSite=Lax + Origin 校验；残余风险接受。
- **API Gateway 直连**：CloudFront 是规范入口，但 API URL 公网可达；第一版接受。

### 18.3 明确接受的风险

- 注册 409 暴露邮箱已注册。
- 锁定 423 暴露账号存在。
- 没有 WAF、没有验证码图片、没有设备指纹。
- 邮件可能重复/延迟；用户可重发。
- 单区域；区域故障会不可用。
- 没有管理员 UI；MFA 邮箱丢失靠控制台处理。

---

## 19. 交给构建者的部分

以下不需要再问产品决策，构建者可以自行决定：

- CSS 具体色值/字号/间距；只要保持 write.as 式浅色窄栏。
- 文案微调；错误提示统一、简短。
- 测试数据、fixture、mock 邮件。
- DynamoDB Local 建表脚本细节。
- CDK 资源命名/告警阈值微调（成本内）。
- SES 邮件正文措辞。
- README/ADR/Runbook 的具体结构。

需要用户/环境提供：

- AWS 账号、区域、预算确认。
- 域名/DNS。
- SES 发件域名/生产访问。
- GitHub 仓库/OIDC。
- 公开首页是否要做（18.1）。

---

## 附录 A：AWS 服务地图

| 服务 | 在本项目中的角色 | 为什么不用替代 |
|---|---|---|
| CloudFront | HTTPS、自定义域名、静态站点 + `/api/*` 同源 | 不用自己管 Nginx/证书 |
| S3 | 存 React build；私有 + OAC | 静态托管便宜 |
| API Gateway HTTP API | HTTP 入口、路由、阶段限流 | 比 REST API 便宜；够用 |
| Lambda | 无状态 API + 邮件 worker | 按请求付费；不用管服务器 |
| DynamoDB | 用户/会话/验证码/限流 | TTL、原子计数、条件写；不用 RDS/VPC |
| SES | 发验证码邮件 | 托管邮件；按量 |
| SQS | 异步发信、重试、DLQ | 不让 API 等 SES |
| Secrets Manager | 存 HMAC 密钥 | 不把密钥放环境变量/仓库 |
| CloudWatch | 日志、指标、告警 | 原生集成 |
| X-Ray | 追踪 API→Lambda→DynamoDB/SQS | 定位冷启动/下游延迟 |
| IAM | Lambda/S3/CloudFront 权限 | 最小权限 |
| ACM | CloudFront 证书 | 免费、自动续期 |
| Route 53 | 域名/别名 | 和 CloudFront/ACM 集成 |
| CDK | 基础设施即代码 | 类型安全、可 review |
| GitHub Actions + OIDC | CI/CD | 不存长期 AWS 密钥 |

---

## 附录 B：MFA/2FA 与参考项目

- **MFA** 是多因素认证的总称；**2FA** 是恰好两个因素。本项目做的是“密码 + 邮箱码”的 2FA，属于 MFA 的一种。
- 参考项目 `Two_Factor_Auth` 做的是：
  - 邮箱验证：20 位 token + 链接，存 `services` 表。
  - 2FA：6 位数字码，SES/SNS 发送，查最近 2 天记录并比对过期时间。
  - 架构：API Gateway + Lambda（Python）+ RDS PostgreSQL + SES/SNS。
- 它不是完整登录系统：没有注册、密码、会话/JWT、受保护资源、前端；2FA 假设调用方已被信任。
- 可借鉴：发码/验码/过期/重发的行为；SES 配置；API Gateway+Lambda 接线；DynamoDB TTL 的替代思路。
- 不要照搬：SQL 字符串拼接、明文存码、创建接口返回 token、公开 RDS、仓库里的 AWS 密钥。
- 本项目用 DynamoDB + HMAC + 条件写重做，安全性和可部署性都更好。

---

## 附录 C：术语

| 术语 | 含义 |
|---|---|
| MFA | 多因素认证；密码 + 邮箱码是其中一种 |
| 2FA | 两因素认证 |
| TTL | DynamoDB 的过期删除属性；不是即时删除 |
| OAC | CloudFront Origin Access Control；让 S3 只允许 CloudFront 读 |
| OIDC | GitHub Actions 用它换 AWS 临时凭证，不存长期密钥 |
| DLQ | Dead Letter Queue；重试失败的消息 |
| PITR | DynamoDB Point-in-Time Recovery |
| 冷启动 | Lambda 首次/缩容后初始化带来的额外延迟 |
