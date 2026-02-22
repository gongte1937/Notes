# PRD：Phase 1 — NestJS Authentication Skeleton

## 1. 背景与问题

当前系统使用 Supabase Auth（`@supabase/supabase-js`）在前端完成：

- OAuth（Google/Azure/GitHub/LinkedIn OIDC）popup 登录
    
- Magic Link OTP + `/verify` 验证
    
- `onAuthStateChange` 驱动重定向
    
- Cookie + Redux 做 app-level 身份编排（`userType/authProvider/isAuthPopUp/user`）
    
- 首次登录写入业务 `users` 表
    

迁移目标是：**认证（Authentication + Session）迁到 NestJS**，但 **数据库与存储仍留在 Supabase**。

Phase 1 的意义：

> 先在 NestJS 建立“可扩展、可上线”的认证骨架（Auth 模块、Token 策略、会话 cookie、用户 provision 基础能力、/auth/me），为 Phase 2（Magic Link 切换）与 Phase 3（OAuth 切换）做地基。

---

## 2. 目标（Goals）

Phase 1 必须交付：

1. **NestJS Auth 模块骨架**
    

- 模块拆分清晰：Controller/Service/Guard/Strategy
    
- 可支撑 OAuth + MagicLink（Phase 2/3 实现细节可后置）
    

2. **会话体系（Session）可用**
    

- Access token + Refresh token 策略确定并实现
    
- HttpOnly cookie 下发、刷新、登出逻辑框架完成
    
- `/auth/me` 可以识别当前用户
    

3. **User Provisioning 基础能力**
    

- 支持“通过 identity（provider/email）找到/创建业务用户（users 表）”
    
- 支持绑定多个 identity（后续多 provider 合并策略）
    

4. **配置、日志、错误码、测试基线**
    

- env 配置规范
    
- auth 相关 audit log & 安全日志结构
    
- 基础 e2e 测试/单测骨架
    

---

## 3. 非目标（Non-goals）

Phase 1 不做（明确避免 scope creep）：

- ❌ 不切换前端登录流量（仍然用 Supabase Auth 跑生产）
    
- ❌ 不实现完整 OAuth provider 流程（Phase 3）
    
- ❌ 不实现 Magic link 发信+校验（Phase 2）
    
- ❌ 不大规模改造“业务数据访问从前端 supabase.from → Nest API”（Phase 4）
    
- ❌ 不重做 Supabase RLS（Phase 1 只做策略建议，不改生产策略）
    

---

## 4. Stakeholders

- 工程：你（全栈），Stefan/后端负责人
    
- 安全/合规（如果有）：对 cookie、token、审计日志有要求
    
- 产品：希望保持现有 UX（popup OAuth、magic link）
    

---

## 5. 用户故事（User Stories）

Phase 1 以“内部可用”为主：

1. 作为开发者，我希望能在本地启动 NestJS 后，通过伪造 session 或 dev endpoint 获取 `/auth/me` 成功返回用户信息，便于迭代后续登录流程。
    
2. 作为系统，我希望能安全地下发 refresh token 并支持轮换/撤销，为后续上线提供安全基础。
    
3. 作为系统，我希望能把一个 provider identity 映射到业务 users 表，确保后续所有业务授权基于统一 user_id。
    

---

## 6. 需求与约束（Requirements & Constraints）

### 6.1 技术栈约束

- NestJS（建议：Passport + JWT）
    
- 数据库仍为 Supabase Postgres
    
- NestJS 访问 DB：优先使用 **Postgres 直连（推荐）** 或 Supabase client（service role）
    
    - 推荐直连原因：避免把业务强绑定 Supabase SDK；但如果你们已大量用 Supabase SDK，也可以先用 service role client 快速落地。
        

### 6.2 安全约束

- Refresh token 必须：
    
    - HttpOnly cookie
        
    - server-side 可撤销（DB 存 hash）
        
    - 支持轮换（rotation）
        
- Access token 短期有效（10–15min）
    
- CSRF：cookie-based session 需要 CSRF 策略（见安全章节）
    

---

## 7. 方案概览（High-level Solution）

Phase 1 输出一个 Auth 系统骨架：

- `AuthController`
    
    - `GET /auth/me`
        
    - `POST /auth/refresh`
        
    - `POST /auth/logout`
        
    - （可选 dev）`POST /auth/dev/login`（仅本地/测试）
        
- `AuthService`
    
    - 签发 token、设置 cookie、清理 cookie
        
    - refresh rotation 逻辑
        
- `UserProvisioningService`
    
    - `findOrCreateUserByIdentity(...)`
        
    - `linkIdentityToUser(...)`
        
