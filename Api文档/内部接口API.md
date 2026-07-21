# 内部接口 API

> 服务：wo-bot-device-api（NestJS）
> 基础 URL：`https://api.wo-bot.com/api`
> 认证方式：JWT（OAuth2 Access Token）
> 调用方：wo-bot-web 前端（用户管理界面）

## 概览

wo-bot-account 子项目内部的 API，供 wo-bot-web 前端调用。用户通过授权服务登录后获取 Access Token，前端携带此 Token 调用以下接口管理设备绑定和授权应用。

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/devices` | 查询当前用户绑定的设备列表 |
| GET | `/devices/:robotId/status` | 查询单个设备状态 |
| POST | `/devices/bind` | 绑定设备到当前用户 |
| DELETE | `/devices/:robotId` | 解绑设备 |
| PATCH | `/devices/:robotId` | 重命名设备 |
| GET | `/apps` | 查询当前用户授权的第三方应用 |
| DELETE | `/apps/:grantId` | 撤销第三方应用授权 |
| GET | `/health` | 健康检查（无需认证） |

## 认证机制

### JWT 认证

前端在请求头携带用户登录后获取的 Access Token：

```
Authorization: Bearer <access_token>
```

Device API 按以下顺序验证 token：

1. **JWKS 验证**（首选）：通过 OIDC JWKS 端点验证 RS256/ES256/ES384/ES512 签名
2. **对称密钥验证**（降级）：使用 `JWT_SECRET` 验证 HS256/HS384/HS512 签名
3. **Opaque Token 验证**（兜底）：调用授权服务 `/oidc/me` 端点验证

验证通过后从 `sub` claim 提取 `userId` 注入请求上下文。

## 查询用户设备列表

返回当前用户绑定的所有设备，包含派生的在线状态（基于心跳时间）。

```
GET /api/devices
```

**认证**：JWT

**响应字段**（`data` 数组）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `robotId` | string | 设备唯一标识 |
| `robotName` | string \| null | 设备显示名称（用户自定义名称优先于注册名称） |
| `clientId` | string | 绑定时客户端标识 |
| `status` | `"online"` \| `"offline"` | 在线状态（3 分钟内有心跳为 online） |
| `lastSeenAt` | string (ISO 8601) | 最后心跳时间 |
| `boundAt` | string (ISO 8601) | 绑定时间 |

**成功响应**（200）：

```json
{
  "success": true,
  "data": [
    {
      "robotId": "robot-a1b2c3d4e5f6",
      "robotName": "客厅机器人",
      "clientId": "wo-bot-web-debug",
      "status": "online",
      "lastSeenAt": "2026-07-19T10:00:00.000Z",
      "boundAt": "2026-07-19T09:30:00.000Z"
    }
  ]
}
```

## 查询设备状态

查询单个设备的注册信息和绑定计数。

```
GET /api/devices/:robotId/status
```

**认证**：JWT

**路径参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `robotId` | string | 设备唯一标识 |

**响应字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `robotId` | string | 设备唯一标识 |
| `robotName` | string \| null | 注册时的设备名称 |
| `status` | `"online"` \| `"offline"` | 在线状态 |
| `lastSeenAt` | string (ISO 8601) | 最后心跳时间 |
| `registeredAt` | string (ISO 8601) | 注册时间 |
| `boundCount` | number | 已绑定到该设备的用户数 |

**成功响应**（200）：

```json
{
  "success": true,
  "data": {
    "robotId": "robot-a1b2c3d4e5f6",
    "robotName": "客厅机器人",
    "status": "online",
    "lastSeenAt": "2026-07-19T10:00:00.000Z",
    "registeredAt": "2026-07-19T09:00:00.000Z",
    "boundCount": 2
  }
}
```

## 绑定设备

将设备绑定到当前用户。需通过 9 步绑定证明验证。

```
POST /api/devices/bind
```

**认证**：JWT

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `payload` | object | 是 | 绑定证明载荷 |
| `payload.robotId` | string | 是 | 设备唯一标识 |
| `payload.clientId` | string | 是 | 发起绑定的客户端标识 |
| `payload.clientTokenHash` | string | 否 | 客户端本地凭证哈希（需与注册时一致） |
| `payload.accountId` | string | 是 | 当前用户 ID（必须与 JWT `sub` 一致） |
| `payload.nonce` | string | 是 | 随机串（防重放，5 分钟 TTL） |
| `payload.expiresAt` | number | 是 | 过期时间戳（毫秒） |
| `proof` | string | 是 | `HMAC-SHA256(canonical_json(payload), ROBOT_SECRET)` 的 hex 编码 |

**绑定证明 9 步验证**：

1. JWT 有效（由 JwtAuthGuard 保证），提取 `userId`
2. `payload.accountId` 必须等于 JWT `sub`
3. HMAC-SHA256 签名验证（`proof` 字段，使用稳定 JSON 序列化匹配 Python `json.dumps(sort_keys=True)`）
4. `payload.expiresAt` 未过期
5. `nonce` 未被使用过（Redis `SET NX`，5 分钟 TTL）
6. `robotId` 已在 `DeviceRegistration` 表注册
7. `clientTokenHash` 与注册时一致（如双方都提供）
8. 用户当日绑定次数 < `RATE_LIMIT_BIND_DAILY`（默认 10 次/天）
9. 用户已绑定设备总数 < `MAX_DEVICES_PER_USER`（默认 20 个）

**请求示例**：

```bash
curl -X POST https://api.wo-bot.com/api/devices/bind \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <access_token>" \
  -d '{
    "payload": {
      "robotId": "robot-a1b2c3d4e5f6",
      "clientId": "wo-bot-web-debug",
      "clientTokenHash": "a3f5...e1b2",
      "accountId": "user-xyz123",
      "nonce": "n7k8j9h0g1",
      "expiresAt": 1721381100000
    },
    "proof": "9b2c4d1e6f3a5b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"
  }'
