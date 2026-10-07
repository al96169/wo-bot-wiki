# deploy.sh 的坑：会删运行时状态 / 参数传错 / aiortc「补丁」反而是病根

**发现日期**：2026-10-07（真机部署 T017 静音时暴露）

在真机（192.168.1.47 Jetson Nano）上跑 `wo-bot-control/scripts/deploy.sh` 时一次性踩到多个问题，
其中第三个直接导致 **WebRTC 完全不可用**。均已修复。

---

## 坑 1：部署会删掉设备上的运行时状态（会丢绑定、红外码库）

**现象**：远端脚本用

```bash
find ${REMOTE_DIR} -mindepth 1 -maxdepth 1 ! -name 'venv' -exec rm -rf {} +
```

清掉 `/opt/wobot` 下除 `venv` 外的**所有**内容，然后只还原两个文件
（`config/.binding_secret`、`config/.binding_password`）。

**后果**：

| 被删内容 | 后果 |
|---|---|
| `config/bindings.json` | 所有已绑定客户端失效（当时有 6 个） |
| `data/ir_codes/` | 学习到的红外码库（R00020）丢失 |
| `config/config.yaml` | 被仓库里较旧的默认模板覆盖，设备上的实际配置回退 |
| `logs/` | 日志历史被清 |

**修复**：保留 `venv/config/data/logs`；解压前后用 `sudo tar` 备份/还原 `config/` + `data/`
（`tar` 以 root 运行可保留属主与权限），使**设备配置优先于仓库默认值**；备份失败即中止部署。
`logs/` 只保留不打包 —— 该目录当时有 739MB（见坑 4），打包会撑爆磁盘。

---

## 坑 2：远端脚本收不到参数（且会在本地执行参数）

**现象**：参数被写在 heredoc 结束标记 `SCRIPT` **之后**：

```bash
eval "${SSH_CMD} ... 'bash -s' <<SCRIPT
${REMOTE_SCRIPT}
SCRIPT
${REMOTE_DIR} ${PACKAGE_NAME} ... ${REMOTE_PASSWORD}"     # ← 错在这里
```

heredoc 结束标记即命令结束，后面的内容会被当成**下一条本地命令**执行
（报 `bash: /opt/wobot: No such file or directory`），远端 `$1..$5` **全是空**。

**后果**：远端 `REMOTE_DIR` 为空，于是

```bash
find ${REMOTE_DIR} -mindepth 1 -maxdepth 1 ! -name 'venv' -exec rm -rf {} +
```

等价于在远端 home 目录（`/home/trae`）里删东西 —— **有删错目录的真实风险**。

**修复**：参数写到 ssh 命令行上，heredoc 只负责通过 stdin 喂脚本。
同时新增 `--sudo-password`，把「SSH 认证方式」与「远端 sudo 密码」解耦
（原设计传 sudo 密码只能用 `--password`，那会强制走 `sshpass`，无 tty 环境不可用）。

---

## 坑 3（最严重）：aiortc 根本不需要打补丁，而「补丁」把 WebRTC 打死了

### 症状

前端报错：

```
WebRTC negotiation failed: OpenSSL call failed
```

机器人侧 traceback：

```
aiortc/rtcdtlstransport.py, line 217, in _create_ssl_context
    _openssl_assert(lib.SSL_CTX_set_read_ahead(ctx, 1) if hasattr(lib, "SSL_CTX_set_read_ahead") else 0 if hasattr(...) else 0 == 0)
aiortc.rtcdtlstransport.py, line 58, in _openssl_assert
    raise DtlsError("OpenSSL call failed")
```

### 根因：`hasattr` 守卫把断言语义改反了

**原始代码**（pip wheel 里的 `aiortc 0.9.28`）是：

```python
_openssl_assert(lib.SSL_CTX_set_read_ahead(ctx, 1) == 0)     # 断言"返回 0"
```

deploy.sh 的补丁把调用替换成三元表达式，`== 0` 被吃掉，变成：

