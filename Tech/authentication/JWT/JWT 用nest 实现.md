
## 第 1 阶段：把“身份验证”补上（别把 login 当后门）

目标：`/auth/login` 真正验证用户是谁

可选路线（选一种）：

- A) 用户名密码（bcrypt 校验）
    
- B) 邮箱/短信 OTP
    
- C) OIDC / WorkOS（验证 id_token / code）
    

✅ 到这一步，你实现了：**签发前必须先“验人”**

---

## 第 2 阶段：Refresh Token（续期）

目标：Access Token 短命，过期后不需要重新登录

你要加：

- `POST /auth/refresh`
    
- Refresh token 存储策略（先简化版）：
    
    - 方案 1：DB 存 refresh token（推荐）
        
    - 方案 2：内存/redis（开发可）
        
- 客户端保存 refresh token（Web 推荐 HttpOnly cookie）
    

✅ 到这一步，你实现了：**续期**

---

## 第 3 阶段：企业级安全（轮换 + 防重放 + 撤销）

目标：refresh token 被偷也很难长期利用

你要加：

- **Rotation**：每次 refresh 都发新的 refresh token，旧的立即作废
    
- **Reuse detection**：旧 refresh token 再用 → 401（并可一键干掉整个 family）
    
- **Revocation**：logout 时标记 refresh token revoked
    
- DB 字段常见：`token_hash, family_id, rotated_from, revoked_at, expires_at, ip, user_agent`
    

✅ 到这一步，你实现了：**轮换 + 防重放 + 撤销**

---

## 第 4 阶段：Cookie 落地（Web 最常见的生产方案）

目标：减少 XSS 风险，降低 token 泄漏概率

建议：

- `tb_at`（access）HttpOnly，Path=/，短期
    
- `tb_rt`（refresh）HttpOnly，Path=/auth/refresh，长期
    
- `SameSite=Strict/Lax` + `Secure(生产)`
    
- 如果你用 cookie 方式带 access token：`JwtStrategy` 里改为从 cookie 读取
    

✅ 到这一步，你实现了：**浏览器端生产可用的存储策略**

---

## 第 5 阶段：权限、审计、多端会话

目标：多端登录、权限控制、可观测

你要加：

- RBAC/ABAC：`roles/scopes` + 自定义 `@Roles()` guard
    
- 多设备 session：一个 refresh token = 一个 device session
    
- 审计：登录/刷新/登出事件记录
    
- 风控：IP/UA 异常、refresh 频率限制、黑名单
    

---

## 第 6 阶段：SSO / WorkOS / OIDC（你后面一定会走到）

目标：外部身份源进来后，你内部仍用自己的 token/session 体系

你要做：

- 校验 `iss/aud/exp` + JWKS（公钥拉取/缓存/轮换）
    
- user provisioning（首次登录创建用户）
    
- 仍按你的 refresh/rotation/revocation 管理会话