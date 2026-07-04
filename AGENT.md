# AGENT.md

> AI 开发助手在接手 wo-bot 任何子项目开发前必须阅读本文档。

## 一、项目上下文

阅读 [项目总览.md](项目总览.md) 了解：
- 项目定位、硬件平台、目标场景
- 所有子仓库地址和职责
- 当前开发阶段与进度
- 通信架构（当前 vs 目标）

## 二、需求获取

**所有需求已迁移到 GitHub Issues，不再使用本地 md 文件。**

- Project 看板：https://github.com/users/al96169/projects/3
- Issue 仓库：https://github.com/al96169/wo-bot-control/issues

### Agent 读取需求的标准流程

```
用户说 "做 #N" 或 "实现 R00035"
  ↓
1. gh issue view N --repo al96169/wo-bot-control
   → 获取完整需求规格（Issue body 包含详细设计）
  ↓
2. Issue body 中有 "参考方案" 链接时
   → Read wo-bot-wiki/方案/xxx.md 获取架构设计
  ↓
3. 开始编码
```

## 三、子项目开发前必读

### wo-bot-control（机器人端）

必读文档（按顺序）：
1. `wo-bot-control/docs/development.md` — 本地开发环境搭建
2. `wo-bot-control/docs/deployment.md` — 部署到 Jetson 的流程
3. `wo-bot-control/docs/protocol.md` — WebSocket/WebRTC 协议定义

关键规则：
- 所有新功能通过 `src/modules/extension/` 目录下的模块实现
- 通信层（WebSocket/WebRTC）在 `src/core/` 中，模块通过事件总线解耦
- 部署到 Jetson 用 `scripts/deploy.sh`，Jetson IP: `192.168.1.47`，用户: `trae`
- 服务重启用 `sudo systemctl restart wo-bot`（需要 root 密码）
- Python 3.10+，严格 asyncio 异步，禁止同步阻塞调用
- 机器人外设操作必须有安全检查和错误恢复

### wo-bot-web-debug（前端）

必读文档（按顺序）：
1. `wo-bot-web-debug/README.md` — 项目启动方式
2. `wo-bot-web-debug/src/composables/useWebSocket.ts` — WebSocket 通信层
3. `wo-bot-web-debug/src/composables/useWebRTC.ts` — WebRTC 通信层

关键规则：
- Vue 3 Composition API + TypeScript，禁止 Options API
- 通信通过 composables（`useWebSocket`, `useWebRTC`）统一管理
- 二进制消息格式：[4B header length big-endian][JSON header][binary data]
- `sendBinary(type, data, binaryData, preferDataChannel)` — preferDataChannel=true 走 DataChannel，否则走 WebSocket
- 移动端兼容必须测试（平板触摸交互不同于鼠标）
- 语音功能涉及 AudioContext / MediaRecorder，需处理浏览器权限

### wo-bot-wiki（本文档仓库）

- 方案文档修改 → 提 PR → Review → 合并
- 协议文档是子项目间的接口契约，修改前需在 Issue 中讨论

## 四、通信协议

### 消息通道

```
浏览器 ←→ 机器人

1. WebSocket：信令 + JSON 控制消息 + 二进制语音数据
2. WebRTC DataChannel：高频控制消息（备用：信令外的所有消息）
3. WebRTC MediaStream：视频流（机器人 → 浏览器）
```

### 二进制消息格式（语音广播专用）

```
[4字节 header_len (big-endian)] [JSON header (UTF-8)] [audio data]
```

JSON header 字段：
- `type`: `"voice_broadcast"`
- `mode`: `"record"` | `"phone"`
- `timestamp`: Unix ms
- `format` (phone mode): `"pcm_s16le"`
- `rate` (phone mode): `48000`

## 五、部署与测试

### 部署到 Jetson

```bash
cd wo-bot-control
bash scripts/deploy.sh
```

部署后必须重启服务：
```bash
ssh trae@192.168.1.47 "sudo systemctl restart wo-bot"
```

### 前端生效

前端是静态文件，刷新浏览器即可。部署前端：
```bash
cd wo-bot-web-debug
bash scripts/deploy.sh
```

### 查看 Jetson 日志

```bash
ssh trae@192.168.1.47 "journalctl -u wo-bot -f --no-pager"
```

## 六、代码规范

- **Python**：ruff 格式化，类型注解必须，禁止 `print` 日志（用 `logging` 模块）
- **TypeScript/Vue**：ESLint + Prettier，提交前必须通过 lint
- **Commit 格式**：`子项目: 简短描述`（如 `wo-bot-control: 新增外设检测模块`）
- **PR 描述**：必须关联 Issue（`Closes #N` 或 `Ref #N`）

## 七、用户交互规则

1. Jetons 凭证在 `secret/jetson.md`，不硬编码
2. GitHub Token 在用户会话中，不记入文件
3. 部署前告知用户改动摘要
4. 遇到需要 root 权限的操作，告知用户原因
5. 同一问题连续失败 2 次，主动建议替代方案而非重复尝试

## 八、常用操作速查

```bash
# 读 Issue
gh issue view N --repo al96169/wo-bot-control

# 创建 Issue
gh issue create --repo al96169/wo-bot-control --title "xxx" --body "xxx"

# 部署到机器人
cd wo-bot-control && bash scripts/deploy.sh

# 重启机器人服务
ssh trae@192.168.1.47 "sudo systemctl restart wo-bot"

# 部署前端
cd wo-bot-web-debug && bash scripts/deploy.sh

# 代码检查
cd wo-bot-control && ruff check src/
cd wo-bot-web-debug && npx eslint src/ --ext .ts,.vue
```
