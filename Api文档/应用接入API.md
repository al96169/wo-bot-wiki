# 应用接入 API

> 服务：授权服务（Logto Core）
> 基础 URL：`https://user.wo-bot.com`
> OIDC 路径前缀：`/oidc`
> 调用方：第三方应用（如 wo-bot-web-debug、wo-bot-web 等）

## 概览

第三方应用通过 OAuth2 Authorization Code + PKCE 流程接入 wo-bot 授权服务，获取用户身份令牌（Access Token / ID Token / Refresh Token）。授权服务基于 Logto，完全兼容 OIDC 1.0 规范。

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/oidc/.well-known/openid-configuration` | OIDC Discovery（动态获取端点配置） |
| GET | `/oidc/auth` | 授权端点（启动登录流程） |
| POST | `/oidc/token` | Token 端点（换取/刷新令牌） |
| GET | `/oidc/me` | UserInfo（查询用户信息） |
| GET | `/oidc/jwks` | JWKS 公钥（验证 JWT 签名） |
| GET | `/oidc/session/end` | 登出端点（结束会话） |

## OIDC Discovery

返回授权服务的完整配置信息。**客户端应通过此端点动态获取配置，避免硬编码端点 URL**。

```
GET /oidc/.well-known/openid-configuration
```

**响应示例**（关键字段）：

```json
{
  "issuer": "https://user.wo-bot.com/oidc",
  "authorization_endpoint": "https://user.wo-bot.com/oidc/auth",
  "token_endpoint": "https://user.wo-bot.com/oidc/token",
  "userinfo_endpoint": "https://user.wo-bot.com/oidc/me",
  "jwks_uri": "https://user.wo-bot.com/oidc/jwks",
  "end_session_endpoint": "https://user.wo-bot.com/oidc/session/end",
  "grant_types_supported": ["authorization_code", "refresh_token", "client_credentials"],
  "response_types_supported": ["code"],
  "scopes_supported": ["openid", "profile", "email", "offline_access"]
}
```

## 授权端点

启动 OAuth2 授权码流程。浏览器跳转到此端点，用户在授权服务登录页完成认证后，重定向回应用的 `redirect_uri` 并附带授权码。

```
GET /oidc/auth
```

**查询参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `client_id` | string | 是 | 应用 ID（在管理后台创建） |
| `redirect_uri` | string | 是 | 回调地址（必须与应用配置一致） |
| `response_type` | string | 是 | 固定值 `code` |
| `scope` | string | 是 | 请求的 scope，空格分隔（如 `openid profile email offline_access`） |
| `code_challenge` | string | 是 | PKCE challenge（`BASE64URL(SHA256(code_verifier))`） |
| `code_challenge_method` | string | 是 | 固定值 `S256` |
| `state` | string | 否 | 客户端随机串（防 CSRF，原样返回） |
| `resource` | string | 否 | 资源指示器（指定 Access Token 的受众） |
| `prompt` | string | 否 | 提示类型（`login` 强制重新登录、`consent` 强制同意） |

**请求示例**（浏览器跳转）：

```
https://user.wo-bot.com/oidc/auth?
  client_id=app-xyz789
  &redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback
  &response_type=code
  &scope=openid+profile+email+offline_access
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
  &state=xyz123
```

**成功响应**（302 重定向）：

```
Location: https://app.example.com/callback?code=auth_code_xxx&state=xyz123
```

## Token 端点

用授权码换取令牌，或用 Refresh Token 刷新令牌。

```
POST /oidc/token
```

**Content-Type**：`application/x-www-form-urlencoded`

### 授权码换取 Token

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `grant_type` | string | 是 | `authorization_code` |
| `code` | string | 是 | 从 `/oidc/auth` 获取的授权码 |
| `redirect_uri` | string | 是 | 必须与授权请求一致 |
| `client_id` | string | 是 | 应用 ID |
| `code_verifier` | string | 是 | PKCE 原始串（用于验证 `code_challenge`） |

### 刷新 Token

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `grant_type` | string | 是 | `refresh_token` |
| `refresh_token` | string | 是 | 上一次获取的 Refresh Token |
| `client_id` | string | 是 | 应用 ID |
| `scope` | string | 否 | 请求的 scope（不能超过原授权范围） |

**成功响应**（200）：

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "refresh_token": "r1t2e3f4g5h6j7k8",
  "id_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "openid profile email offline_access"
}
```