```

**成功响应**（200）：

```json
{
  "success": true,
  "data": {
    "userId": "user-xyz123",
    "robotId": "robot-a1b2c3d4e5f6",
    "robotName": "客厅机器人",
    "clientId": "wo-bot-web-debug",
    "boundAt": "2026-07-19T10:00:00.000Z"
  }
}
```

**常见错误**：

| 状态码 | 错误信息 | 说明 |
|--------|---------|------|
| 400 | Payload accountId does not match authenticated user | accountId 与 JWT 不符 |
| 400 | Invalid binding proof signature | HMAC 签名错误 |
| 400 | Binding payload has expired | expiresAt 已过期 |
| 400 | Nonce has already been used | nonce 重放 |
| 400 | Device is not registered | 设备未注册 |
| 400 | Client token hash mismatch | clientTokenHash 不一致 |
| 400 | Daily binding limit (10) exceeded | 当日绑定次数超限 |
| 400 | Maximum devices per user (20) reached | 用户设备数已满 |
| 400 | Device is already bound to this user | 已绑定过 |

## 解绑设备

解除当前用户与设备的绑定关系。仅删除 `DeviceBinding` 记录，不影响设备注册信息。

```
DELETE /api/devices/:robotId
```

**认证**：JWT

**路径参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `robotId` | string | 设备唯一标识 |

**成功响应**（200）：

```json
{
  "success": true,
  "data": null
}
```

**错误响应**（404）：绑定关系不存在

## 重命名设备

修改用户视角下的设备显示名称。仅影响 `DeviceBinding.robotName`，不修改 `DeviceRegistration.robotName`。

```
PATCH /api/devices/:robotId
```

**认证**：JWT

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `robotName` | string | 否 | 新设备名称 |

**成功响应**（200）：

```json
{
  "success": true,
  "data": {
    "userId": "user-xyz123",
    "robotId": "robot-a1b2c3d4e5f6",
    "robotName": "卧室机器人",
    "clientId": "wo-bot-web-debug",
    "boundAt": "2026-07-19T10:00:00.000Z"
  }
}
```

## 查询授权应用列表

查询当前用户已授权的第三方应用列表。后端通过 M2M token 调用 Logto Management API（`/api/users/{userId}/grants`）并关联应用详情。

```
GET /api/apps
```

**认证**：JWT

**响应字段**（`data` 数组）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `grantId` | string | 授权记录 ID |
| `appId` | string | 应用 ID |
| `appName` | string | 应用名称 |
| `appDescription` | string | 应用描述 |
| `appType` | string | 应用类型代码（SPA / Native / Traditional / MachineToMachine / SAML / Protected） |
| `appTypeLabel` | string | 应用类型中文标签 |
| `scopes` | string[] | 授权的 scope 列表 |
| `createdAt` | string (ISO 8601) | 授权时间 |

**成功响应**（200）：

```json
{
  "success": true,
  "data": [
    {
      "grantId": "grant-abc123",
      "appId": "app-xyz789",
      "appName": "wo-bot-web-debug",
      "appDescription": "本地调试客户端",
      "appType": "SPA",
      "appTypeLabel": "Web 应用",
      "scopes": ["openid", "profile"],
      "createdAt": "2026-07-19T10:00:00.000Z"
    }
  ]
}
```

## 撤销应用授权

撤销当前用户对指定第三方应用的授权。后端通过 M2M token 调用 Logto Management API（`DELETE /api/users/{userId}/grants/{grantId}`）。

```
DELETE /api/apps/:grantId
```

**认证**：JWT

**路径参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `grantId` | string | 授权记录 ID |

**成功响应**（200）：

```json
{
  "success": true,
  "data": null
}
```

## 健康检查

用于 Docker 健康检查和负载均衡探针。无需认证。

```
GET /api/health
```

**成功响应**（200）：

```json
{
  "status": "ok",
  "timestamp": "2026-07-19T10:00:00.000Z"
}
```

## 相关文档

- [设备接入 API](./设备接入API.md) — 设备注册和心跳（HMAC 认证）
- [应用接入 API](./应用接入API.md) — 第三方应用接入授权（OAuth2/OIDC）
- [R00040-1 用户设备与授权管理后台](../../docs/需求/进行中/R00040-1%20用户设备与授权管理后台.md) — 需求定义
- [部署文档](../部署/用户帐号服务器/README.md) — 部署配置
