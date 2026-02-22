# Refresh Token Reuse Detection（重放攻击防护）笔记

## 背景

在使用 **JWT + Refresh Token** 的认证体系中，  
Refresh Token 通常具有 **较长的有效期**（如 7–30 天），并且具备 **续签 Access Token 的能力**。

因此：

> **Refresh Token 一旦泄露，本质上等同于身份被盗。**

---

## 常见误区

> ❌「Refresh token 有过期时间（expiresAt），被偷了也没关系」

这是错误的认知。

- `expiresAt` 只是**时间上限**
    
- 并不能防止 **有效期内的滥用**
    
- 任何在有效期内拿到 refresh token 的人，都可以使用它
    

---

## Token Rotation（轮换）机制回顾

标准的 refresh token 设计通常包含：

- **单次使用** refresh token
    
- 每次刷新：
    
    - 旧 token → 标记 `revokedAt`
        
    - 新 token → 生成（同一个 `familyId`）
        

```text
RT1 (valid)
↓ refresh
RT1 (revokedAt set)
RT2 (valid)
```

---

## 核心安全问题：Refresh Token Reuse（重放攻击）

### 什么是 Reuse？

当一个 **已经被轮换 / 吊销的 refresh token**（`revokedAt !== null`）  
**再次被使用** 时，就发生了 refresh token reuse。

这意味着：

> 该 token 已经被复制并落入了非预期的一方（攻击者）。

---

## 改动前的行为（❌ 不安全）

```ts
if (record.revokedAt !== null) {
  return null;
}
```

问题：

- 系统 **发现异常**
    
- 但只做了「静默失败」
    
- 同一 `familyId` 下的其他 refresh token **仍然有效**
    

结果：

- 攻击者被拒一次
    
- 但如果已获得新 token，仍可继续刷新
    
- 系统错过了唯一一次明确的攻击信号
    

---

## 改动后的行为（✅ 正确防护）

```ts
if (record.revokedAt !== null) {
  // Refresh token reuse detected
  revoke entire token family
}
```

具体策略：

- 一旦检测到 refresh token reuse
    
- 认为 **整个 token family 已泄露**
    
- 立即：
    
    - revoke 该 `familyId` 下所有未吊销的 refresh token
        
    - 强制重新登录
        

---

## 这个改动解决了什么问题？

### ✅ 1. 防止 Refresh Token Replay Attack（重放攻击）

- reuse 不再是“普通失败”
    
- 而是被识别为 **安全事件**
    

---

### ✅ 2. 防止 token 泄露后继续被使用

- 以前：系统“知道异常但不处理”
    
- 现在：系统主动切断整个信任链
    

---

### ✅ 3. 让 Token Rotation 真正具备安全意义

> 没有 reuse detection 的 rotation  
> 只是延迟攻击，而不是阻止攻击

---

### ✅ 4. 以 Token Family 为最小安全单元

- 单 token 视角 ❌
    
- family 级别视角 ✅
    

---

## Access Token vs Refresh Token 对比

|类型|Access Token|Refresh Token|
|---|---|---|
|生命周期|短（分钟）|长（天/周）|
|是否轮换|否|是|
|被盗后策略|等过期|**立即止损**|
|安全级别|时间限制|身份控制|

---

## 关键记忆点（一句话）

> **Refresh token 被轮换后再次出现 = token 泄露信号，必须 revoke 整个 token family**

---

## 标准表述（PR / 面试可用）

> This change adds refresh token reuse detection.
> 
> When a rotated refresh token is presented again, the system treats it as a security breach and revokes the entire token family to prevent replay attacks.

---

## 参考标准

- OAuth 2.1 — Refresh Token Rotation & Reuse Detection
    
- RFC 6819 — OAuth 2.0 Threat Model
    
- Auth0 / Okta Security Best Practices