```python
_openssl_assert(lib.SSL_CTX_set_read_ahead(ctx, 1) if hasattr(...) else 0)
```

`_openssl_assert` 要求参数 `== 1`。而该调用实测**返回 0** → `0 != 1` → 必然抛
`DtlsError: OpenSSL call failed`。也就是说这个"保护"从第一天起就是错的。

### 实测结论：这个机器人上两个补丁都不需要

```python
hasattr(lib, "SSL_CTX_set_read_ahead")  →  True     调用返回 0    # 原始断言 == 0 本来就通过
hasattr(lib, "BIO_ctrl_pending")        →  True     返回 0        # 原始 _write_ssl 本来就正确
hasattr(lib, "BIO_ctrl")                →  False                  # 但原始代码并不用它
```

**所以正确做法是：不打任何补丁。** 把 venv 里的 `rtcdtlstransport.py` 还原成 wheel 原始文件后，
DTLS 回环自测（SDP→ICE→DTLS→DataChannel）一次通过：

```
DataChannel 状态 = open
连接状态        = completed
✓ 对端收到消息: ['hello-from-pc1']
```

### 附带的次生 bug：补丁不幂等

同一份补丁的幂等守卫检查的字符串是 `'# NOTE: BIO_ctrl_pending is not available'`，
但它实际写入的标记是 `'pass  # patched: cryptography binding lacks BIO_ctrl*'` —— 永不相等，
于是**每次部署都重复叠加补丁**；叠加后正则替换留下的缩进错乱，还会产生
`IndentationError`，让服务进入崩溃重启循环（连续跑两次 `deploy.sh` 必然触发）。

### 修复

- **`deploy.sh` 不再给 aiortc 打任何补丁**，改为跑一次
  [`scripts/webrtc_selftest.py`](../../wo-bot-control/scripts/webrtc_selftest.py)
  （DTLS 回环冒烟测试）。这一层一旦被破坏，部署阶段立刻可见。
- 已删除误导性的 `scripts/patch_aiortc.py`。

### 真机上 aiortc 已被改坏时怎么恢复

venv 内容不归 `deploy.sh` 管，需要手动还原。pip 的 wheel 缓存里有原始文件：

```bash
W=$(ls ~/.cache/pip/wheels/*/*/*/*/aiortc-0.9.28-*.whl | head -1)
rm -rf /tmp/ao && mkdir -p /tmp/ao && cd /tmp/ao && unzip -o -q "$W" 'aiortc/rtcdtlstransport.py'
cp /tmp/ao/aiortc/rtcdtlstransport.py \
   /opt/wobot/venv/lib/python3.7/site-packages/aiortc/rtcdtlstransport.py
/opt/wobot/venv/bin/python /opt/wobot/scripts/webrtc_selftest.py   # 应输出 [通过]
sudo systemctl restart wobot-control
```

判断是否被改坏：

```bash
grep -n 'set_read_ahead' /opt/wobot/venv/lib/python3.7/site-packages/aiortc/rtcdtlstransport.py
# 正常应为:  _openssl_assert(lib.SSL_CTX_set_read_ahead(ctx, 1) == 0)
# 若出现 hasattr(...) 三元表达式 → 已被错误补丁污染
```

---

## 坑 4：`logs/` 里一个 739MB 的陈旧轮转文件把根分区吃到 91%

**现象**：根分区 55G 用了 48G，只剩 5.0G（91%）。

**根因**：`/opt/wobot/logs/` 共 758MB，其中

```
739M  7月 10 00:53  wobot.log.2     ← 三个月前的陈旧轮转文件，再没被清理
 10M  8月 22        wobot.log.1
9.5M  10月 7        wobot.log
```

`RotatingFileHandler` 的 `backupCount` 只管理它自己那几代，不会清理历史遗留的大文件。

**处理**：删除即可释放约 739MB（约 13% 磁盘）：

```bash
sudo rm -f /opt/wobot/logs/wobot.log.2
```

安装蓝牙（T023，需 `apt install bluez-alsa-utils`）之前建议先做这一步。
