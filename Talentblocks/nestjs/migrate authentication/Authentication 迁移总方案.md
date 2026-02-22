## 目标架构（最终形态）

- **NestJS**：认证中心（OAuth + Magic Link）、会话（JWT/refresh）、鉴权、业务 API
    
- **Supabase**：继续做数据库/存储（Postgres + Storage），**不再负责 Auth**
    
- **前端**：只跟 NestJS 交互（登录 + 业务 API），不依赖 `supabase.auth.*`
    

✅ 好处：不用再被 Supabase Auth / RLS 绑定；权限逻辑集中在 Nest，最可控。  
⚠️ 代价：前端直连 Supabase 的查询/写入要改成调用 Nest API（但这是换取安全与可维护性的关键）。

---

## Phase 0：安全基线先定下来（非常关键）

### 0.1 立规矩：哪些东西可以在前端出现？

- ✅ 前端可以出现：`NEXT_PUBLIC_SUPABASE_URL`（可选）、但**尽量不再用 anon key 访问业务表**
    
- ❌ 前端绝不能出现：`SUPABASE_SERVICE_ROLE_KEY`
    

### 0.2 目标安全策略（推荐）

- **业务表**：只允许 service role（也就是 NestJS）访问  
    （做法：RLS 全开 + policy 只放行 service role，或干脆不给 anon 访问权限）
    
- **Storage 上传/下载**：由 NestJS 生成 **signed upload url / signed download url** 或走 Nest 代理
    

> 产物：一份“哪些表/哪些 bucket 允许前端直连”的清单（一般业务表都不允许）

---

## Phase 1：NestJS 先把会话系统搭起来（不碰数据迁移）

### 1.1 建立 Auth 模块

- OAuth（Google/Azure/GitHub/LinkedIn OIDC）
    
- Magic Link（替代 Supabase OTP）
    
- Session：Access Token + Refresh Token（HttpOnly cookies）
    
- `/auth/me`：返回当前登录用户（以及 userType、profile completion 等）
    

### 1.2 数据表（仍然在 Supabase Postgres 里建）

建议新增/确认这些表（都在 Supabase 数据库中）：

- `users`（业务用户）
    
- `auth_identities`（user_id + provider + provider_subject + email）
    
- `magic_links`（token_hash + expires_at + used_at + email + userType）
    
- `refresh_tokens`（token_hash + user_id + revoked_at + rotated_from）
    

> 重点：**你数据仍在 Supabase**，只是由 NestJS 来读写这些表。

---

## Phase 2：先迁 Magic Link（最容易、最快验证闭环）

把你现在：

- `supabase.auth.signInWithOtp(...)`
    
- `/verify` + `supabase.auth.verifyOtp(...)`
    

替换为 NestJS：

### 新流程

1. `POST /auth/magic-link`（email + userType）
    

- 在 NestJS 做 **client free email domain block**（你现在 Redux 那段规则迁过来）
    
- 生成一次性 token（hash 存 DB）
    
- 发邮件（链接指向 NestJS 的 verify）
    

2. `GET /auth/verify?token=...`
    

- 校验 token、标记 used
    
- 创建/查找 user + identity
    
- 下发 session cookies
    
- 重定向到 `/freelancer/profile` 或 `/client/profile`
    

✅ 验收：不依赖 Supabase Auth，也能完成登录、落 cookie、`/auth/me` 能拿到用户。

---

## Phase 3：迁 OAuth（保持你现在的 popup 体验）

你现在是 `skipBrowserRedirect: true` + `onAuthStateChange` 收口。

在 NestJS 模式，建议改为标准 “popup + postMessage”：

### 新流程

1. 前端打开 popup：`GET /auth/oauth/:provider/start`
    
2. provider 回调到：`GET /auth/oauth/:provider/callback`
    
3. callback 成功后：
    

- NestJS 完成 user/identity provision
    
- 设置 session cookies
    
- 返回一个极简页面，在 popup 里 `window.opener.postMessage({ ok: true }, origin)` 然后 close
    

4. 主窗口收到 message 后调用：`GET /auth/me` → 初始化 Redux → 跳转 profile
    

✅ 验收：Google/Azure/GitHub/LinkedIn 全部跑通，体验与现在一致。

---

## Phase 4：把“业务数据访问”收口到 NestJS（DB 仍在 Supabase）

这是你补充前提下**最重要的一步**：你现在很多地方是 `supabase.from(...).select/insert/update`，这些要逐步改成 Nest API。

### 迁移原则

- 前端：`fetch('/api/...')`
    
- NestJS：用 service role key 或直连 Postgres 操作 Supabase 数据库
    
- 权限：NestJS guard 负责（owner 校验、role 校验）
    

### 落地顺序（建议按风险/频率）

1. **用户资料写入**（你最敏感）
    
    - freelancer_personal_details / client_personal_details
        
2. **支付信息**（最敏感）
    
3. bookings / timesheets（业务核心）
    
4. skills / work history（相对低风险）
    
5. browse/search（读多写少，可最后）
    

> 你们原来的“RLS based on auth.uid()”这套会失效，所以**不要再让前端用 anon key 直接访问业务表**，否则权限会漏。

---

## Phase 5：Storage 策略（仍在 Supabase Storage）

你现在前端直接：

- `supabase.storage.from(bucket).upload(...)`
    
- `download(...)`
    

迁移后推荐两种方式（二选一）：

### A) Signed URL（推荐）

- `POST /files/upload-url` → NestJS 生成 signed upload url（限定路径/内容类型/有效期）
    
- 前端用 PUT 直接传到 Supabase Storage（但权限由签名控制）
    
- 下载同理：`GET /files/download-url?...`
    

✅ 好处：文件流不经过 NestJS，性能好

### B) 代理上传/下载

- 前端把文件传 NestJS
    
- NestJS 用 service role 上传到 storage
    

✅ 好处：逻辑简单  
⚠️ 坏处：NestJS 带宽压力大

---

## Phase 6：脚本/Job（booking-expiry 等）

继续保留在 server-side，但统一走：

- NestJS cron / worker
    
- 使用 service role key 访问 Supabase DB
    

把“散落脚本”逐步收编到一个 `JobsModule`，好管理、好监控。

---

## Phase 7：切流与下线 Supabase Auth（但保留 Supabase 数据）

### 切流策略

- feature flag：`AUTH_MODE=supabase | nest`
    
- 先 internal → 10% → 100%
    

### 数据关联（非常关键）

你现在 Supabase Auth 有 `auth.users.id`。迁移后建议：

- 在 `auth_identities` 里保留 `legacy_supabase_user_id`
    
- 首次 Nest 登录时把老 id 映射到新 user_id（避免你们业务表里历史关联断掉）
    

### 最终下线

- 关闭 Supabase Auth providers / OTP
    
- 清理前端所有 `supabase.auth.*` 逻辑
    
- Supabase 继续作为 DB/Storage 正常用
    

---

## 你现在这套流程里，“需要特别注意会踩坑的点”

1. **RLS 依赖 Supabase Auth 的 `auth.uid()`**：迁移后会空掉 → 必须改为 “Nest 统一访问”
    
2. **cookie/domain/samesite**：popup OAuth + 跨域非常容易翻车（尤其 Safari）
    
3. **同邮箱多 provider 合并规则**：要提前定（google 登录后再 linkedin 登录，是合并还是新建？）
    
4. **前端 cookie（user/userType/authProvider）**：建议逐步废掉，让 `/auth/me` 成为唯一真相