- `TokenService`
    
    - JWT sign/verify
        
    - refresh token hash/compare
        
    - rotation、revoke
        
- `AuthGuard`
    
    - `JwtAccessGuard`：解析 access token，注入 `req.user`
        
- 数据表（在 Supabase DB 内）
    
    - `users`
        
    - `auth_identities`
        
    - `refresh_tokens`
        

> Phase 2/3 将分别补齐 MagicLinkService / OAuthService，而不会推翻 Phase 1 的结构。

---

## 8. 数据模型（Schema）

> 表都在 Supabase Postgres 中创建/维护。

### 8.1 users（业务用户）

最小字段建议：

- `id` (uuid, pk)
    
- `email` (text, nullable，OAuth 不一定有 verified email？但通常有)
    
- `display_name` (text, nullable)
    
- `user_type` (enum: `freelancer|client`, nullable；Phase 1 可先 nullable)
    
- `created_at`, `updated_at`
    

### 8.2 auth_identities（身份绑定）

支持一个 user 绑定多个 provider：

- `id` (uuid pk)
    
- `user_id` (uuid fk → users.id)
    
- `provider` (text) — `google|azure|github|linkedin_oidc|email`
    
- `provider_subject` (text) — OIDC `sub` / provider user id
    
- `email` (text, nullable)
    
- `email_verified` (bool, default false)
    
- `created_at`
    

唯一约束：

- unique(provider, provider_subject)
    
- 可选 unique(lower(email), provider) 用于 email identity（看你们合并策略）
    

### 8.3 refresh_tokens（可撤销的 refresh）

- `id` (uuid pk)
    
- `user_id` (uuid fk)
    
- `token_hash` (text) — 存 hash，不存明文
    
- `family_id` (uuid) — 用于 rotation 家族追踪（推荐）
    
- `rotated_from` (uuid nullable) — 指向上一个 token id
    
- `revoked_at` (timestamptz nullable)
    
- `expires_at` (timestamptz)
    
- `created_at`
    
- `ip` / `user_agent`（可选，审计）
    

---

## 9. API 设计（Phase 1 必须实现）

> 统一前缀：`/auth`

### 9.1 GET /auth/me

**目的**：验证会话与返回当前用户。Phase 2/3 前端成功登录后都靠它初始化 Redux。

- Auth：需要 access token（cookie 或 Authorization header）
    
- Response 200：
    

`{   "user": {     "id": "uuid",     "email": "x@y.com",     "displayName": "Alice",     "userType": "freelancer"   },   "identities": [     { "provider": "google", "email": "x@y.com" }   ],   "session": {     "expiresAt": "ISO"   } }`

- Response 401：未登录/过期
    

### 9.2 POST /auth/refresh

**目的**：使用 refresh cookie 换取新的 access（以及 refresh rotation）。

- Auth：refresh token cookie
    
- 行为：
    
    - 校验 refresh token hash
        
    - 若已 revoked 或过期 → 清 cookie + 401
        
    - 否则：
        
        - 生成新 access token
            
        - **生成新 refresh token**（rotation）并 revoke 旧 token（或标记 rotated）
            
        - 下发新的 cookies
            
- Response 200：可返回简化 `{ ok: true }`
    

### 9.3 POST /auth/logout

**目的**：登出并撤销 refresh token（当前设备）。

- 行为：
    
    - revoke 当前 refresh token（若存在）
        
    - 清除 cookies
        
- Response 200：`{ ok: true }`
    

### 9.4 （可选，仅开发/测试）POST /auth/dev/login

**目的**：Phase 1 在没有 OAuth/MagicLink 的情况下也能联调后续页面。

- 输入：email/userType
    
- 行为：findOrCreateUser + 下发 cookies
    
- 仅允许 `NODE_ENV=development` 或显式开关
    

---

## 10. Token 与 Cookie 策略（关键）

### 10.1 Token

- Access JWT：
    
    - TTL：10–15 分钟
        
    - Claims：
        
        - `sub` = users.id
            
        - `email`、`userType`（可选）
            
        - `jti`（可选）
            
- Refresh：
    
    - TTL：7–30 天（建议先 14 天）
        
    - 随机高熵 token（至少 32 bytes）
        
    - 只存 hash 到 DB
        
    - rotation：每次 refresh 发新 refresh
        

### 10.2 Cookies

建议 cookie 名称：

- `tb_at`（access token）
    
- `tb_rt`（refresh token）
    

属性建议：

- `HttpOnly: true`
    
- `Secure: true`（本地可通过配置关闭）
    
