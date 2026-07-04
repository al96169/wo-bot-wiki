# WebSocket 协议（用于局域网通讯）

> 版本: v1.0 | 协议版本号: 1
> 定义 wo-bot-web-debug（控制端）与 wo-bot-control（机器人端）之间的 WebSocket 通信协议。

---

## 1. 架构概述

```
控制端 (浏览器/Vue3)                    机器人端 (Python/asyncio)
┌──────────────────────┐              ┌──────────────────────────┐
│ wo-bot-web-debug     │   ws://      │ wo-bot-control            │
│ useWebSocket.ts      │◄────────────►│ websocket_server.py       │
│                      │   :8765      │  (port 8765)              │
│ 发送优先级:           │              │                          │
│  DataChannel > WS    │              │  信令 + 业务消息兜底       │
└──────────────────────┘              └──────────────────────────┘
```

WebSocket 承担三重角色：
1. **设备发现握手** — 连接后返回机器人基础信息（ID/名称/型号/版本/功能列表）
2. **WebRTC 信令** — 中继 SDP offer/answer 和 ICE candidate 交换
3. **业务消息兜底** — 当 WebRTC DataChannel 不可用时，所有业务消息回退到 WebSocket 传输

---

## 2. 连接建立

### 2.1 端点

```
ws://{robot_ip}:8765?protocol_version=1
```

开发模式（通过 Vite 代理）：
```
ws://localhost:9093/api/device-ws?host={robot_ip}&port=8765&protocol_version=1
```

### 2.2 URL 参数

| 参数 | 必填 | 说明 |
|------|------|------|
| `protocol_version` | 是 | 客户端协议版本号（当前为 1） |
| `token` | 否 | 安全认证令牌（auth_enabled=true 时必填） |
| `client_ip` | 否 | Vite 代理注入的真实客户端 IP（修复 mDNS .local 解析问题） |
| `debug_pv` | 否 | 调试用，强制覆盖协议版本号 |

### 2.3 连接流程

```
1. 客户端发起 WebSocket 连接（带 protocol_version 参数）
2. 服务端版本兼容性检查:
   - 无 protocol_version → 关闭连接 (code 4001)
   - client_protocol < min_protocol_version → 关闭连接 (code 4001)
   - reject_newer=true 且 client_protocol > server_protocol → 关闭连接 (code 4001)
3. (可选) 安全认证: URL token 或首条 auth 消息
4. 认证通过 → 发送 "connected" 消息（含机器人信息 + features 列表）
5. 客户端收到 connected → 自动 subscribe + 请求摄像头列表
6. 双方开始心跳（15s 间隔 / 5s 超时）
```

### 2.4 关闭码

| 关闭码 | 含义 |
|--------|------|
| 4001 | 版本不匹配或缺少协议版本（不透露服务端版本） |
| 1000 | 正常关闭 |

---

## 3. 消息格式

所有消息采用 JSON 格式：

