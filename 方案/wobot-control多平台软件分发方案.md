# wobot-control 多平台软件分发方案

> 技术设计文档 · v1.0 · Draft · 2026-07-25

为支持多种 Linux 发行版和异构嵌入式机器人平台，设计一套从构建、分发、安装到 OTA 自升级的完整软件交付体系。

---

## 1 背景与目标

### 1.1 现状回顾

当前 `wobot-control` 的分发机制处于**开发验证阶段**，采用 tar.gz 打包 + SCP 手动推送的方式部署到 Jetson Nano（Ubuntu 18.04, arm64）。整体流程依赖开发者手动执行 `scripts/deploy.sh`，没有自动化构建、没有标准包格式、没有版本化发布。

关键限制包括：

- **无标准包格式** — 没有 .deb 打包，无法利用系统包管理器进行依赖解析和卸载
- **纯手动部署** — 依赖 SSH/SCP 手动推送，不支持批量或远程部署
- **自身不可 OTA 升级** — `software_manager` 中将 wobot-control 标记为 systemd 类型（只读），无法通过软件管理面板升级自身
- **市场服务仅局域网** — `wo-bot-market` 运行在开发电脑，机器人无法从公网获取更新
- **平台单一** — 仅针对 Jetson (arm64 + Ubuntu 18.04 + Python 3.7)，未适配其他嵌入式平台
- **无安全验证** — manifest 无签名，下载仅做 SHA256 校验而无公钥信任链

### 1.2 目标场景

未来需要覆盖的机器人平台分为四个层级：

| 层级 | 系统类型 | 典型平台 | 优先级 |
|------|---------|---------|--------|
| **Tier 1** | Debian 系 | Ubuntu 18.04–24.04, Debian 10–12, Raspbian (树莓派) | 高 |
| **Tier 2** | 其他主流 Linux | Fedora, Arch Linux, openSUSE, CentOS/Rocky/Alma | 中 |
| **Tier 3** | 嵌入式定制系统 | Buildroot, Yocto Project, OpenWrt | 中 |
| **Tier 4** | 国产/异构嵌入式 | 旭日X3派, 香橙派, 瑞芯微 RK3588, 算能 SG200x | 低 |

### 1.3 核心目标

- **一次构建，多平台分发**：通过 CI/CD 矩阵构建，同一版本号产出 amd64/arm64/armhf 三种架构的产物，覆盖主流嵌入式平台。
- **OTA 自升级**：wobot-control 可通过 software_manager 检测、下载、校验并以 sidecar 模式替换自身，支持自动回滚。
- **安全可靠**：GPG 签名 manifest + SHA256 校验 + 自动回滚，构建完整的软件供应链信任链。
- **用户友好**：提供一键安装 bootstrap 脚本，自动检测系统类型并选择最优安装方式。

---

## 2 包格式策略

### 2.1 分层策略

考虑到不同平台的兼容性和易用性，采用**三种包格式分层覆盖**，根据目标系统的能力自动选择最优格式：

| 格式 | 适用系统 | 优势 | 劣势 | 阶段 |
|------|---------|------|------|------|
| `.deb` | Debian 系 | 原生包管理、依赖自动解析、dpkg 生态 | 仅限 Debian 系 | Phase 1 |
| `.tar.gz` (自包含) | 所有 Linux | 通用性强、无需包管理器 | 需手动处理依赖和服务安装 | Phase 2 |
| 单文件可执行 (Nuitka) | 资源受限系统 | 零依赖、极小体积 | 编译复杂、调试困难 | Phase 4 |

### 2.2 Debian 包设计 (.deb)

对 Debian 系平台，`.deb` 是**首选格式**。包内包含预构建的 Python 虚拟环境（含所有依赖）、源代码、systemd 服务文件以及安装/卸载钩子。

