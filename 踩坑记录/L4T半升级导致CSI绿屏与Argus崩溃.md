# L4T 半升级导致 CSI 绿屏与 Argus 崩溃

> 发生时间：2026-09-02 埋下 → 2026-09-25 重启后暴露 → 2026-10-07 定位修复
> 影响：CSI 摄像头（imx219）**完全不可用**，画面为纯绿；`nvargus-daemon` 崩溃循环
> 定位耗时：约 3 小时（走了"硬件坏 → 传感器坏 → ISP 坏"的弯路）

## 现象

- UI「摄像头2」显示**纯绿色**画面；日志里 `First frame: 640x480 mean_BGR=(0.0,154.0,0.0)`
- 日志反复出现：
  - `SCF: Error BadParameter (CameraDriver.cpp addSourceByIndex line 305)`
  - `Acquiring SCF Camera device source via index 0 has failed`
  - `nvargus-daemon` 反复 SIGBUS/SIGILL（`NRestarts` 持续增长）
- `gst-inspect-1.0 nvarguscamerasrc` 退出码 132（SIGILL）或 135（SIGBUS）
- 但**内核层是好的**：`vi 54080000.vi: subdev imx219 8-0010 bound`，`/dev/video0` 存在
- V4L2 raw 通路只能出 `RG10`（10bit Bayer）。应用把它当 YUYV 读 → 高字节全 0 → **纯绿**
- 用户的原话是关键线索：**"但是之前是好的"**

## 根因

`/var/log/dpkg.log` 揭示 **2026-09-02 有一次中断的 L4T 升级**（32.7.5 → 32.7.6）：

```
2026-09-02 20:34:23 remove  nvidia-l4t-bootloader         32.7.5
2026-09-02 20:43:49 upgrade nvidia-l4t-kernel-headers     32.7.5 → 32.7.6
2026-09-02 20:46:03 upgrade nvidia-l4t-kernel-dtbs        32.7.5 → 32.7.6   ← 设备树
2026-09-02 20:47:51 upgrade nvidia-l4t-kernel             32.7.5 → 32.7.6   ← 内核
2026-09-02 20:50:23 upgrade nvidia-l4t-tools              32.7.5 → 32.7.6
2026-09-02 20:50:24 upgrade nvidia-l4t-initrd             32.7.5 → 32.7.6
```

**但摄像头/多媒体的用户态包没有一起升级**，留在 32.7.5：

| 版本 | 包 |
|---|---|
| 32.7.6 | `kernel`、`kernel-dtbs`、`kernel-headers`、`tools`、`initrd`、`xusb-firmware` |
| 32.7.5 | `camera`、`gstreamer`、`multimedia`、`multimedia-utils`、`jetson-multimedia-api`、`cuda`… |

即 **32.7.6 的内核 + 设备树，配 32.7.5 的 Argus/SCF 用户态** → Argus 无法把 imx219
注册为采集源 → daemon 在 `SCF`（闭源组件）里崩溃。

为什么"之前是好的"：8 月时整套都是 32.7.5（一致），Argus 正常，日志里能看到
`CSI subprocess capture started (3 frame files)` 成功记录；9/2 升到一半后，
9/25 那次重启切到 32.7.6 内核，摄像头就再没好过。

## 修复

```bash
sudo rm -rf /var/lib/apt/lists/*      # 顺带修掉损坏的 apt 索引
sudo apt-get update
sudo apt-get install -y nvidia-l4t-camera nvidia-l4t-gstreamer \
  nvidia-l4t-multimedia nvidia-l4t-multimedia-utils nvidia-l4t-jetson-multimedia-api
sudo reboot
```

**只动用户态，不碰内核/bootloader**（内核已经在正常启动，避免引导风险）。

修复后验证：

```
gst-inspect-1.0 nvarguscamerasrc   exit 0        # 之前 132/135
gst-launch-1.0 nvarguscamerasrc sensor_id=0 ...   45647 bytes 真实 1280x720 JPEG
nvargus-daemon                     0 次重启
First CSI frame: 640x480 mean_BGR=(109.2,111.9,114.1)   # 之前 (0,154,0) 纯绿
```

> 仍有 6 个与摄像头无关的包（`apt-source`/`configs`/`core`/`gputools`/`jetson-io`/`oem-config`）
> 停在 32.7.5。**有意不动**：摄像头已经正常，而 `nvidia-l4t-core` 牵涉面广，
> 为了"版本好看"去动它可能把可用状态弄坏。巡检脚本对此只报 WARN。

## 为什么走了弯路（教训）

1. **一开始只查硬件**（传感器、接线、CSI 排线），没查**系统版本一致性**
2. 看到 `nvargus-daemon` 崩溃就当作"闭源组件坏了"，没去问**"它什么时候开始坏的、之前发生了什么"**
3. **用户的一句"但是之前是好的"才是决定性线索** → 立刻去翻 `dpkg.log` / `apt history`
   ——**"什么时候开始坏的"永远比"现在为什么坏"更容易查**

## 预防

- `scripts/healthcheck.sh` 第 3 项专门检查**摄像头关键包版本一致性**，不一致直接 FAIL
- 任何时候装 `nvidia-l4t-*`，都要**整套对齐**（内核/设备树 + 摄像头/多媒体用户态）
- 遵循 [AGENT.md](../AGENT.md) 的「零、接手第一步」

## 相关

- [磁盘写满会静默损坏 apt 与 dpkg 元数据.md](磁盘写满会静默损坏apt与dpkg元数据.md)
  —— 排查期间发现这次还有叠加伤害，两者常一起出现
