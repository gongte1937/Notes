## 一句话总览（TL;DR）

> **这是一个以 Supabase Auth 为身份源，前端主导登录、Redux + Cookie 做状态编排、业务表自行初始化的多登录方式认证体系。**

---

## 1️⃣ Auth 架构分层（非常清晰）

### Client-side（浏览器）

- 使用 `@supabase/supabase-js`
    
- Key：
    
    - `NEXT_PUBLIC_SUPABASE_URL`
        
    - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
        
- 职责：
    
    - 发起登录
        
    - 处理 OAuth / Magic Link
        
    - 监听 auth state
        
    - 驱动 UI & Redux
        

📍 文件核心：

- `helper.tsx`
    
- `page.tsx`
    
- `auth.tsx`
    

---

### Server-side / Script

- 使用 `SUPABASE_SERVICE_ROLE_KEY`（**不暴露给浏览器**）
    
- 用途：
    
    - cron / job（如 booking-expiry）
        
    - 后台绕过 RLS 的管理型操作
        

📍 示例：

- `booking-expiry.ts`
    

---

## 2️⃣ 登录方式（Login Flows）

你们支持 **4 + 1 + 1** 种方式 👇

### ✅ OAuth（Popup 模式）

- Google
    
- Microsoft / Azure
    
- GitHub
    
- LinkedIn OIDC
    

`supabase.auth.signInWithOAuth({   provider,   options: {     skipBrowserRedirect: true   } })`

**特点**

- Popup 登录（不整页跳转）
    
- 登录完成后靠 `onAuthStateChange` 收口
    

📍 `page.tsx`

---

### ✅ Magic Link（Email OTP）

`supabase.auth.signInWithOtp({   email,   shouldCreateUser: true,   emailRedirectTo: /verify })`

- 用于 Freelancer / Client
    
- `/verify` 页面使用 `verifyOtp` 完成最终登录
    
- 成功后根据角色跳转 profile
    

📍 `page.tsx`

---

### ⚠️ Dev-only Email / Password

`supabase.auth.signInWithPassword()`

- **仅 E2E / 本地**
    
- 生产环境禁用
    

📍 `page.tsx`

👉 这个点你们做得很好，安全边界很清楚。

---

## 3️⃣ Auth State & Redirect 机制（核心 glue）

### 监听 Supabase Auth 状态

`supabase.auth.onAuthStateChange(...)`

用途：

- 识别 popup OAuth 登录完成
    
- 触发 redirect（profile / dashboard）
    
- 同步 Redux 状态
    

📍 `page.tsx`

---

### Magic Link 校验

- `/verify` 页面：
    

`supabase.auth.verifyOtp(...)`

- 成功 → `/freelancer/profile` 或 `/client/profile`
    

📍 `page.tsx`

---

## 4️⃣ Session / User 初始化（最关键的一段）

### 页面加载时（Bootstrapping）

- HOC：
    
    - 读 cookie
        
    - 检查 Redux
        
    - 拉 Supabase session / user
        

📍

- `auth.tsx`
    
- `userReducer.ts`
    

---

### 首次登录逻辑（Business Init）

- 如果是第一次登录：
    
    - 从 OAuth provider 拿 name / email
        
    - 插入你们自己的 `users` 表
        
    - 设置 cookies
        
    - 初始化 Redux
        

📍 `userReducer.ts`

👉 **这一步是你们“业务身份”的起点，不是 Supabase 自动做的**

---

### 业务规则（很重要）

- **Client magic link**
    
    - 禁止 free email domain
        
    - 校验失败 → Redux error → UI 提示
        

📍

- `page.tsx`
    
- `userReducer.ts`
    

---

## 5️⃣ Cookies 的角色（你们的“状态中枢”）

你们用 cookie + Redux 双轨并行：

|Cookie|用途|
|---|---|
|`user`|基础用户信息|
|`userType`|freelancer / client|
|`authProvider`|Google / LinkedIn 等|
|`isAuthPopUp`|popup 流程 guard|

📍 设置 / 使用位置：

- `userReducer.ts`
    
- `page.tsx`
    
- `helperFunctions.tsx`
    

👉 **Supabase 管 session，你们管“产品态身份”**

---

## 6️⃣ 登出（Sign Out）

流程非常干净：

1. `supabase.auth.signOut()`
    
2. 清 cookies
    
3. reload / redirect
    

📍

- `AuthButton.tsx`
    
- `helperFunctions.tsx`
    
- `sidebar.tsx`
    

---

## 7️⃣ 环境变量分组（很规范）

### 必须（前端）

- `NEXT_PUBLIC_SUPABASE_URL`
    
- `NEXT_PUBLIC_SUPABASE_ANON_KEY`
    

### Server-only

- `SUPABASE_SERVICE_ROLE_KEY`
    

### Auth UX / Infra

- `NEXT_PUBLIC_SITE_URL`
    
- `NEXT_PUBLIC_COOKIES_MAX_AGE`
    

---

## 8️⃣ 用一句“架构语言”帮你总结（给 Stefan / 文档用）

> The system uses Supabase as the authentication and session authority, with client-side initiated OAuth and magic-link flows, Redux and cookies for application-level identity orchestration, and explicit business-level user initialization decoupled from Supabase’s `auth.users`.