- `SameSite`：
    
    - 如果前后端同域：`Lax` 即可
        
    - 如果 OAuth popup/跨站较多：可能需要 `None`（必须 Secure）
        
- `Path`：
    
    - refresh 可设 `Path=/auth/refresh` 限制发送面（加分项）
        

---

## 11. 鉴权与 Guard

### 11.1 JwtAccessGuard

- 从 cookie 或 Authorization header 读取 access token
    
- 验证 JWT
    
- 注入 `req.user = { id, email, userType }`
    
- 供 `/auth/me` 与未来业务 endpoints 使用
    

### 11.2 权限模型（Phase 1 只定义框架）

- Phase 4 业务 API 迁移时使用：
    
    - `@UseGuards(JwtAccessGuard)`
        
    - owner check / role check（后续 Policy/Ability）
        

---

## 12. 与 Supabase 的集成方式（仍保留数据库）

Phase 1 推荐两种实现路径（二选一，建议 A）：

### A) Nest 直连 Supabase Postgres（推荐）

- 使用 `pg` / Prisma / TypeORM 连接 Supabase 的 Postgres
    
- 优点：代码更可移植、不过度依赖 Supabase SDK
    
- 缺点：需要维护连接池
    

### B) Nest 使用 Supabase service role client

- `createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY)`
    
- 优点：上手快
    
- 缺点：容易把业务继续绑定 Supabase SDK；而且“绕过 RLS”的能力更强，必须谨慎控制
    

> 无论哪种，service role 只能在 server-side。

---

## 13. 日志与审计（Logging & Audit）

必须记录（至少）：

- refresh token 使用（成功/失败）
    
- refresh token rotation（旧 token id → 新 token id）
    
- logout（撤销 token）
    
- 异常：token 重放、family 异常（同 family 出现并行刷新）
    

日志字段建议：

- user_id
    
- action（REFRESH_SUCCESS / REFRESH_REUSED / LOGOUT）
    
- ip
    
- user_agent
    
- timestamp
    
- request_id（如果你们有）
    

---

## 14. 错误处理规范

定义统一错误码（便于前端识别）：

- `AUTH_UNAUTHORIZED`
    
- `AUTH_REFRESH_EXPIRED`
    
- `AUTH_REFRESH_REVOKED`
    
- `AUTH_INVALID_TOKEN`
    
- `AUTH_DEV_LOGIN_DISABLED`
    

返回结构建议：

`{ "code": "AUTH_UNAUTHORIZED", "message": "Unauthorized" }`

---

## 15. 配置（Environment Variables）

必须：

- `JWT_ACCESS_SECRET`
    
- `JWT_ACCESS_TTL_MINUTES`
    
- `REFRESH_TTL_DAYS`
    
- `COOKIE_SECURE`
    
- `COOKIE_SAMESITE`
    
- `COOKIE_DOMAIN`（如有）
    
- `DATABASE_URL`（直连 Postgres）或 `SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY`
    

可选：

- `ENABLE_DEV_LOGIN`（开发登录开关）
    

---

## 16. 测试计划（Phase 1）

### 单元测试

- TokenService：hash/compare、sign/verify
    
- AuthService：issue session、refresh rotation、logout revoke
    

### E2E 测试（最少 4 条）

1. dev login → `/auth/me` 200
    
2. access 过期 → `/auth/me` 401
    
3. refresh 成功 → 新 access 可用
    
4. logout → refresh 再用失败
    

---

## 17. 验收标准（Acceptance Criteria）

Phase 1 完成的“可验收清单”：

-  NestJS 启动后，存在 `/auth/me /auth/refresh /auth/logout`
    
-  能通过 dev login（或内部方式）下发 cookie，并成功访问 `/auth/me`
    
-  refresh rotation 生效：旧 refresh 被标记 revoked/rotated，新的 refresh 可用
    
-  logout 会撤销当前 refresh 并清 cookie
    
-  数据库中存在 users/auth_identities/refresh_tokens 表（或等价结构），并且 refresh 明文不落库
    
-  基础日志可追踪 refresh/rotation/logout
    
-  e2e 测试在 CI 或本地可跑通
    

---

## 18. 里程碑与交付物

### Deliverables

- NestJS `AuthModule` 完整代码
    
- DB migrations（SQL 或 migration tool）
    
- Postman/Insomnia collection（或 Swagger）
    
- E2E 测试脚本
    

### Milestone

- M1：/auth/me + JwtAccessGuard 可用
    
- M2：refresh rotation + logout 完成
    
- M3：dev login + e2e 覆盖完成