```
wobot-control_0.3.0_arm64.deb
├── DEBIAN/
│   ├── control          # 包元数据 (名称/版本/架构/依赖)
│   ├── preinst          # 安装前钩子 (检查 Python 版本)
│   ├── postinst         # 安装后钩子 (enable + start systemd)
│   ├── prerm            # 卸载前钩子 (stop + disable systemd)
│   └── postrm           # 卸载后钩子 (清理残留)
└── opt/wobot/
    ├── venv/            # 预构建的 Python 3.7–3.12 虚拟环境
    ├── src/             # 源代码
    ├── config/          # 默认配置文件
    ├── scripts/         # 运维脚本 (setup_audio, 等)
    ├── updater.sh       # Sidecar 升级脚本
    └── version.txt      # 版本标识
```

> **设计决策**：在 `.deb` 中内嵌预构建 venv 而非声明 `Depends: python3-xxx`，原因是嵌入式平台的 apt 源通常版本陈旧且不完整。预构建 venv 确保依赖版本一致性，代价是包体积较大。

### 2.3 自包含归档设计 (.tar.gz)

对非 Debian 系平台，提供**自包含归档**，结构类似 .deb 但通过 bootstrap 脚本完成安装：

```
wobot-control_0.3.0_linux_arm64.tar.gz
├── opt/wobot/           # 同 .deb 内容
│   ├── venv/
│   ├── src/
│   ├── config/
│   └── ...
└── install.sh           # Bootstrap 脚本
    ├── 检测发行版 (Fedora / Arch / openSUSE...)
    ├── 安装系统依赖 (python3, libcamera, ...)
    ├── 部署 systemd / init.d / openrc 服务
    └── 启动服务
```

### 2.4 架构支持

每种格式均构建以下三种架构的产物：

| 架构标识 | 指令集 | 典型设备 |
|---------|--------|---------|
| `amd64` | x86_64 | Intel NUC, 工控机, 开发机测试 |
| `arm64` | aarch64 (ARMv8) | Jetson Nano/Orin, 树莓派 4/5, 旭日X3派, RK3588 |
| `armhf` | armv7l (ARMv7) | 树莓派 3/Zero 2W, 香橙派 (32位) |

> **关于 Python 版本兼容**：当前 Jetson Nano 运行 Python 3.7，而部分依赖（aiortc, av）的新版本已不再支持。方案是构建时使用目标平台的最低 Python 版本（3.7）来编译 venv，同时利用 `sys.version_info` 做运行时兼容适配。长期来看，建议 Jetson 平台升级到 Ubuntu 20.04+ 以获得 Python 3.8+。

---

## 3 分发通道架构

### 3.1 整体架构

三层分发架构：

```
GitHub Release ──→ wo-bot-market ──→ 机器人
  (制品存储/CDN)    (元数据+代理下载)    (software_manager)
```

- **制品层** — GitHub Release：存储各版本 .deb / .tar.gz 制品、SHA256SUMS、GPG 签名文件，利用全球 CDN 加速
- **元数据层** — wo-bot-market：提供签名后的 manifest.json、版本查询 API、下载代理（302 重定向到 GitHub Release）
- **客户端层** — software_manager：周期拉取 manifest、对比版本、下载校验、触发升级、上报结果

### 3.2 各层职责

| 层级 | 组件 | 职责 | 部署位置 |
|------|------|------|---------|
| **制品层** | GitHub Releases | 存储各版本 .deb / .tar.gz 制品、SHA256SUMS、GPG 签名文件 | github.com (免费 CDN) |
| **元数据层** | wo-bot-market | 提供签名后的 manifest.json、版本查询 API、下载代理（302 重定向到 GitHub Release） | 腾讯云 VPS |
| **客户端层** | software_manager | 周期拉取 manifest、对比版本、下载校验、触发升级、上报结果 | 机器人端 |

