# AGENT.md

> AI 开发助手在接手 wo-bot 任何子项目开发前必须阅读本文档。
>
> **注意**：文档中使用 `<JETSON_IP>`、`<JETSON_USER>`、`<SERVICE_NAME>` 等占位符表示因环境而异的配置项，请根据实际环境替换。

## 零、接手第一步（强制）

**接手后第一条命令**（在机器人上执行，脚本只读、不改动任何东西）：

```bash
SUDO_PASS='<密码>' bash /opt/wobot/scripts/healthcheck.sh --with-webrtc
```

一次性报告：磁盘、服务自启、**L4T 版本一致性**、apt/dpkg 元数据完整性、摄像头与 Argus、
音频、venv/aiortc、端口、绑定客户端数、WebRTC DTLS 回环。
**先看汇总：有 FAIL 就先修 FAIL，再谈别的。**

**为什么强制**：本项目的故障几乎不来自代码本身，而来自**机器人上积累的隐性状态**
——半升级的 L4T、写满的磁盘、被改写的依赖、未自启的 sshd、损坏的 apt 索引。
不先巡检就动手，就会重复踩已经踩过的坑（历史上每次 Agent 接手都发生过，见下表）。

### 已知地雷（每一个都真实发生过）

| 地雷 | 症状 | 防法 / 现状 |
|---|---|---|
| **L4T 半升级**（内核对不上摄像头用户态） | CSI **绿屏**、`nvargus-daemon` SIGILL/SIGBUS、`SCF addSourceByIndex failed` | 装 L4T 包必须**整套对齐**；healthcheck 第 3 项会报 |
| **磁盘写满** | apt 索引损坏、**dpkg 元数据被写成全 NUL**、GStreamer 插件 SIGILL | 保持 >15% 空闲；healthcheck 第 1 项 |
| `deploy.sh` 删运行时状态 | 绑定/配置/红外码全丢 | 已修（保留 venv/config/data/logs）；**但跑任何部署脚本前先读它做什么** |
| aiortc 被错误补丁改写 | `WebRTC negotiation failed: OpenSSL call failed` | 补丁已删并加自测；healthcheck 会校验断言未被改写 |
| sshd 未设自启 | 重启后彻底失联（只能物理接触） | healthcheck 第 2 项；`systemctl enable ssh` |
| GStreamer 注册缓存损坏 | `gst-inspect` SIGILL/SIGBUS → 摄像头/音频全挂 | 删 `~/.cache/gstreamer-1.0` 后重建 |
| 前端 WebRTC 无自愈 | 摄像头**白屏**（占位层盖住） | 已加 `syncVideoStreams` + `ensureVideoLink` 自愈 |
| **USB 摄像头重枚举** | 画面**卡在最后一帧**（服务端已停帧，前端无提示） | 驱动 oops 后节点会从 `video1` 漂到 `video2`；已加 `_resolve_device_path()` 自动跟随；healthcheck 第 5 项检测漂移节点 |
| 未经核实的"修复" | 服务崩溃循环、停机数分钟 | 见下方改动纪律 |

### 改动纪律

- **不要凭"应该是这样"打补丁**。本项目有过一次因未验证补丁语义（`hasattr` 守卫写反）导致
  服务崩溃循环、停机约 4 分钟的记录。改之前先**验证函数真实返回值**。
- **同一类操作连续失败 2 次就停下来提替代方案**，不要反复重试。
- **动系统（装包 / 改内核 / 改权限）前先备份并告知用户**。
  `/var/lib/dpkg/info`、`config/`、`data/` 都是高价值状态。
- **自己引入的副作用要主动交代**（例如为复现而新增的绑定客户端、留下的备份文件）。
- **只读优先**：能用只读命令看清的，就不要先写。
- **测试要覆盖「调用点」，而不只是被测函数本身**。本项目出现过：辅助方法定义在 A 类、
  却从 B 类调用，`AttributeError` 直接把两个摄像头全部打挂 —— 而单测只测了辅助函数本身，
  **全绿通过**，部署时的 DTLS 自检也照过。凡"新增方法并从别处调用"，
  都必须有一条**真正跑到调用点**的测试（必要时把整个方法跑一遍，硬件依赖用 mock 挡掉）。
- **部署后用真实路径验证一次**，不要只核对文件 md5 就宣布完成。
  md5 一致只能证明"代码传过去了"，不能证明"这条路能跑"。

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

### 仓库结构（含账号级默认文件）

除下列子仓库外，账号下还有一个特殊仓库 **`al96169/.github`**：

- 它存放 **账号级默认社区健康文件**（`CONTRIBUTING.md`、`SECURITY.md`、
  `CODE_OF_CONDUCT.md`、`.github/ISSUE_TEMPLATE/`、`.github/PULL_REQUEST_TEMPLATE.md`），
  GitHub 会把它作为**所有未自带同名文件**的仓库的默认值。
