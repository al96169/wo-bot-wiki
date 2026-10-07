# 小米妙播不能靠 AirConnect 接收（方案证伪）

**最终结论（2026-10-07 决策）**：T022 小米妙播投屏 **判定不做**（无合适方案），
与 T030/T031/T033 处理一致。若未来小米开放接收端协议或出现可信的 Linux 实现，可重新评估。

**现象**：路线图 T022「小米妙播投屏」原计划用 `AirConnect` 桥接，让机器人伪装成妙播设备被小米手机投送。

**结论**：该方案不成立，已证伪。两个独立原因：

**1. AirConnect 的方向是相反的（决定性原因）**

AirConnect 的定位是把 **UPnP / Sonos / Chromecast 音箱**伪装成 **AirPlay 接收端**：

```
iPhone/Mac (AirPlay 发送端) ──> AirConnect ──HTTP──> 真正的 UPnP/Sonos/Chromecast 播放器
```

也就是说它是「AirPlay 接收端 → 网络播放器」的桥，**不是**「手机 → 本机」的接收端。它：

- 不说妙播（MiPlay）协议，无法终止妙播会话
- **没有本地 ALSA 输出路径**，音频只以 HTTP 送给远端播放器，因此根本无法喂给
  `wobot_local` / `wobot_dlna` / `wobot_airplay` 这套 softvol → dmix 音频层
- 与本机已装好的 shairport-sync（AirPlay 本地渲染）功能重叠且无增益

参考：https://github.com/philippe44/AirConnect

**2. 妙播是小米私有协议，没有官方 Linux 接收端**

妙播不是 DLNA/UPnP，也不是 Miracast/AirPlay/Chromecast，而是小米的 MiPlay / MiConnect(Lyra) 栈：

- mDNS 需要**同时**通告 `_lyra-mdns._udp`(SRV 5353) 与 `_mi-connect._udp`(SRV 56666)
- 控制通道 UDP 55982，KCP 风格分帧 + protobuf
- 安全：SafetyAuth ECDH P-256 + HKDF-SHA256 + AES-256-GCM
- 媒体：TCP 8899 + 动态 RTSP/RTP，载荷为**加密 AAC**

小米官方支持矩阵只列小米音箱 / 小米电视 / 装小米电脑管家的笔记本 / 小米汽车，
**没有任何第三方接收端**。因此标准 UPnP MediaRenderer（gmediarender 之类）
**不会**出现在妙播选择器里。

**排查动作**（真机上先抓包再决定是否投入）：

```bash
# 手机打开妙播并扫描时抓包
sudo tcpdump -i any -n -s0 -A 'udp port 5353 or udp port 56666 or udp port 55982 or tcp port 8899'

# 对照看 mDNS 服务与 SSDP 设备
avahi-browse -art | grep -Ei 'lyra|mi-connect'
gssdp-discover --timeout=3     # gupnp-tools
```

- 看到 `_lyra-mdns` / `_mi-connect` 查询 → 只有实现 MiPlay 才可能被投送
- 只看到 1900 端口的 SSDP `M-SEARCH` → 说明走的是 DLNA，与妙播按钮无关

**如果确实要做妙播接收**，只有两条路，且都没有官方保障：

| 路线 | 说明 | 风险 |
|---|---|---|
| 移植 FusionPlay MiPlay SDK | 纯 Rust 反向工程实现，含 mDNS/KCP/SafetyAuth/AAC 解码；**未发布 Linux 目标**，需自行加 `aarch64-unknown-linux-gnu` | AGPL-3.0-only（传染性许可）；能否被手机接受仍受手机端准入策略影响 |
| 试点 MiXPlay / mixplay-hub | 第三方，宣称 Linux arm64 可接收妙播（Docker + `/dev/snd`） | 闭源/付费档，可信度未验证 |

**顺带记录**：`wo-bot-control/src/sub_services/py_dlna_renderer.py`（纯 Python UPnP 渲染器）
全库无任何引用，是死代码；DLNA 实际用的是 `gmediarender`（见 `music_player._start_dlna`）。