### 3.3 市场服务 API 设计

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/manifest` | GET | 完整清单 JSON（含 GPG 签名） |
| `/api/v1/packages/{name}/versions` | GET | 指定包的版本列表 |
| `/api/v1/packages/{name}/latest` | GET | 最新版本的元数据 |
| `/api/v1/download/{name}/{version}/{arch}/{format}` | GET | 302 重定向到 GitHub Release 下载 URL |
| `/api/v1/check?name=X&version=Y&arch=Z` | GET | 轻量更新检查（仅返回是否有新版本） |
| `/health` | GET | 健康检查 |

> **为什么用 302 重定向而非直接托管文件？** GitHub Release 在全球有多层 CDN 加速，且对开源项目完全免费。市场服务只需管理轻量的 manifest.json（几十 KB），不需要承担大文件流量。机器人下载时经市场服务获取实际地址后直接连接 GitHub CDN。

### 3.4 Manifest 结构增强

```json
{
  "version": 2,
  "generated_at": "2026-07-25T12:00:00Z",
  "expires_at": "2026-08-01T12:00:00Z",
  "signature": "-----BEGIN PGP SIGNATURE-----\n...\n-----END PGP SIGNATURE-----",
  "public_key_id": "0xABCD1234",
  "public_key_url": "https://get.wobot.cn/keys/release.pub",
  "packages": {
    "wobot-control": {
      "display_name": "WoBot Control",
      "description": "机器人主控制服务",
      "critical": true,
      "versions": {
        "0.3.0": {
          "release_date": "2026-07-25",
          "changelog": "新增多平台支持；修复摄像头画质；新增自升级能力",
          "min_python": "3.7",
          "max_python": "3.12",
          "assets": {
            "amd64": {
              "deb": { "url": "https://github.com/.../v0.3.0/wobot-control_0.3.0_amd64.deb", "sha256": "abc123..." },
              "tar.gz": { "url": "https://github.com/.../v0.3.0/wobot-control_0.3.0_linux_amd64.tar.gz", "sha256": "def456..." }
            },
            "arm64": { "deb": {...}, "tar.gz": {...} },
            "armhf": { "deb": {...}, "tar.gz": {...} }
          }
        }
      }
    }
  }
}
```

> **关键设计**：`expires_at` 字段确保即使市场服务不可用，机器人也不会无限期缓存旧清单。超过有效期后机器人应停止操作并告警，避免使用过时的签名信息。

---

## 4 安装与部署流程

### 4.1 Bootstrap 一键安装

终端用户通过一行命令完成安装，脚本**自动检测系统环境**并选择最优安装方式：

```bash
curl -fsSL https://get.wobot.cn/install.sh | sudo bash
```

安装流程（5 步）：

1. **环境检测**：识别发行版（/etc/os-release）、架构（uname -m）、Python 版本、可用包管理器
2. **格式选择**：Debian 系且 dpkg 可用 → 下载 .deb；其他 → 下载 .tar.gz + 运行内嵌 install.sh
3. **下载与校验**：从市场服务获取最新版本信息，下载 SHA256SUMS 并校验制品完整性
4. **安装与启动**：.deb 走 dpkg -i；.tar.gz 走内嵌脚本。安装 systemd/init 服务并启动
5. **验证与清理**：等待 5 秒后健康检查（HTTP GET :8000/health），清理下载缓存

### 4.2 首次部署 vs OTA 升级对比

| 场景 | 触发方式 | 安装路径 | 服务处理 | 回滚能力 |
|------|---------|---------|---------|---------|
| **首次部署** | 手动执行 bootstrap 脚本 | 全新安装到 /opt/wobot/ | 创建 systemd service 并 enable | 无 (fresh install) |
| **OTA 升级** | Web 前端 → software_manager | 下载到 .update/ → 原子替换 | stop → 替换 → start | 自动回滚到 .backup/ |

### 4.3 多 init 系统支持

`.tar.gz` 包的 `install.sh` 需要适配不同 init 系统：

| Init 系统 | 典型发行版 | 服务管理方式 |
|-----------|-----------|------------|
| **systemd** | Ubuntu, Debian, Fedora, Arch, CentOS 7+ | systemctl start/stop/enable wobot-control |
| **init.d (SysV)** | 旧版 Debian, CentOS 6, 嵌入式定制 | service wobot-control start / update-rc.d |
| **openrc** | Alpine Linux, Gentoo | rc-service wobot-control start / rc-update add |
| **procd** | OpenWrt | /etc/init.d/wobot-control start |

---

## 5 OTA 自升级机制

### 5.1 核心挑战

wobot-control **升级自身的最大难点**在于：它是运行中的进程，不能简单地覆盖自身文件——Python 模块已被加载到内存，二进制依赖库（av, OpenCV）持有文件句柄。需要一个外部机制来停止旧进程、替换文件、启动新进程。

### 5.2 Sidecar Updater 模式

升级流程分为三个角色协作：

```
software_manager → [检测新版本 + 下载校验 + 通知 Web 前端]
     ↓
