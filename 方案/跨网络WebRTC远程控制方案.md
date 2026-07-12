# 跨网络 WebRTC 远程控制方案

> 目标：支持浏览器/手机App在任何网络环境下（跨WiFi、4G/5G）连接和控制机器人，同时保持端到端加密和低延迟。

---

## 一、背景与动机

### 1.1 当前架构局限

当前通信架构为**同局域网 P2P**模式：

```
[浏览器] ← WebSocket (ws://192.168.1.47:8765) → [机器人 Jetson]
[浏览器] ← WebRTC DataChannel + 视频流 → [机器人 Jetson]
```

**问题**：浏览器必须与机器人在同一局域网。一旦移动到其他 WiFi 或使用移动网络，完全无法连接。

### 1.2 目标架构

```
                              ┌──────────────────┐
                              │   云服务器         │
                              │  (公网 IP + 域名)  │
                              │                   │
                              │  ┌─────────────┐  │
                              │  │ coturn      │  │  ← TURN 中继 (3478)
                              │  │ TURN/STUN   │  │
                              │  └─────────────┘  │
                              │                   │
                              │  ┌─────────────┐  │
                              │  │ wo-bot-signal│  │  ← 信令 + 授权 (WSS)
                              │  │ Node.js      │  │
                              │  └─────────────┘  │
                              │                   │
                              └────┬───┬──────────┘
                                   │   │
                    WSS (信令)     │   │     WSS (信令)
                    WebRTC P2P/Relay│   │     WebRTC P2P/Relay
                                   │   │
                    ┌──────────────┘   └──────────────┐
                    ↓                                  ↓
          ┌─────────────────┐              ┌─────────────────┐
          │ 浏览器 / App     │              │ 机器人 Jetson    │
          │ (任意网络)       │              │ (家庭WiFi)       │
          └─────────────────┘              └─────────────────┘
```

---

## 二、涉及子项目

| 子项目 | 角色 | 变更范围 |
|--------|------|----------|
| **wo-bot-signal** | **新建** — Node.js 信令+授权服务器 | 全新开发 |
| **wo-bot-control** | 机器人后端 (Python) | 连接信令服务器 + 自身鉴权 |
| **wo-bot-web-debug** | 浏览器控制端 (Vue 3) | 连接信令服务器 + 用户登录 |
| **wo-bot-app** | 手机原生客户端 (Flutter) | 连接信令服务器 + 用户登录 |

---

## 三、新仓库：wo-bot-signal

### 3.1 概述

- **仓库地址**：`https://github.com/al96169/wo-bot-signal`（待创建）
- **技术栈**：Node.js + `ws` + `jsonwebtoken`
- **协议**：WSS（WebSocket Secure，TLS 加密）
- **端口**：443（Nginx 反代）

### 3.2 核心模块

```
wo-bot-signal/
├── src/
│   ├── index.ts              # 入口，启动 WSS + HTTP 服务器
│   ├── signaling.ts          # WebRTC 信令转发（offer/answer/ICE）
│   ├── auth.ts               # 用户登录 + Token 签发/验证
│   ├── rooms.ts              # 设备-用户房间管理
│   └── turn.ts               # TURN 短期凭证生成（HMAC）
├── package.json
├── tsconfig.json
└── README.md
```

### 3.3 信令转发（signaling.ts）

WebSocket 连接分两种角色：

| 角色 | 标识 | 职责 |
|------|------|------|
| `robot` | URL 参数 `role=robot` | 机器人自注册，等待信令 |
| `browser` | URL 参数 `role=browser` | 浏览器发起连接请求 |

```javascript
// 房间映射: { deviceId -> robotWs }
const deviceRooms = new Map();

ws.on('message', (msg) => {
  const { type, payload } = JSON.parse(msg);

  switch (type) {
    case 'register':   // 机器人: "我在线，叫 device-abc"
      deviceRooms.set(payload.deviceId, ws);
      break;

    case 'call':       // 浏览器: "我要呼叫 device-abc，这是我的 offer"
      const robotWs = deviceRooms.get(payload.deviceId);
      if (robotWs) robotWs.send(JSON.stringify({ type: 'offer', payload }));
      break;

    case 'answer':     // 机器人/浏览器: 回复 SDP answer
    case 'ice':        // 任意方: 交换 ICE candidate
      // 路由到对方的 WebSocket
      routeToPeer(payload.targetDeviceId || payload.callerId, msg);
      break;
  }
});
```

### 3.4 授权（auth.ts）

**用户侧**（浏览器/App）：
- 邮箱 + 验证码登录 → 服务端签发 JWT（7 天有效）
- JWT 包含 `{ userId, exp }`
- 连接 WSS 时 URL 带 `token=<JWT>`

**机器人侧**：
- 预共享密钥 `ROBOT_SECRET`（云服务器环境变量）
- 连接时带 `role=robot&deviceId=<ID>&signature=<HMAC>`
- 服务端用相同密钥重新计算 HMAC 比对

