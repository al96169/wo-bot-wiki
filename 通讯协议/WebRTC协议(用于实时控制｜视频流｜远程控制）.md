# WebRTC 协议（用于实时控制 | 视频流 | 远程控制）

> 版本: v1.0
> 定义 wo-bot-web-debug（Web 控制端）与 wo-bot-control（机器人端）之间的 WebRTC P2P 通信协议。

---

## 1. 架构概述

```
浏览器 (Vue3)                           机器人 (Python/aiortc)
┌──────────────────────────┐           ┌───────────────────────────┐
│ wo-bot-web-debug         │           │ wo-bot-control             │
│                          │           │                            │
│ RTCPeerConnection        │ 信令      │ RTCPeerConnection          │
│ - 2 x Video Transceiver  │◄── WS ──►│ - CameraVideoTrack (VP8)   │
│   (recvonly)             │  :8765    │ - DataChannel "wobot-      │
│ - DataChannel            │           │   control-srv"             │
│   "wobot-control"        │           │ - ice-lite 模式            │
│                          │           │                            │
│       P2P 直连 ══════════╪═══════════╪══════════════════════════  │
│       UDP/RTP/DTLS       │           │                            │
│  DataChannel (控制指令)   │◄────────►│ DataChannel (状态推送)      │
│  MediaStream (视频渲染)   │◄─────────│ VideoTrack (摄像头帧)       │
└──────────────────────────┘           └───────────────────────────┘
```

WebRTC 承担两重角色：
1. **P2P 控制通道** — DataChannel 承载所有业务消息（消息格式与 WebSocket 完全一致），优先级高于 WebSocket
2. **P2P 视频流** — MediaTrack 传输 VP8 编码的摄像头实时画面，支持双摄像头

---

## 2. 连接建立流程

### 2.1 前提条件

- WebSocket 信令通道已建立（收到 `connected` 消息）
- `connected.features` 包含 `"webrtc"`
- 服务端已安装并启动 aiortc

### 2.2 握手流程

```
1. 客户端: 检查 features 含 "webrtc" → 创建 RTCPeerConnection
2. 客户端: 创建双 Video Transceiver (recvonly)
3. 客户端: 创建 DataChannel "wobot-control"
4. 客户端: createOffer() → setLocalDescription()
5. 客户端: 通过 WebSocket 发送 webrtc_offer → 服务端
6. ─────────────────────────────────────────────
7. 服务端: 接收 offer → setRemoteDescription()
8. 服务端: 创建 CameraVideoTrack → 绑定到 video transceiver (sendonly)
9. 服务端: 回填编解码器（aiortc 0.9.10 workaround）
10. 服务端: createAnswer() → setLocalDescription()
11. 服务端: SDP 后处理（注入 ice-lite、替换 .local → 真实 IP）
12. 服务端: 创建服务端 DataChannel "wobot-control-srv"
13. 服务端: 通过 WebSocket 发送 webrtc_answer → 客户端
14. ─────────────────────────────────────────────
15. 客户端: setRemoteDescription(answer)
16. 双方: ICE candidate 交换 → 连通性检查 → DTLS 握手
17. 双方: DataChannel OPEN → 业务通道就绪
18. 客户端: VideoTrack ontrack → 渲染到 <video>
```

### 2.3 STUN 配置

```
STUN: stun.l.google.com:19302
```

双方使用 Google 公共 STUN 服务器获取公网地址，仅用于同局域网场景。

### 2.4 ICE 策略

| 特性 | 说明 |
|------|------|
| **ice-lite 模式** | 服务端不执行主动 ICE 连通性检查，由浏览器单侧验证 |
| **aioice Monkey Patch** | 修复 ICE 连通性检查问题（Python 3.7 兼容） |
| **IP 地址处理** | 服务端自动探测局域网 IP，SDP 中 `.local` mDNS 替换为真实 IP |
| **双候选策略** | 对客户端 `.local` ICE candidate，保留原始 + 追加高优先级真实 IP 候选 |
| **ICE Fallback** | 浏览器端 connectionState 5s 未变化 → 使用 ICE completed 作为兜底 |
| **Gathering Fallback** | Android/iPadOS WebKit 上 gathering 完成后 8s 仍 connecting → 强制标记 connected |

---

## 3. SDP 信令格式

WebRTC 信令通过 WebSocket 中继，消息格式与业务消息一致。

### 3.1 webrtc_offer（客户端 → 服务端）

```json
{
  "type": "webrtc_offer",
  "data": { "sdp": "v=0\r\no=- ..." }
}
```

客户端 offer 特点：
- 双 Video Transceiver (recvonly)，对应双摄像头
- DataChannel `m=application`（SCTP）
- 等待 answer 超时：10 秒

### 3.2 webrtc_answer（服务端 → 客户端）

```json
{
  "type": "webrtc_answer",
  "data": { "sdp": "v=0\r\no=- ..." }
}
```

服务端 SDP 后处理：
1. 注入 `a=ice-lite`（在第一个 `m=` 行之前）
2. `.local` mDNS 候选替换为真实 IP（从 `config.server.advertised_ip` 或自动探测获取）
3. 视频 transceiver 设为 `sendonly`

### 3.3 webrtc_ice_candidate（双向）

```json
{
  "type": "webrtc_ice_candidate",
  "data": {
    "candidate": "candidate:...",
    "sdpMid": "0",
    "sdpMLineIndex": 0
  }
}
```

服务端对客户端 `.local` 候选的处理：
- 保留原始 `.local` 候选作为兜底
- 追加高优先级真实 IP 候选（foundation 前缀 `h`，priority=2122252543）
- 真实 IP 候选先添加到 `_remote_candidates`，确保优先尝试

### 3.4 webrtc_close（客户端 → 服务端）

```json
{ "type": "webrtc_close", "data": {} }
```

---

## 4. DataChannel 协议

### 4.1 通道规格

| 属性 | 客户端 DC | 服务端 DC |
|------|-----------|-----------|
| label | `wobot-control` | `wobot-control-srv` |
| ordered | true | true |
| 创建方 | 客户端（主动） | 服务端（answer 后） |
| 消息格式 | JSON（与 WebSocket 一致） | JSON（与 WebSocket 一致） |

### 4.2 消息格式

DataChannel 消息格式与 WebSocket **完全相同**：

```json
{ "type": "消息类型", "data": { ... } }
```

支持全部 WebSocket 消息类型（motion、camera、system、exec、gimbal 等），详见 [WebSocket 协议文档](websocket协议（用于局域网通讯）.md)。

### 4.3 发送优先级（前端）

```
DataChannel (readyState="open") > WebSocket (readyState=OPEN) > 内存队列（上限 50 条）
```

- DataChannel 就绪时，所有业务消息优先走 DC
- WebSocket 作为降级通道
- 心跳（ping/pong）始终走 WebSocket 直发

### 4.4 服务端 DC 消息响应

服务端通过 DataChannel 收到的业务消息会回传响应。响应可能被包裹在 `response` 信封中：

```json
// 直接响应
{ "type": "motion_ack", "data": { "linear": 0.5, "angular": 0.3 } }

// 信封响应（服务端消息处理器返回）
{
  "type": "response",
  "data": {
    "type": "camera_status",
    "data": { "cameras": [...] }
  }
}
```

前端自动解包 `response` 信封，与 WebSocket 路径统一处理。

---

## 5. 视频流

### 5.1 视频编码

| 参数 | 值 |
|------|-----|
| 编码格式 | VP8（软编码） |
| 像素格式 | RGB24（BGR → RGB 转换） |
| 帧率 | 单客户端 10fps / 多客户端 5fps |
| 分辨率 | 由 CameraManager 决定（默认 640x480） |
| 无帧时行为 | 推送黑色帧 |

### 5.2 CameraVideoTrack 实现

```
CameraManager.get_frame(camera_id)
  → cv2.cvtColor(BGR → RGB)
    → VideoFrame.from_ndarray(rgb24)
      → VP8 编码
        → RTP/UDP 发送
```

帧丢弃策略：
- 每 30 帧记录一次吞吐量日志
- 编码落后超过 2 倍帧间隔时，丢弃中间帧，只保留最新帧
- 保持低延迟优先

### 5.3 双摄像头支持

客户端创建 2 个 Video Transceiver (recvonly)：

```typescript
peerConnection.addTransceiver("video", { direction: "recvonly" });
peerConnection.addTransceiver("video", { direction: "recvonly" });
```

服务端按摄像头 ID 排序，逐个绑定到对应 transceiver：
- `videoStream0` → 摄像头 0
- `videoStream1` → 摄像头 1

### 5.4 多客户端帧率自适应

```python
client_count = len(self._connections)
track_fps = 5 if client_count > 1 else 10
```

多客户端同时连接时自动降至 5fps，减轻 Jetson Nano VP8 软编码压力。

### 5.5 视频轨生命周期

| 事件 | 处理 |
|------|------|
| `onunmute` | 视频轨开始接收数据 → 标记 `_videoPlaying=true`，清除媒体超时 |
| `onmute` | 视频轨静音 → 记录警告日志 |
| `onended` | 视频轨结束 → 记录警告日志 |
| 媒体超时 | 5 秒内未收到 `onunmute` → 自动重试建立连接（最多 3 次） |

---

## 6. 连接状态管理

### 6.1 前端状态机

```
idle → connecting → connected → (断开) → idle
                 ↘ failed → idle
```

触发 `connected` 的条件（任一满足即切换）：
- ICE state → `connected` 或 `completed`
- connectionState → `connected`
- DataChannel OPEN（本地或远程 DC）
- ICE completed 兜底（5s 内 connectionState 未变化）
- Gathering 完成兜底（WebKit/Android 8s 后仍 connecting）

### 6.2 ICE 状态监控

```typescript
// 监听以下 5 个状态变化：
peerConnection.oniceconnectionstatechange    // ICE 层连接状态
peerConnection.onicegatheringstatechange      // ICE 候选收集状态
peerConnection.onconnectionstatechange        // 整体连接状态
peerConnection.onsignalingstatechange         // 信令协商状态
channel.onopen / onclose                      // DataChannel 状态
```

### 6.3 诊断机制

DTLS 状态诊断（连接建立 3 秒后自动触发）：
```
每个 transceiver 的 DTLS 状态 (ICE/DTLS/encrypted)
SCTP 传输层的 DTLS 状态
```

---

## 7. aioice Monkey Patch

服务端三个关键补丁（`aioice_patch.py`）：

### 7.1 ICE 连通性检查绕过

- **问题**：Python 3.7 环境 + ice-lite 模式下，aiortc 可能触发不必要的连通性检查
- **修复**：Monkey-patch `ice.connection.check_incoming` / `get_candidates`，强制服务端 pair

### 7.2 SCTP DataChannel 重复 OPEN 崩溃容错

- **问题**：DataChannel 已 OPEN 时再次收到 OPEN 消息导致 aiortc 崩溃
- **修复**：Patch `sctp.data_channel_receive`，忽略重复的 OPEN 状态

### 7.3 DTLS 数据流诊断

- **问题**：DTLS 握手失败时缺少诊断信息
- **修复**：包装 `dtls.sendto` / `data_received`，记录数据流日志

---

## 8. 连接清理

### 8.1 清理触发条件

- WebSocket 断开（客户端主动或被动）
- ICE 状态变为 `failed` / `closed` / `disconnected`
- connectionState 变为 `failed` / `closed`
- DataChannel 关闭（远程 DC）
- `webrtc_close` 消息

### 8.2 清理步骤

```
1. 关闭 RTCPeerConnection
2. 移除 DataChannel 引用
3. 停止 CameraVideoTrack
4. 停止摄像头流 (camera_manager.stop_stream)
5. 如果无其他活跃连接，清空全局 ICE 候选队列
```

清理加锁（`_cleaning_up` set）防止并发重复清理。

---

## 9. IP 地址解析（服务端）

对外公告 IP 的解析优先级：

```
1. config.server.advertised_ip（配置文件显式指定）
2. config.server.host（非 0.0.0.0 时）
3. 自动探测: socket.connect(8.8.8.8, 80) → getsockname()
4. fallback: 127.0.0.1
```

配置示例（`config.yaml`）：
```yaml
server:
  host: "0.0.0.0"
  advertised_ip: "192.168.1.47"  # 显式指定对外 IP
  port: 8765
```

---

## 10. 稳定性保障

| 机制 | 说明 |
|------|------|
| 媒体超时自动重试 | 连接后 5s 无视频数据 → 重试，最多 3 次 |
| ICE Fallback 兜底 | connectionState 5s 未变 → 标记 connected |
| Gathering Fallback | 8s 后仍 connecting → 强制 connected（WebKit/Android 兜底） |
| 连接计数器 | 旧 PC 的 onclose/onstatechange 事件检查 `pc.value !== peerConnection` |
| 并发防护 | `_establishing` 标记防止并发 establishConnection |
| SDP 超时 | 等待 answer 最大 10 秒 |
| 旧 resolver 清理 | 新连接时拒绝旧的 answer resolver |

---

## 11. HTTP 降级方案

当 WebRTC 不可用时，提供 HTTP 视频备选：

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/camera/{id}/snapshot` | GET | JPEG 截图 |
| `/api/camera/{id}/stream` | GET | MJPEG 视频流 |
| `/api/status` | GET | 系统状态 |
| `/api/health` | GET | 健康检查 |
| `/api/modules` | GET | 模块列表 |
| `/api/software/install` | POST | 安装软件 |

HTTP API 服务端口：`8000`（FastAPI）