Web 前端 → [用户确认升级] → software_manager
     ↓
software_manager → updater.sh → [stop → 替换 → start → 健康检查 → 回滚 if 失败]
```

software_manager 作为 wobot-control 的子进程负责下载和校验，触发独立的 updater.sh 脚本执行原子替换。

### 5.3 升级详细流程

1. **更新检测**：software_manager 周期（30 分钟）从市场服务拉取 manifest，对比本地 `/opt/wobot/version.txt`
2. **通知用户**：通过 WebSocket 推送 `software_updates_available` 到 Web 前端。用户点击"升级"确认
3. **下载与校验**：根据架构选择对应制品下载到 `/opt/wobot/.update/`，SHA256 校验。校验失败则终止并报告
4. **预备阶段**：解压到 .update/，向所有 WebSocket 客户端发送"即将重启"通知（30 秒倒计时）
5. **原子替换（updater.sh）**：software_manager 调用独立脚本：`systemctl stop` → 备份旧版本到 .backup/ → `mv .update/wobot-control /opt/wobot/` → `systemctl start`
6. **健康检查与回滚**：等待 10 秒，检查端口 8765/8000 是否可达。若健康检查失败：自动从 .backup/ 恢复旧版本并启动

### 5.4 updater.sh 伪代码

```bash
#!/bin/bash
set -euo pipefail
INSTALL_DIR="/opt/wobot"
STAGING="$INSTALL_DIR/.update"
BACKUP="$INSTALL_DIR/.backup"
HEALTH_URL="http://127.0.0.1:8000/health"
TIMEOUT=15

echo "[updater] Stopping wobot-control..."
systemctl stop wobot-control

echo "[updater] Backing up current version..."
rm -rf "$BACKUP"
mv "$INSTALL_DIR/src" "$INSTALL_DIR/venv" "$INSTALL_DIR/config" "$BACKUP/" 2>/dev/null || true

echo "[updater] Deploying new version..."
mv "$STAGING/src" "$STAGING/venv" "$STAGING/config" "$INSTALL_DIR/"
echo "$NEW_VERSION" > "$INSTALL_DIR/version.txt"

echo "[updater] Starting new version..."
systemctl start wobot-control

echo "[updater] Health check..."
for i in $(seq 1 $TIMEOUT); do
  if curl -sf "$HEALTH_URL" > /dev/null 2>&1; then
    echo "[updater] Health check passed! Upgrade successful."
    rm -rf "$STAGING" "$BACKUP"
    exit 0
  fi
  sleep 1
done

echo "[updater] Health check FAILED! Rolling back..."
systemctl stop wobot-control
rm -rf "$INSTALL_DIR/src" "$INSTALL_DIR/venv" "$INSTALL_DIR/config"
mv "$BACKUP/src" "$BACKUP/venv" "$BACKUP/config" "$INSTALL_DIR/"
systemctl start wobot-control
echo "[updater] Rollback complete."
exit 1
```

> **关键约束**：updater.sh 必须作为**独立进程**运行（通过 nohup 或 at 调度），因为在替换过程中 software_manager 自身也会被停止。实际调用方式为：`nohup /opt/wobot/updater.sh > /var/log/wobot-updater.log 2>&1 &`

### 5.5 升级安全矩阵

| 检查点 | 方法 | 失败处理 |
|--------|------|---------|
| Manifest 签名验证 | GPG verify (预置公钥) | 终止升级，报告"清单签名无效" |
| 制品 SHA256 校验 | 对比 manifest 中的 sha256 | 删除下载文件，报告"校验失败" |
| 磁盘空间检查 | df -h 检查 > 2x 包大小 | 终止升级，提示用户清理空间 |
| 版本号校验 | 新版本 > 当前版本 | 跳过（已是同版本或更新） |
| 健康检查 | HTTP GET /health (15s 内响应) | 自动回滚到 .backup/ |

---

## 6 安全与签名验证

### 6.1 信任链模型

三级信任链：

```
开发者 GPG 私钥 → 签名 manifest.json
         ↓