```json
{
  "type": "消息类型",
  "timestamp": 1699999999000,
  "data": { ... }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| type | string | 是 | 消息类型标识 |
| timestamp | number | 否 | 毫秒时间戳 |
| data | object | 否 | 消息载荷 |

WebSocket 与 WebRTC DataChannel 使用**完全相同的消息格式**，业务代码无需区分传输通道。

### 3.1 发送优先级（前端）

```
DataChannel (open) > WebSocket (open) > 内存队列 (最多 50 条)
```

心跳（ping/pong）始终走 WebSocket 直发，不走 DataChannel。

---

## 4. 消息类型定义

### 4.1 连接管理

#### connected — 连接成功（服务端 → 客户端）

```json
{
  "type": "connected",
  "data": {
    "robot_id": "wobot-001",
    "name": "My Robot",
    "model": "jetson-nano",
    "version": "1.0.0",
    "features": ["websocket", "webrtc", "exec", "motion", "system", "camera", "gimbal"]
  }
}
```

features 含义：
- `websocket` — WebSocket 基础通信
- `webrtc` — 支持 WebRTC P2P（可发起 offer 建立 DataChannel + 视频流）
- `websocket-fallback` — 不支持 WebRTC，WebSocket 承载所有业务消息
- `exec` — 支持远程命令执行（PTY）、进程管理、软件管理
- `motion` — 支持运动控制
- `system` — 支持系统操作（重启/关机）
- `camera` — 支持摄像头管理
- `gimbal` — 支持云台控制
- `dance` — 支持舞蹈编排
- `music` — 支持音乐播放

#### ping / pong — 心跳

```
客户端 → 服务端: { "type": "ping", "data": { "ts": 1699999999000 } }
服务端 → 客户端: { "type": "pong", "data": { "ts": 1699999999000 } }
```

心跳参数（前端）：
- 间隔：15 秒
- 超时：5 秒无 pong 则判定断开并触发重连
- 心跳始终通过 WebSocket 直发

#### subscribe / unsubscribe — 事件订阅

```json
// 订阅
{ "type": "subscribe", "data": { "events": ["status"] } }

// 取消订阅
{ "type": "unsubscribe" }
```

订阅后，服务端每秒通过 WebSocket 广播 `status` 消息给所有已订阅客户端。

#### auth — 安全认证（客户端 → 服务端）

```json
{ "type": "auth", "data": { "token": "your-secret-token" } }
```

认证方式（二选一）：
1. URL 参数 `?token=xxx`
2. 连接后首条消息发 `auth`

认证超时：10 秒，超时或失败均返回 error 并关闭连接。

---

### 4.2 WebRTC 信令

当 `connected.features` 包含 `"webrtc"` 时，客户端可发起 WebRTC 协商。

#### webrtc_offer — 客户端 SDP Offer

```json
// 客户端 → 服务端
{
  "type": "webrtc_offer",
  "data": { "sdp": "v=0\r\no=- ..." }
}
```

#### webrtc_answer — 服务端 SDP Answer

```json
// 服务端 → 客户端
{
  "type": "webrtc_answer",
  "data": { "sdp": "v=0\r\no=- ..." }
}
```

Answer SDP 特点：
- 注入 `a=ice-lite`（服务端不执行主动 ICE 连通性检查）
- `.local` mDNS 地址替换为真实 IP

#### webrtc_ice_candidate — ICE 候选交换

```json
// 双向
{
  "type": "webrtc_ice_candidate",
  "data": {
    "candidate": "candidate:...",
    "sdpMid": "0",
    "sdpMLineIndex": 0
  }
}
```

服务端对 `.local` 候选的特殊处理：
- 保留原始 `.local` 候选
- 同时追加一个高优先级真实 IP 候选（priority=2122252543）
- 真实 IP 候选先添加，优先尝试

#### webrtc_close — 关闭 WebRTC 连接

```json
// 客户端 → 服务端
{ "type": "webrtc_close", "data": {} }
```

---

### 4.3 系统状态

#### get_status — 请求系统状态

```json
// 客户端 → 服务端
{ "type": "get_status", "data": {} }

// 服务端 → 客户端
{
  "type": "status",
  "data": {
    "battery": { "level": 85, "status": "discharging", "temperature": 25.5 },
    "system": {
      "cpu_percent": 35.2, "memory_percent": 45.8, "disk_percent": 62.1,
      "uptime": 3600, "temperature": 42.0, "hostname": "jetson"
    },
    "network": { "ip": "192.168.1.47", "ssid": "MyWiFi", "signal_strength": -45 },
    "features": ["webrtc", ...],
    "services": [
      { "service_id": "main", "name": "主服务", "status": "running", "pid": 1234 }
    ]
  }
}
```

#### status — 状态推送（服务端主动广播）

subscribe 后每秒推送，格式同上。WebRTC DataChannel 也会收到相同消息。

---

### 4.4 运动控制

#### motion — 运动指令

支持两种协议格式，自动识别：

**三轴麦轮协议**（优先）：
```json
{
  "type": "motion",
  "data": { "v_x": 0.5, "v_y": 0.0, "v_z": 0.3 }
}
```

**双轴兼容协议**：
```json
{
  "type": "motion",
  "data": { "linear": 0.5, "angular": 0.3, "mode": "manual" }
}
```

#### motion_stop — 停止运动

```json
{ "type": "motion_stop", "data": {} }
```

#### emergency_stop — 急停

```json
// 客户端 → 服务端
{ "type": "emergency_stop", "data": {} }

