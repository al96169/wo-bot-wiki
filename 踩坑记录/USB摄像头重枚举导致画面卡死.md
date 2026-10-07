# USB 摄像头重枚举导致画面"卡死"

> 发现时间：2026-10-07
> 现象：遥控页「摄像头1」开启一段时间后**画面停在最后一帧**，无任何错误提示
> 实际时长：约 **19 分钟**后才偶然自愈（不是永久卡死，但用户感知就是卡死）
> 定性：**应用把设备节点当成了稳定标识符**

## 现象

- 摄像头1（USB）画面**停在最后一帧**不再更新，前端无提示（既不报错也不转圈）
- 服务端日志：
  ```
  17:26:40  camera: cap.read() returning empty frames (5x): USB Camera
  17:26:40  camera: cap.read() failed 10 consecutive times, releasing device: USB Camera
  17:26:40  camera: Auto-restarting camera stream (attempt 1): USB Camera
  17:26:42  camera: Auto-restarting camera stream (attempt 2): USB Camera
    ...     （指数退避，最多 30s 一次，一路重试到第 40 次）
  17:45:15  camera: Camera stream auto-recovered after 40 attempt(s): USB Camera
  ```
- 期间 `/dev/video1` **不存在**，摄像头实际在 **`/dev/video2`**

## 根因

两件事叠加：

**1）USB 摄像头掉线并重新枚举（内核层）**

```
dmesg:
  uvcvideo: Found UVC 1.00 device USB 2.0 Camera (038f:6001)
  uvcvideo 1-2.1:1.0: Entity type for entity Processing 2 was not initialized!
  [<...>] uvc_status_cleanup+0x3c/0x48 [uvcvideo]        ← 内核 oops（同一时段 4 次）
```

`uvcvideo` 驱动崩溃后设备被重新枚举，**拿到了不同的次设备号**：`/dev/video1` → `/dev/video2`。
（之后又一次重枚举，恰好又拿回了 `video1`，旧代码这才在第 40 次尝试时"自愈"。）

**2）应用把节点路径缓存了下来（应用层，真正的缺陷）**

`camera.py` 里：

```python
device = self.camera_info.get("device", ...)   # 检测时定下，之后一直沿用
```

`_auto_restart_with_backoff()` 反复调用 `start()`，但每次都用同一个陈旧的
`camera_info["device"]` → 永远去开已经不存在的 `/dev/video1` → 重试永不成功，
**只能等设备"碰巧"重新枚举回同一个节点号**。

## 修复

新增 `CameraManager._resolve_device_path()`，在 `start()` 打开设备**之前**重新解析节点：

| 情况 | 行为 |
|---|---|
| 旧节点仍存在 | 复用它（不做无谓抖动） |
| 旧节点消失 | **跟随漂移后的新节点** |
| 只剩 CSI 节点 | 返回 None，绝不把 CSI 当 USB 摄像头 |
| CSI 自身 | 不做漂移处理（节点由 nvargus/v4l2 固定占用） |

节点变化时会打日志，便于事后确认修复是否生效：

```
Camera device node changed: /dev/video1 -> /dev/video2 (old node gone, likely USB re-enumeration)
```

单测：`tests/test_camera_device_resolve.py`（4 例，覆盖上表四种情况）。

## 排查方法（可复用）

**画面卡死时，先确认服务端是否还在出帧**，不要先怀疑前端：

```bash
# 1) 服务端还有没有在重启/失败
grep -aE 'cap.read|Auto-restart|Camera stream' /opt/wobot/logs/wobot.log | tail
# 2) 绕开应用，直接抓一帧看硬件是否健康
ls /dev/video*
timeout 20 v4l2-ctl -d /dev/video2 --set-fmt-video=width=640,height=480,pixelformat=MJPG \
  --stream-mmap --stream-count=1 --stream-to=/tmp/t.jpg && file /tmp/t.jpg
# 3) 内核有没有 oops / USB 错误
sudo dmesg | grep -iE 'uvcvideo|usb 1-2' | tail
```

本次就是靠第 2 步定性的：**`/dev/video2` 能正常抓到帧** → 硬件没问题，
是应用找错了门。

## 教训

- **`/dev/videoN` 不是稳定标识符**。USB 设备会重枚举、会换号。要么每次重新解析
  （本次采用），要么改用 `/dev/v4l/by-path/` 这类基于物理拓扑的稳定路径。
- **自动重启逻辑必须每轮重新解析依赖**，否则"重试"只是在重复同一个错误。
- 客户端的"卡死"通常是**服务端早已停止出帧**；前端能做的只是把最后一帧留着
  （以及未来应该给出"画面已中断"的提示）。

## 预防

- `scripts/healthcheck.sh` 第 5 项会检测**漂移节点**（正常只有 `video0`/`video1`，
  出现 `video2+` 即说明 USB 摄像头曾重枚举）并直接给出处理建议
- 根因侧的 `uvcvideo` oops 属内核驱动问题（4.9.337-tegra），无法从应用根治。
  反复出现时考虑：**带供电的 USB Hub**、更换短线、减少 USB 带宽占用

## 修复过程中的二次事故（值得单独记一笔）

第一版修复把 `_resolve_device_path()` 写在了 **`CameraManager`** 上，
却从 **`CameraStream.start()`** 里调用 —— 这是两个不同的类。结果：

```
Failed to start camera stream: 'CameraStream' object has no attribute '_resolve_device_path'
```

**两个摄像头全部打挂。** 而当时的单测用 `__new__` 绕开 `__init__`、只测了辅助函数本身，
**全绿通过**；部署时的 DTLS 自检和巡检也都照过（它们都不经过"打开摄像头"这条路径）。

修正：

1. 把逻辑搬到真正被调用的 `CameraStream` 上 —— 改用 **MJPG/YUYV 特征**区分
   USB 与 CSI（CSI 只有 RG10），不再依赖 `CameraManager` 的摄像头列表
2. **补两条「调用点」测试**：
   - `test_resolve_device_path_exists_on_camera_stream`（属性存在性）
   - `test_stream_start_does_not_raise_attribute_error`（真跑一遍 `start()`，cv2 用 mock 挡掉）
3. 部署后用真机跑一遍解析逻辑验证（含"声称在 video2、实际只有 video1"的漂移场景）

另有两条结论已提炼进 [AGENT.md](../AGENT.md) 的「改动纪律」：

- **测试要覆盖「调用点」，而不只是被测函数本身**
- **部署后用真实路径验证一次**，md5 一致只能证明"代码传过去了"，不能证明"这条路能跑"