manifest.json.sig → 包含每个包的 SHA256
         ↓
SHA256SUMS → 每个制品的哈希
         ↓
制品文件 (.deb / .tar.gz)
```

机器人预置公钥验证 manifest 签名 → manifest 中的 sha256 校验下载的制品

### 6.2 公钥分发策略

- **编译时嵌入**：GPG 公钥硬编码在 `software_manager.py` 中，编译进 venv
- **首次安装**：bootstrap 脚本下载 `get.wobot.cn/keys/release.pub` 并通过 TLS 完整性保证
- **公钥轮换**：manifest.json 可携带 `public_key_url` 指向新公钥，由当前公钥签名过渡

### 6.3 安全威胁模型

| 威胁 | 缓解措施 |
|------|---------|
| 中间人篡改制品 | SHA256 校验（manifest 签名保证哈希不被篡改） |
| 中间人篡改 manifest | GPG 签名 + 有效期（expires_at） |
| 回滚攻击（重放旧 manifest） | manifest 含 `generated_at` + 有效期，客户端检查时间戳单调递增 |
| 市场服务被入侵 | 攻击者无法伪造 manifest 签名（没有私钥）；仅可 DoS（拒绝服务） |
| 恶意 .deb 包 | 白名单机制（仅 manifest 中的包可安装）；关键包禁止卸载 |

---

## 7 CI/CD 发布流水线

### 7.1 流水线概览

```
推送 tag v*.*.* → CI: ruff + mypy + pytest → 质量检查通过?
    → 否: 通知失败
    → 是: 多架构构建(amd64/arm64/armhf)
        → 生成 SHA256SUMS
        → GPG 签名
        → 创建 GitHub Release
        → 上传制品
        → 更新 manifest.json
        → GPG 签名 manifest
        → 部署 manifest 到 market 服务器
        → 通知发布成功