// 服务端 → 客户端 (确认)
{ "type": "emergency_stop_ack", "data": {} }
```

#### emergency_release — 解除急停

```json
{ "type": "emergency_release", "data": {} }
```

#### motion_config — 运动配置

```json
{
  "type": "motion_config",
  "data": {
    "drive_type": "mecanum",
    "max_linear_speed": 1.0,
    "max_angular_speed": 1.0
  }
}
```

---

### 4.5 云台控制

#### gimbal — 云台控制

```json
// 居中
{ "type": "gimbal", "data": { "action": "center" } }

// 增量移动
{ "type": "gimbal", "data": { "action": "move", "pan_delta": 5.0, "tilt_delta": 3.0, "step": 3.0 } }

// 持续移动（按下）
{ "type": "gimbal", "data": { "action": "move_begin", "pan_speed": 1.0, "tilt_speed": 0.0 } }

// 更新速度（拖动中）
{ "type": "gimbal", "data": { "action": "move_update", "pan_speed": 0.5, "tilt_speed": -0.5 } }

// 停止移动（松开）
{ "type": "gimbal", "data": { "action": "move_end" } }

// 设置绝对角度
{ "type": "gimbal", "data": { "axis": "pan", "angle": 90 } }
```

#### gimbal_status — 云台状态推送

```json
{ "type": "gimbal_status", "data": { "pan": 90, "tilt": 45 } }
```

#### gimbal_limit — 云台限位通知

```json
{ "type": "gimbal_limit", "data": { "pan": 0, "tilt": 90, "limit_axis": "pan", "limit": "min" } }
```

---

### 4.6 摄像头管理

#### camera — 摄像头控制

```json
// 开启
{ "type": "camera", "data": { "action": "start", "camera_id": 0 } }

// 停止
{ "type": "camera", "data": { "action": "stop", "camera_id": 0 } }

// 切换
{ "type": "camera", "data": { "action": "switch", "camera_id": 1 } }

// 列出所有摄像头
{ "type": "camera", "data": { "action": "list" } }
```

#### camera_status — 摄像头状态

```json
{
  "type": "camera_status",
  "data": {
    "cameras": [
      {
        "id": 0,
        "name": "CSI Camera",
        "status": "streaming",
        "resolution": "640x480",
        "fps": 15,
        "stream_url": "webrtc://..."
      }
    ]
  }
}
```

---

### 4.7 系统操作

#### system — 系统操作

```json
{ "type": "system", "data": { "action": "reboot" } }
{ "type": "system", "data": { "action": "shutdown" } }
{ "type": "system", "data": { "action": "restart_service" } }

// 确认
{ "type": "system_ack", "data": { "action": "reboot", "status": "ok" } }
```

#### exec — 远程命令执行（PTY 终端）

```json
// 客户端 → 服务端
{ "type": "exec", "data": { "command": "ls -la", "timeout": 5000 } }

// 服务端 → 客户端
{
  "type": "exec_result",
  "data": { "stdout": "...", "stderr": "", "return_code": 0, "cwd": "/home/user" }
}
```

---

### 4.8 软件管理

#### software_list — 已安装软件

```json
// 请求
{ "type": "software_list", "data": {} }

// 响应
{
  "type": "software_list",
  "data": {
    "packages": [
      { "name": "python3", "version": "3.10.6", "status": "installed", "source": "apt" }
    ]
  }
}
```

#### software_search — 搜索可用软件

```json
// 请求
{ "type": "software_search", "data": { "keyword": "vim" } }