详见 [第七节：安全性](#七安全性)。

### 3.5 TURN 短期凭证（turn.ts）

每次连接时，信令服务器用 coturn 的 `static-auth-secret` 生成一次性 HMAC 凭证：

```javascript
function generateTurnCredentials(userId) {
  const timestamp = Math.floor(Date.now() / 1000) + 86400; // 24h 有效
  const username = `${timestamp}:${userId}`;
  const password = crypto.createHmac('sha1', TURN_SECRET)
    .update(username).digest('base64');
  return { username, password, ttl: 86400 };
}
```

浏览器和机器人在建立 PeerConnection 前，通过信令服务器获取各自的 TURN 凭证。

---

## 四、现有子项目改造

### 4.1 wo-bot-control（机器人）

#### 改动点

| 文件 | 改动 |
|------|------|
| `config/config.yaml` | 新增 `signal_server` 配置段（URL + 密钥） |
| `src/core/signal_client.py` | **新建** — WebSocket 客户端连信令服务器 |
| `src/core/webrtc_service.py` | 从信令服务器获取 TURN 凭证 + 接收 offer |
| `src/core/websocket_server.py` | **保留** — 局域网直连模式作为降级方案 |

#### signal_client.py 核心逻辑

```python
class SignalClient:
    """连接到云端信令服务器的 WebSocket 客户端"""

    def __init__(self, signal_url: str, device_id: str, secret: str):
        self.signal_url = signal_url
        self.device_id = device_id
        self.secret = secret

    def _build_auth_url(self) -> str:
        timestamp = str(int(time.time()))
        signature = hmac.new(
            self.secret.encode(),
            f"{self.device_id}:{timestamp}".encode(),
            hashlib.sha256
        ).hexdigest()
        return (
            f"{self.signal_url}?"
            f"role=robot&device_id={self.device_id}"
            f"&timestamp={timestamp}&signature={signature}"
        )

    async def connect(self):
        async with connect(self._build_auth_url(), ssl=...) as ws:
            await ws.send(json.dumps({"type": "register", "payload": {"deviceId": self.device_id}}))
            await self._on_offer(ws)  # 等待浏览器的 WebRTC offer
            await self._on_ice(ws)    # 交换 ICE candidate

    async def _on_offer(self, ws):
        # 收到浏览器发来的 SDP offer → 交给 webrtc_service 创建 answer
        ...
```

#### 启动流程变更

```
机器人启动
  ├─ 局域网模式（保留）：启动 websocket_server 监听 ws://:8765
  └─ 云模式（新增）：启动 signal_client 连接 wss://signal.xx.com
       └─ 注册 device_id → 等待浏览器呼叫 → WebRTC 建立
```

### 4.2 wo-bot-web-debug（浏览器控制端）

#### 改动点

| 文件 | 改动 |
|------|------|
| `src/composables/useWebSocket.ts` | 新增 `connectToSignal(serverUrl, token)` 方法 |
| `src/composables/useWebRTC.ts` | TURN 凭证从信令服务器动态获取 |
| `src/components/views/LoginView.vue` | **新建** — 登录/注册页面 |
| `src/components/views/DeviceListView.vue` | **新建** — 已绑定设备列表 |
| `src/App.vue` | 路由守卫：未登录 → 跳转登录页 |

#### 连接流程

```
用户打开网页
  ↓
检测是否已登录（localStorage 中有 JWT）
  ├─ 否 → 登录页（邮箱+验证码 → 获取 JWT）
  ↓
连接信令服务器 wss://signal.xx.com?token=<JWT>
  ↓
获取已绑定设备列表
  ↓
点击设备 → 发送 call → 交换 SDP → WebRTC 建立
  ↓
后续所有业务消息走 DataChannel（与当前局域网模式一致）
```

#### 局域网直连兼容

保留现有 mDNS 发现 + 直连模式。两种模式在前端用连接方式切换：

```
┌──────────────────────┐
│  连接方式             │
│  ○ 局域网直连（mDNS）  │  ← 保留
│  ● 云端远控（信令服务） │  ← 新增
│                      │
│  设备列表（云端远控）：  │
│  ┌──────────────────┐ │
│  │ 🏠 客厅小蜗       │ │  ← 点击连接
│  │ 🏠 卧室小蜗       │ │
│  └──────────────────┘ │
└──────────────────────┘
```

### 4.3 wo-bot-app（手机客户端）

> 当前为 Flutter 骨架阶段，不做具体改造规划，仅列出需求。

- 复用浏览器端相同的信令连接逻辑
- 新增登录/注册页面
- 新增设备绑定/设备列表页面
- 支持扫描机器人二维码快速绑定

---

## 五、设备绑定流程

```
1. 首次绑定（需同局域网，只做一次）
   ┌─────────────────────────────────────────────┐
   │                                             │
   │   [手机/浏览器] ─── WiFi A ─── [机器人]       │
   │        │                        │           │
   │        │  ① 发现机器人(mDNS)     │           │
   │        │  ② 点击"绑定到我的帐号"  │           │
   │        │  ③ 机器人生成 deviceId + │           │
   │        │     HMAC签名 → 发给云端  │           │
   │        │  ④ 云端存储绑定关系      │           │
   │        │     (userId ↔ deviceId)  │           │
   │        │                        │           │
   │   [云服务器: 保存 userId + deviceId]         │
   └─────────────────────────────────────────────┘

2. 日常使用（任何网络，无需配对）
   ┌─────────────────────────────────────────────┐
   │                                             │
   │   [你（任何地方）]     [云信令服务器]  [机器人] │
   │        │                    │           │   │
   │        │ ① 登录 → JWT      │           │   │
   │        │ ② 获取设备列表     │           │   │
   │        │ ③ 点击"客厅小蜗"   │           │   │
   │        │ ─── call ────────→│── 转发 ──→│   │
   │        │ ←── WebRTC P2P / TURN 中继 ──→│   │
   │        │                    │           │   │
   └─────────────────────────────────────────────┘
```

---

## 六、实施阶段

### Phase 1：本地机器人与客户端绑定（→ R00035）✅ 已完成

- [x] 外设检测模块 `peripheral_detector.py`（显示器 / 喇叭 / 云台）
- [x] 屏幕验证码：全屏显示 6 位数字，客户端输入匹配
- [x] 喇叭播报验证码：TTS 播报 4 位数字
- [x] 云台随机转动：随机方向，客户端观察选择
- [x] 密码绑定：PBKDF2 哈希存储，兜底方案
- [x] 绑定持久化：`config/bindings.json` + localStorage
- [x] 前端绑定流程页面
- [x] 分享绑定：生成分享码 + 绑定链接

### Phase 2：授权与信令服务器（→ R00036）

- [ ] 创建 `wo-bot-signal` 仓库
- [ ] 用户帐号系统：注册/登录 + JWT（1h access + 7d refresh）
- [ ] 设备云端注册：本地绑定后同步到云端
- [ ] WebRTC 信令转发（offer/answer/ICE）
- [ ] 机器人 HMAC 鉴权 + 时间戳防重放
- [ ] TURN 短期凭证生成（24h 有效）
- [ ] 云服务器部署：coturn + Nginx + Let's Encrypt + PM2

### Phase 3：跨网络远程连接与远程控制（→ R00037）

- [ ] 机器人 `signal_client.py`：连接信令服务器 + 接收远程 offer
- [ ] 前端远程连接：登录 → 设备列表 → 发起连接
- [ ] 连接模式切换 UI（局域网 / 云端远控）
- [ ] 远程控制功能验证（视频 + 云台 + 运动 + 语音喊话）
- [ ] 断连恢复 + 多种网络拓扑测试

---

## 七、安全性

| 层 | 措施 | 实现方式 |
|---|---|---|
| **传输加密** | TLS/WSS | Nginx + Let's Encrypt |
| **媒体加密** | DTLS | WebRTC 协议自带，零配置 |
| **用户认证** | JWT（7天有效期） | 邮箱 + 验证码登录 |
| **机器人认证** | HMAC-SHA256 预共享密钥 | `ROBOT_SECRET` 环境变量 |
| **设备归属** | userId ↔ deviceId 绑定 | 服务端数据库 |
| **TURN 防滥用** | 24h 短期 HMAC 凭证 | coturn `static-auth-secret` |
| **速率限制** | fail2ban + Nginx limit_req | 防暴力破解 |
| **机器人入站** | 无 | 机器人主动出站连接，不需开放任何端口 |

---

## 八、成本估算

| 项目 | 月费 | 说明 |
|------|------|------|
| 云服务器（1核2G） | ~50元 | 跑 Node.js 信令 + coturn |
| 域名 | ~5元 | 已有 `wo-bot.cn` 可复用 |
| SSL 证书 | 0元 | Let's Encrypt 免费 |
| TURN 带宽 | 按量 | 仅 NAT 穿透失败时走中继，日常 P2P 不消耗 |

---

## 九、架构备注

### 信令服务器断线时

- 已建立的 WebRTC P2P 连接**不受影响**（P2P 直连不经过服务器）
- 但无法建立新连接、无法交换 ICE candidate
- 机器人会定时自动重连信令服务器

### 为什么保留局域网直连模式

- 同一 WiFi 下 P2P 延迟更低（<5ms vs 云端中继 20-50ms）
- 信令服务器挂掉时，局域网照常可用
- 开发调试时更方便

### 与 R00029（4G/5G 远控）的关系

本方案与 R00029 独立并存。R00029 聚焦于机器人自身的 4G/5G 模块上网能力，本方案聚焦于跨网络连接架构。两者互补：
- 机器人插 4G 卡上网 + 本方案 → 机器人可部署在无 WiFi 环境
- 机器人连 WiFi + 本方案 → 客户端可从任何网络接入

