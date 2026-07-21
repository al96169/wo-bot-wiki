# API 文档

本目录收录 wo-bot 项目的对外 API 规范，按调用方分为三类。

## 文档目录

| 文档 | 调用方 | 认证方式 | 说明 |
|------|--------|---------|------|
| [设备接入 API](./设备接入API.md) | 机器人设备 | HMAC-SHA256 | 设备注册、心跳上报 |
| [应用接入 API](./应用接入API.md) | 第三方应用 | OAuth2 PKCE | 授权登录、Token 签发、UserInfo |
| [内部接口 API](./内部接口API.md) | wo-bot-web 前端 | JWT | 设备绑定、解绑、授权应用管理 |

## 分类说明

### 设备接入 API（设备 → 授权服务）

机器人设备（wo-bot-control）通过 HMAC 签名认证调用，完成设备注册和心跳上报。设备无需用户登录，仅凭 `ROBOT_SECRET` 共享密钥进行身份验证。

### 应用接入 API（应用 → 授权服务）

第三方应用（如 wo-bot-web-debug、wo-bot-web）通过 OAuth2 Authorization Code + PKCE 流程接入，获取用户身份令牌。基于 Logto 授权服务，兼容 OIDC 1.0 规范。

### 内部接口 API（wo-bot-web 前端 → wo-bot-device-api 后端）

wo-bot-account 子项目内部的 API，供用户管理前端调用。用户通过授权服务登录后获取 Access Token，前端携带此 Token 调用以下接口管理设备绑定和授权应用。

> Logto 作为外部组件，其内部 Management API 不在本目录文档范围内。

## 服务域名

| 域名 | 服务 | 说明 |
|------|------|------|
| `api.wo-bot.com` | wo-bot-device-api | 设备接入 API + 内部接口 API |
| `user.wo-bot.com` | 授权服务（Logto Core） | 应用接入 API（OIDC 端点）+ 用户管理页面 |
| `admin.wo-bot.com` | Logto Admin Console | 管理员控制台（Cloudflare Access + WAF 防护） |

## 通用响应格式

### 成功响应

```json
{
  "success": true,
  "data": { ... }
}
```

### 错误响应

```json
{
  "success": false,
  "error": "错误描述信息",
  "statusCode": 400
}
```

### 常见 HTTP 状态码

| 状态码 | 含义 | 说明 |
|--------|------|------|
| 200 | OK | 请求成功 |
| 400 | Bad Request | 参数校验失败、业务校验失败（如 HMAC 签名错误、绑定证明无效） |
| 401 | Unauthorized | 认证失败（缺少 Authorization 头、JWT 过期、HMAC 头缺失） |
| 404 | Not Found | 资源不存在（如设备未注册、绑定关系不存在） |
| 500 | Internal Server Error | 服务器内部错误 |
