# 设备接入 API

> 服务：wo-bot-device-api（NestJS）
> 基础 URL：`https://api.wo-bot.com/api`
> 认证方式：HMAC-SHA256
> 调用方：机器人设备（wo-bot-control）

## 概览

机器人设备通过 HMAC 签名认证调用以下接口，完成设备注册和心跳上报。设备无需用户登录，仅凭 `ROBOT_SECRET` 共享密钥进行身份验证。

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/devices/register` | 设备注册（首次接入或更新信息） |
| POST | `/devices/heartbeat` | 心跳上报（维持在线状态） |

## 认证机制

### HMAC-SHA256 签名

每次请求需在 HTTP 头中携带以下三个字段：

| 请求头 | 说明 | 示例 |
|--------|------|------|
| `X-Robot-Id` | 设备唯一标识 | `robot-a1b2c3d4e5f6` |
| `X-Timestamp` | 毫秒级时间戳（5 分钟有效窗口） | `1721380800000` |
| `X-Signature` | `HMAC-SHA256("{robotId}:{timestamp}", ROBOT_SECRET)` 的 hex 编码 | `8a7f9b2c...` |

**签名计算**：

```python
import hmac, hashlib, time

robot_id = "robot-a1b2c3d4e5f6"
timestamp = str(int(time.time() * 1000))
payload = f"{robot_id}:{timestamp}"
signature = hmac.new(
    ROBOT_SECRET.encode(),
    payload.encode(),
    hashlib.sha256
).hexdigest()

headers = {
    "Content-Type": "application/json",
    "X-Robot-Id": robot_id,
    "X-Timestamp": timestamp,
    "X-Signature": signature,
}
```

**防重放**：时间戳超出 5 分钟窗口的请求会被拒绝。

## 设备注册

注册新设备或更新已注册设备。使用 upsert 语义，相同 `robotId` 会更新 `robotName`、`secretHash`、`clientTokenHash` 和 `lastSeenAt`。

```
POST /api/devices/register
```

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `robotId` | string | 是 | 设备唯一标识（格式 `robot-{GUID}`，GUID 来自系统 machine-id 或 IOPlatformUUID） |
| `robotName` | string | 否 | 设备显示名称 |
| `clientTokenHash` | string | 否 | 客户端本地绑定凭证的 SHA-256 哈希（用于绑定证明验证） |

**请求示例**：

```bash
curl -X POST https://api.wo-bot.com/api/devices/register \
  -H "Content-Type: application/json" \
  -H "X-Robot-Id: robot-a1b2c3d4e5f6" \
  -H "X-Timestamp: 1721380800000" \
  -H "X-Signature: 8a7f9b2c4d1e6f3a5b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1" \
  -d '{
    "robotId": "robot-a1b2c3d4e5f6",
    "robotName": "客厅机器人",
    "clientTokenHash": "a3f5...e1b2"
  }'
```

**成功响应**（200）：

```json
{
  "success": true,
  "data": {
    "robotId": "robot-a1b2c3d4e5f6",
    "robotName": "客厅机器人",
    "secretHash": "...",
    "clientTokenHash": "a3f5...e1b2",
    "status": "online",
    "lastSeenAt": "2026-07-19T10:00:00.000Z",
    "registeredAt": "2026-07-19T10:00:00.000Z"
  }
}
```

## 设备心跳

设备定期上报心跳，更新 `lastSeenAt` 和在线状态。超过 3 分钟未上报心跳的设备会被定时任务标记为 `offline`。

```
POST /api/devices/heartbeat
```

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `robotId` | string | 是 | 设备唯一标识 |

**请求示例**：

```bash
curl -X POST https://api.wo-bot.com/api/devices/heartbeat \
  -H "Content-Type: application/json" \
  -H "X-Robot-Id: robot-a1b2c3d4e5f6" \
  -H "X-Timestamp: 1721380800000" \
  -H "X-Signature: 8a7f9b2c..." \
  -d '{"robotId": "robot-a1b2c3d4e5f6"}'
```

**成功响应**（200）：

```json
{
  "success": true,
  "data": null
}
```

**错误响应**（404）：设备未注册

```json
{
  "success": false,
  "error": "Device not registered",
  "statusCode": 404
}
```

## 设备标识生成规则

设备唯一标识 `robotId` 由后端机器人服务（wo-bot-control）在首次启动时生成：

- **Linux**：读取 `/etc/machine-id`
- **macOS**：执行 `ioreg -d2 -c IOPlatformExpertDevice | awk -F'"' '/IOPlatformUUID/{print $4}'`
- **格式**：`robot-{GUID}`，如 `robot-a1b2c3d4-e5f6-7890-abcd-ef1234567890`

## 相关文档

- [应用接入 API](./应用接入API.md) — 第三方应用如何接入授权
- [内部接口 API](./内部接口API.md) — wo-bot-account 前后端接口
- [R00040-1 用户设备与授权管理后台](../../docs/需求/进行中/R00040-1%20用户设备与授权管理后台.md) — 需求定义