```

### 7.2 构建矩阵设计

通过 GitHub Actions 的 matrix 策略 + QEMU 用户态模拟，在单次 workflow 中产出 6 个制品（3 架构 × 2 格式）。QEMU 模拟 arm64/armhf 环境来构建对应架构的 venv，无需物理设备。

```yaml
# .github/workflows/release.yml (关键部分)
jobs:
  build:
    strategy:
      matrix:
        arch: [amd64, arm64, armhf]
        format: [deb, tar.gz]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-qemu-action@v3    # QEMU 模拟 arm64/armhf
      - name: Build for ${{ matrix.arch }}
        run: |
          docker run --rm -v $PWD:/workspace \
            --platform linux/${{ matrix.arch }} \
            python:3.7-slim bash -c "
              cd /workspace &&
              pip install --target=/workspace/build/venv -r requirements-jetson.txt &&
              cp -r src config scripts /workspace/build/
            "
      - name: Package as ${{ matrix.format }}
        run: scripts/package.sh ${{ matrix.arch }} ${{ matrix.format }}
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: wobot-control_${{ github.ref_name }}_${{ matrix.arch }}.${{ matrix.format }}
          path: dist/*
```

### 7.3 分发包签名流程

构建完成后，在 GitHub Actions 中使用存储在 Secrets 中的 GPG 私钥对 SHA256SUMS 和 manifest.json 分别签名：

```bash
- name: Sign artifacts
  env:
    GPG_PRIVATE_KEY: ${{ secrets.GPG_PRIVATE_KEY }}
    GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
  run: |
    echo "$GPG_PRIVATE_KEY" | gpg --batch --import
    cd dist
    sha256sum *.deb *.tar.gz > SHA256SUMS
    echo "$GPG_PASSPHRASE" | gpg --batch --yes --pinentry-mode loopback \
      --passphrase-fd 0 --detach-sign --armor SHA256SUMS
```

### 7.4 Manifest 自动更新与部署

Release 创建后，通过一个独立的 workflow job 拉取现有 manifest.json、追加新版本信息、重新签名并推送到 market 服务器（通过 SSH 或 Webhook）：

```bash
- name: Update and deploy manifest
  run: |
    python scripts/update_manifest.py \
      --version ${{ github.ref_name }} \
      --release-url ${{ github.event.release.html_url }}
      
    scp manifest.json manifest.json.sig \
      wobot@vps:/opt/wobot-market/
```

---

## 8 迁移路径

### 8.1 从当前方案到目标方案的过渡

| 阶段 | 内容 | 时间 |
|------|------|------|
| **Phase 1** | .deb 打包脚本、GitHub Actions 多架构 CI、manifest 签名 | 即刻 |
| **Phase 2** | Sidecar updater 实现、市场服务公网部署、Bootstrap 安装脚本 | 1–2 周 |
| **Phase 3** | .tar.gz 通用包、多 init 系统支持、自动回滚验证 | 1–2 月 |
| **Phase 4** | Nuitka 编译探索、APT 仓库、增量更新 (delta) | 3+ 月 |

### 8.2 兼容性保障

- **现有 Jetson 设备**：支持从当前 tar.gz 手动部署迁移到 .deb 包（`dpkg -i` 覆盖安装）
- **配置文件保留**：升级时 `config/config.yaml` 标记为 conffile（dpkg 不会覆盖用户修改过的配置）
- **回退路径**：旧版 tar.gz 手动部署仍然可用，不强制升级到新方案
- **protocol 版本**：WebSocket 协议保持向后兼容，frontend 和 control 独立升级

---

## 9 风险与缓解

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|---------|
| Python 3.7 EOL 导致依赖不可用 | 高 | 中 | 锁定依赖版本（requirements-jetson.txt）；长期推动 Jetson 升级到 Ubuntu 20.04+ |
| updater.sh 升级失败导致机器人不可用 | 高 | 低 | 自动回滚机制 + 健康检查；保留 .backup/ 直到下次成功升级 |
| GPG 私钥泄露 | 高 | 低 | 私钥仅存于 GitHub Secrets；支持公钥轮换流程；硬件安全模块（HSM）长期方向 |
| QEMU 模拟构建的 arm 包与原生机行为不一致 | 中 | 中 | 添加 arm64 原生 CI runner（Jetson 或树莓派）；关键版本手动验证 |
| 网络受限环境无法连接市场服务 | 中 | 中 | 支持离线升级（本地 manifest.json + USB 导入制品） |
| 跨平台依赖冲突（如 OpenCV、av 的 C 扩展） | 中 | 中 | 预构建 venv 针对每个架构独立编译；提供纯 Python 回退路径 |

---

## 10 路线图

| 里程碑 | 内容 | 预计时间 |
|--------|------|---------|
| **M1** | **基础分发**：.deb 打包脚本 + 目录结构规范化，GitHub Actions 质量检查 CI (已有)，software_manager 支持 systemd 类型升级 | 当前 → 1 周 |
| **M2** | **自动化发布**：多架构 CI 构建 (amd64 + arm64)，GPG 签名 manifest，GitHub Release 自动发布 | 1 → 2 周 |
| **M3** | **OTA 能力**：Sidecar updater 实现，健康检查 + 自动回滚，市场服务公网部署 + API 增强 | 2 → 4 周 |
| **M4** | **多平台覆盖**：.tar.gz 通用包 + armhf 架构，多 init 系统服务配置，Bootstrap 一键安装脚本 | 4 → 8 周 |
| **M5** | **体验优化**：离线升级包导入，升级进度可视化，Rust 重写 updater（零依赖、极小体积） | 8 → 12 周 |
| **M6** | **极致优化**：Nuitka 编译单文件可执行，APT 仓库托管，增量更新（bsdiff/delta） | 12 周 + |