- **修改它会影响账号下全部仓库**（包括与本项目无关的其它项目），因此其中的内容
  写成**通用**措辞；wo-bot 专属的开发规范（如 Python 3.7 约束）放在本文件里，不要塞进去。
- 路径有硬性要求，放错会**静默不生效**：根目录（CONTRIBUTING/SECURITY/CODE_OF_CONDUCT）、
  `.github/ISSUE_TEMPLATE/`（Issue 模板与 config.yml）、`.github/PULL_REQUEST_TEMPLATE.md`。
- **默认 LICENSE 不被支持**，LICENSE 必须逐仓库添加。
- 模板里**不要设 labels** —— 所设标签必须在所有目标仓库中存在，否则提交报错。

本地对应克隆目录：`../.github`（与仓库同名）。

### wo-bot-control（机器人端）

必读文档（按顺序）：
1. `wo-bot-control/docs/development.md` — 本地开发环境搭建
2. `wo-bot-control/docs/deployment.md` — 部署到 Jetson 的流程
3. `wo-bot-control/docs/protocol.md` — WebSocket/WebRTC 协议定义

关键规则：
- 所有新功能通过 `src/modules/extension/` 目录下的模块实现
- 通信层（WebSocket/WebRTC）在 `src/core/` 中，模块通过事件总线解耦
- 部署到 Jetson 用 `scripts/deploy.sh`
- 服务重启用 `sudo systemctl restart <SERVICE_NAME>`（需要 root 密码）
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

### 部署前

`scripts/deploy.sh` **已内置**这一步：它会在停服务之前自动跑一次巡检并打印结果
（只读、失败不阻断），因此"是不是这次部署弄坏的"一眼可判。

需要单独看时：

```bash
SUDO_PASS='<密码>' bash /opt/wobot/scripts/healthcheck.sh --with-webrtc
```

### L4T / 系统包升级（高风险）

装任何 `nvidia-l4t-*` 包时必须**整套对齐**：内核/设备树与摄像头/多媒体用户态
**版本不一致会让 Argus 失效、CSI 摄像头变绿屏**（详见
[踩坑记录/L4T半升级导致CSI绿屏与Argus崩溃.md](踩坑记录/L4T半升级导致CSI绿屏与Argus崩溃.md)）。
升级后必须重启，并用巡检第 3 项确认关键包版本一致。

### 部署到 Jetson

```bash
cd wo-bot-control
bash scripts/deploy.sh
```

部署后必须重启服务：
```bash
ssh <JETSON_USER>@<JETSON_IP> "sudo systemctl restart <SERVICE_NAME>"
```

### 前端生效

前端是静态文件，刷新浏览器即可。部署前端：
```bash
cd wo-bot-web-debug
bash scripts/deploy.sh
```

### 查看 Jetson 日志

```bash
ssh <JETSON_USER>@<JETSON_IP> "journalctl -u <SERVICE_NAME> -f --no-pager"
```

## 六、代码规范

- **Python**：ruff 格式化，类型注解必须，禁止 `print` 日志（用 `logging` 模块）
- **TypeScript/Vue**：ESLint + Prettier，提交前必须通过 lint
- **Commit 格式**：`子项目: 简短描述`（如 `wo-bot-control: 新增外设检测模块`）
- **PR 描述**：必须关联 Issue（`Closes #N` 或 `Ref #N`）

## 七、用户交互规则

1. 密码或 Token 优先存储到本机密码库，如需存文件放在 `secret/` 下
2. `secret/` 下的文件绝对不能提交到代码仓或云端
3. Jetons 凭证在 `secret/jetson.md`，不硬编码
4. GitHub Token 在用户会话中，不记入文件
5. 部署前告知用户改动摘要
6. 遇到需要 root 权限的操作，告知用户原因
7. 同一问题连续失败 2 次，主动建议替代方案而非重复尝试
8. 开发中踩过的坑记录在 `wo-bot-wiki/踩坑记录/`，但不要随意往里面丢东西
9. 开发前先根据需求文档了解项目结构，可以向用户提出不明白的问题，逐一提出，等用户回答后再问下一个，直到你有80%的把握了解需求

## 八、常用操作速查

```bash
# 读 Issue
gh issue view N --repo al96169/wo-bot-control

# 创建 Issue
gh issue create --repo al96169/wo-bot-control --title "xxx" --body "xxx"

# 部署到机器人
cd wo-bot-control && bash scripts/deploy.sh

# 重启机器人服务
ssh <JETSON_USER>@<JETSON_IP> "sudo systemctl restart <SERVICE_NAME>"

# 部署前端
cd wo-bot-web-debug && bash scripts/deploy.sh

# 代码检查
cd wo-bot-control && ruff check src/
cd wo-bot-web-debug && npx eslint src/ --ext .ts,.vue
```
