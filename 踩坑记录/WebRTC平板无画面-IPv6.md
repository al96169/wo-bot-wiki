# WebRTC 平板无画面：ICE IPv4 vs IPv6 不匹配

**现象**：Mac Chrome 有视频，Android 平板 Edge 无画面（ICE=connected, DTLS=connecting 卡死）。

**根因**：Jetson 的 ICE 候选对选用了 IPv4，但平板 DTLS socket 绑定在 IPv6（Android 默认行为）。
Jetson 的 ClientHello 通过 IPv4 发到平板，平板收不到 → DTLS 死锁。

**为何 Mac 不受影响**：Mac Chrome 对 DTLS 角色冲突有容错，且同一网络栈同时监听 IPv4/IPv6。

**修复**：`aioice_patch.py` 的 `connect()` 补丁收集所有兼容候选对后，**IPv6 优先排序**。
平板与 Jetson 同 `/64` 子网（无 NAT），IPv6 直连后 DTLS 握手正常完成。

**无效尝试**（已删除）：
- DTLS 角色反转补丁（controlled↔controlling swap）—— aiortc 0.9.10 虽有角色 Bug，但在 IPv6 同类型网络上不影响握手
- SDP `a=setup:passive` 手动替换——同上的原因，无实际效果

**有效保留**：SCTP DataChannel 重复 OPEN 容错（浏览器 DTLS 重连时可能 crash）。