**Token 有效期**：
- Access Token：约 1 小时
- Refresh Token：14 天（启用轮换，每次刷新返回新的 Refresh Token）

## UserInfo 端点

用 Access Token 查询当前用户信息。

```
GET /oidc/me
```

**认证**：`Authorization: Bearer <access_token>`

**成功响应**（200）：

```json
{
  "sub": "user-xyz123",
  "email": "user@example.com",
  "email_verified": true,
  "name": "张三",
  "picture": "https://...",
  "username": "zhangsan"
}
```

## JWKS 端点

返回授权服务公钥集合，用于客户端验证 JWT 签名。

```
GET /oidc/jwks
```

**成功响应**（200）：

```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "key-1",
      "use": "sig",
      "alg": "RS256",
      "n": "v123...",
      "e": "AQAB"
    }
  ]
}
```

## 登出端点

结束用户会话并重定向回应用。

```
GET /oidc/session/end
```

**查询参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id_token_hint` | string | 否 | ID Token（标识用户） |
| `post_logout_redirect_uri` | string | 否 | 登出后重定向地址（需预先注册） |
| `state` | string | 否 | 随机串（原样返回） |

## 应用类型

应用接入授权服务前，需在管理后台创建应用并选择类型：

| 类型 | 适用场景 | 认证方式 | 示例 |
|------|---------|---------|------|
| SPA | 单页 Web 应用 | PKCE | wo-bot-web（用户管理前端） |
| Traditional | 传统 Web 应用 | Client Secret | 服务端渲染应用 |
| Native | 移动/桌面应用 | PKCE | wo-bot-web-debug（本地调试） |
| MachineToMachine | 后端服务 | Client Credentials | 服务间调用 |
| SAML | 企业 SSO | SAML 协议 | 企业身份提供商 |
| Protected | 受保护资源 | — | API 资源标识 |

## 完整接入流程（PKCE）

```
1. 生成 code_verifier（43-128 字符随机串）
2. 计算 code_challenge = BASE64URL(SHA256(code_verifier))
3. 浏览器跳转到授权端点：
   GET https://user.wo-bot.com/oidc/auth?
     client_id=app-xyz789
     &redirect_uri=https://app.example.com/callback
     &response_type=code
     &scope=openid+profile+email+offline_access
     &code_challenge=<code_challenge>
     &code_challenge_method=S256
     &state=<random_state>
4. 用户在授权服务登录页完成认证
5. 浏览器重定向回 redirect_uri?code=<auth_code>&state=<random_state>
6. 应用服务端用 code 换取 token：
   POST https://user.wo-bot.com/oidc/token
     grant_type=authorization_code
     &code=<auth_code>
     &redirect_uri=https://app.example.com/callback
     &client_id=app-xyz789
     &code_verifier=<code_verifier>
7. 获取到 access_token、refresh_token、id_token
8. 后续 API 请求携带 Authorization: Bearer <access_token>
9. Token 过期时用 refresh_token 刷新：
   POST https://user.wo-bot.com/oidc/token
     grant_type=refresh_token
     &refresh_token=<refresh_token>
     &client_id=app-xyz789
```

## 相关文档

- [设备接入 API](./设备接入API.md) — 设备注册和心跳
- [内部接口 API](./内部接口API.md) — wo-bot-account 前后端接口
- [R00040-2 用户授权服务选型](../../docs/需求/进行中/R00040-2%20用户授权服务选型.md) — 选型分析
- [Logto 官方文档](https://docs.logto.io/) — 协议规范