// 响应
{
  "type": "software_search_result",
  "data": {
    "packages": [
      { "name": "vim", "description": "Vi IMproved", "source": "apt" }
    ]
  }
}
```

#### software_install / uninstall / upgrade

```json
// 安装 / 卸载 / 升级
{ "type": "software_install", "data": { "package": "vim" } }
{ "type": "software_uninstall", "data": { "package": "vim" } }
{ "type": "software_upgrade", "data": { "package": "vim" } }

// 确认
{ "type": "software_install_ack", "data": { "package": "vim", "status": "installed" } }
{ "type": "software_uninstall_ack", "data": { "package": "vim", "status": "uninstalled" } }
{ "type": "software_upgrade_ack", "data": { "package": "vim", "status": "upgraded" } }
```

---

### 4.9 WiFi 管理

#### wifi_scan — 扫描 WiFi

```json
// 请求
{ "type": "wifi_scan", "data": {} }

// 响应
{
  "type": "wifi_scan_result",
  "data": {
    "current_ssid": "MyWiFi",
    "current_device": "wlan0",
    "networks": [
      { "ssid": "MyWiFi", "signal": -45, "security": "WPA2", "connected": true },
      { "ssid": "Neighbor", "signal": -70, "security": "WPA2", "connected": false }
    ]
  }
}
```

#### wifi_connect / disconnect

```json
{ "type": "wifi_connect", "data": { "ssid": "MyWiFi", "password": "12345678" } }
{ "type": "wifi_disconnect", "data": { "device": "wlan0" } }

// 连接结果
{ "type": "wifi_connect_result", "data": { "ssid": "MyWiFi", "status": "connected" } }
{ "type": "wifi_connect_result", "data": { "ssid": "MyWiFi", "status": "failed", "error": "..." } }
{ "type": "wifi_disconnect_result", "data": {} }
```

---

### 4.10 服务管理

#### service_status — 获取所有服务状态

```json
// 请求
{ "type": "service_status", "data": {} }

// 响应
{
  "type": "service_status",
  "data": {
    "services": [
      { "service_id": "main", "name": "主服务", "status": "running", "pid": 1234 },
      { "service_id": "webrtc", "name": "WebRTC服务", "status": "running", "pid": 5678 }
    ]
  }
}
```

#### service_control — 控制服务

```json
{ "type": "service_control", "data": { "service_id": "webrtc", "action": "restart" } }
// action 可选: start, stop, restart

// 确认
{
  "type": "service_control_ack",
  "data": {
    "service_id": "webrtc",
    "action": "restart",
    "status": "running",
    "services": [ ... ]
  }
}
```

#### service_message — 服务通知

```json
{
  "type": "service_message",
  "data": {
    "subject": "服务异常",
    "summary": "WebRTC 服务已重启",
    "body": "连续异常重启超过阈值...",
    "source": "service_manager",
    "severity": "warning"
  }
}
```

---

### 4.11 音乐播放

```json
// 播放控制
{ "type": "music_play", "data": { "song_id": "xxx" } }
{ "type": "music_pause", "data": {} }
{ "type": "music_stop", "data": {} }
{ "type": "music_resume", "data": {} }
{ "type": "music_next", "data": {} }
{ "type": "music_previous", "data": {} }
{ "type": "music_seek", "data": { "position": 30 } }

// 音量
{ "type": "music_volume", "data": { "volume": 75 } }

// 获取状态/列表
{ "type": "music_status", "data": {} }
{ "type": "music_list", "data": {} }

// 响应
{
  "type": "music_status",
  "data": {
    "status": "playing",
    "song_id": "xxx",
    "title": "Song Name",
    "artist": "Artist",
    "duration": 240,
    "position": 30,
    "volume": 75,
    "streaming": false
  }
}

