## 总体流程

1. **Login**：用户提供“身份证明”（密码/OTP/OIDC）
    
2. **签发**：后端签一个 **Access Token（JWT，短期）**
    
3. **携带**：客户端每次请求带上 token（Header 或 Cookie）
    
4. **验证**：后端验签 + 看过期 + 看权限 → 放行/拒绝
    
5. **续期**：Access 过期后，用 **Refresh Token（长期）** 换新 Access（通常还会轮换 Refresh）
    
6. **登出/撤销**：刷新凭证被吊销后，无法再续期

