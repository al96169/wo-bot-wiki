# 本地调试部署方案

## 概述

本方案描述如何在开发电脑上运行 wo-bot-market 服务，让局域网内的机器人通过 HTTP 直接获取软件包，实现本地调试部署闭环，避免每次测试都上传到公网服务器。

整体架构：

```
┌─────────────────────┐     HTTP API      ┌─────────────────────┐
│   开发电脑 (macOS)    │ ◄──────────────► │    机器人 (Jetson)    │
│                     │                   │                     │
│  wo-bot-market      │  /api/manifest    │  wobot-control      │
│  ├─ server.py:9099  │  /api/packages/*  │  ├─ software_manager│
│  ├─ manifest.json   │                   │  └─ ...             │
│  └─ packages/*.deb  │                   │                     │
└─────────────────────┘                   └─────────────────────┘
```

## 1. Market 服务本地启动

### 启动

```bash
cd wo-bot-market
PYTHONPATH= python3 server.py  # 清除 PYTHONPATH 避免版本冲突
```

服务监听 `0.0.0.0:9099`，提供以下端点：

| 端点 | 用途 |
|------|------|
| `GET /api/manifest` | 返回白名单 manifest.json |
| `GET /api/packages/<filename>` | 下载制品文件，支持 Range 断点续传 |

### 验证

```bash
curl http://192.168.1.14:9099/api/manifest | python3 -m json.tool
curl -sI http://192.168.1.14:9099/packages/wobot-control_1.0.3_arm64.deb
```

## 2. 构建 .deb 包

使用 `build_deb.sh` 在 macOS 上构建，兼容 `ar` 打包（无需 dpkg-deb）：

```bash
# 脚本位置: wo-bot-control 项目根
bash build_deb.sh
```

### 关键改进点

- **postinst 自动初始化 config.yaml**：首次安装时从 `config.yaml.example` 自动创建，避免 purge 后配置丢失
- **prerm 非阻塞停止**：先 `pkill` 子进程，再用 `systemctl stop --no-block`，避免旧版本 prerm 阻塞 dpkg
- **macOS 兼容**：用 `rsync` 替代 `cp`、`xattr -rc` 去除扩展属性、`tar --no-xattrs` 打包

## 3. 自升级机制（Detached dpkg）

### 问题背景

wobot-control 升级自身时存在**自杀问题**：

```
dpkg -i wobot-control_1.0.3.deb
  → 触发旧包 prerm
    → systemctl stop wobot-control
      → systemd 杀死 wobot-control cgroup 内所有进程
        → software_manager (本进程) 被杀死
        → dpkg 子进程 被连带杀死
        → 升级中断，dpkg 状态变成 iFR
```

### 解决方案：Detached dpkg

`software_manager._dpkg_install` 对 wobot-control 自升级采用**三步策略**：

**Step 1** — 生成独立的 bash 安装脚本，写入 `/tmp/wobot-dpkg-wobot-control.sh`

**Step 2** — 通过 `systemd-run --scope`（或回退到 `setsid`）将脚本运行在**独立 cgroup** 中，即使 wobot-control 的 cgroup 被销毁，dpkg 也不受影响

**Step 3** — 脚本内容（Phase 流程）：

```
Phase 0: pkill -f software_manager & src/main.py（抢占式杀进程，避免 prerm 阻塞）
Phase 1: dpkg -i --force-depends <deb_path>
Phase 2: systemctl start wobot-control → 健康检查 curl /api/health（20s 超时）
Phase 3: 健康检查失败 → dpkg -i 回滚到 1.0.2 备用包 → systemctl start
```

`software_manager` 在启动 detached 脚本后**立即返回成功**（optimistic success），因为自身即将被 Phase 0 杀死，无法轮询结果。前端收到 `requires_reconnect: true` 后等待服务重启重连。

### 关键代码位置

| 文件 | 方法/位置 | 作用 |
|------|-----------|------|
| `software_manager.py:627` | `_dpkg_install` | Detached 安装入口 |
| `software_manager.py:558` | `_url_install` | 自升级不清理 .deb 文件 |
| `build_deb.sh:62` | prerm 脚本 | 非阻塞停止服务 |
| `scripts/updater.sh` | — | 非自升级场景的 Sidecar 升级脚本 |

### 健康检查 URL

**正确端点**：`http://127.0.0.1:8000/api/health`（返回 `{"status": "ok"}`）

> 注意：`/health` 不是有效端点，必须使用 `/api/health`。

## 4. 完整部署流程

### 首次部署

```bash
# 1. SSH 到机器人
ssh trae@192.168.1.47

# 2. 安装 wobot-control
sudo curl -s http://192.168.1.14:9099/packages/wobot-control_1.0.2_arm64.deb \
  -o /tmp/wc-102.deb
sudo dpkg -i --force-depends /tmp/wc-102.deb

# 3. 验证
dpkg -l | grep wobot       # 期望: ii  wobot-control  1.0.2
systemctl is-active wobot-control  # 期望: active
curl -s http://127.0.0.1:8000/api/health  # 期望: {"status":"ok"}
```

### 自升级测试流程

```bash
# 前置条件：
# 1. market 服务在开发电脑上运行
# 2. config.yaml 中 market_endpoint 指向开发电脑 IP
# 3. manifest.json 中 wobot-control latest_version 高于当前版本

# 在机器人上预放回滚备用包
sudo curl -s http://192.168.1.14:9099/packages/wobot-control_1.0.2_arm64.deb \
  -o /tmp/wobot-control_1.0.2_arm64.deb

# 通过 Web 界面点击升级按钮
# 或通过 WebSocket 直接发送升级命令
```

### 验证升级结果

```bash
# 升级后（等待约 30 秒服务重启）
dpkg -l | grep wobot                  # 期望: ii  wobot-control  1.0.3
cat /opt/wobot/version.txt            # 期望: 1.0.3
systemctl is-active wobot-control     # 期望: active
cat /tmp/wobot-dpkg-wobot-control.log # 查看 detached 脚本完整链路
cat /tmp/wobot-dpkg-wobot-control.status  # 期望: 0
```

## 5. 故障排查

### dpkg 状态异常（iFR）

```bash
# 清理半配置状态
sudo dpkg --remove --force-depends wobot-control
# 重新安装
sudo dpkg -i --force-depends /tmp/wc-102.deb
```

### market unavailable

```bash
# 检查 config.yaml 是否丢失
cat /opt/wobot/config/config.yaml | grep market_endpoint
# 确认 market 连通性
curl -s http://192.168.1.14:9099/api/manifest
```

### 升级日志查看

```bash
cat /tmp/wobot-dpkg-wobot-control.log      # Detached dpkg 日志
cat /tmp/wobot-dpkg-wobot-control.status   # 退出码（0=成功）
sudo journalctl -u wobot-control -n 50      # systemd 服务日志
cat /var/log/wobot-updater.log             # updater.sh 日志
```

## 6. 配置文件管理

### config.yaml 生命周期

- **构建时**：`build_deb.sh` 将 `config.yaml` 重命名为 `config.yaml.example` 打包
- **首次安装**：`postinst` 脚本检测 `config.yaml` 不存在时自动从 `.example` 复制
- **升级时**：`dpkg -i` 保留已有 `config.yaml`，不被覆盖
- **purge 后**：reinstall 时重新从 `.example` 创建

### 机器人配置要点

```yaml
# /opt/wobot/config/config.yaml
software_manager:
  market_endpoint: "http://192.168.1.14:9099"  # 指向开发电脑 IP

security:
  auth_enabled: false  # 局域网测试关闭认证

binding:
  enabled: true
  password: "wobot123"
```