{
  "type": "music_list",
  "data": {
    "songs": [
      { "id": "xxx", "title": "Song", "artist": "Artist", "duration": 240, "file_path": "/path" }
    ]
  }
}
```

---

### 4.12 舞蹈编排

```json
// 获取舞蹈列表
{ "type": "dance", "data": { "action": "list" } }

// 执行舞蹈
{ "type": "dance", "data": { "action": "play", "dance_id": "xxx" } }

// 停止
{ "type": "dance", "data": { "action": "stop" } }

// 响应
{
  "type": "dance_list",
  "data": { "dances": [{ "id": "xxx", "name": "舞蹈1", "duration": 60 }] }
}

{
  "type": "dance_status",
  "data": { "status": "playing", "dance_id": "xxx", "progress": 0.5 }
}
```

---

### 4.13 模块管理

```json
// 获取模块列表
{ "type": "module_list", "data": {} }

// 响应
{
  "type": "module_list",
  "data": {
    "modules": [
      { "id": "env-sensor", "name": "环境传感器", "version": "1.0.0", "status": "running", "enabled": true }
    ]
  }
}

// 模块控制
{ "type": "module_control", "data": { "module_id": "env-sensor", "action": "start" } }

// 确认
{ "type": "module_control_ack", "data": { "module_id": "env-sensor", "action": "start", "status": "ok" } }
```

---

### 4.14 错误通知

```json
{
  "type": "error",
  "data": {
    "code": 400,
    "message": "Invalid motion parameters"
  }
}
```

| 错误码 | 说明 |
|--------|------|
| 400 | 请求参数错误 / JSON 格式错误 |
| 401 | 未授权 / 认证失败 |
| 403 | 禁止访问 / 服务端缺少 token 配置 |
| 500 | 服务器内部错误 |
| 503 | 服务不可用（如 WebRTC 未启用） |

---

## 5. 自动重连机制（前端）

| 参数 | 值 |
|------|-----|
| 连接超时 | 5 秒 |
| 重连间隔 | 3 秒 × 重连次数（指数递增） |
| 首次连接最大重试 | 3 次（未收到 connected 前） |
| 保活最大重连 | 30 次（收到过 connected 后） |
| 心跳间隔 | 15 秒 |
| 心跳超时 | 5 秒 |
| 消息队列上限 | 50 条（超限丢弃最旧） |
| 不重连条件 | 主动断开 / 服务端 code >= 4000 / Mock 模式 |

---

## 6. 版本兼容性

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `min_protocol_version` | 1 | 客户端协议版本必须 ≥ 此值 |
| `reject_newer` | false | 是否拒绝比服务端版本更高的客户端 |
| `force_min_protocol_version` | 0 | 调试用，强制提升最低版本（0=禁用） |

配置位于 `wo-bot-control/config/config.yaml` → `compatibility` 节。

---

## 7. 安全机制

| 机制 | 说明 |
|------|------|
| Token 认证 | URL 参数 `?token=xxx` 或首条 `auth` 消息 |
| 版本检查 | 连接时即验证，拒绝时不透露服务端版本（统一返回 4001） |
| 超时保护 | 认证超时 10 秒，心跳超时 5 秒 |
| 消息大小限制 | 1MB（`max_size=2**20`） |

安全配置位于 `wo-bot-control/config/config.yaml` → `security` 节：
```yaml
security:
  auth_enabled: false
  token: ""
```

---

## 8. mDNS 设备发现

### 服务注册

机器人端使用 zeroconf 广播服务：

```
服务类型: _wobot._tcp.local.
端口: 8765
```

TXT 记录：
```
name=My Robot
model=jetson-nano
version=1.0.0
id=wobot-001
```

### 发现方式

前端使用双重策略：
1. **mDNS API** — 通过 Vite 插件的 `/api/discover` 接口（Node.js bonjour-service 扫描）
2. **同网段 IP 快速探活** — 扫描局域网 IP 段，对 8765 端口发起 WebSocket 连接尝试
