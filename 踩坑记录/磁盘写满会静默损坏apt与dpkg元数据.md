# 磁盘写满会静默损坏 apt 与 dpkg 元数据

> 发现时间：2026-10-07
> 影响：apt 完全不可用、`dpkg` 安装中断、若干包的元数据文件被写成全 NUL
> 特点：**不报错、不崩溃，只是"文件的字节变成了 0"** —— 极易被误判成别的问题

## 现象

磁盘用到 **91%**（`/` 剩 4.3G），之后出现一串看似无关的故障：

1. **apt 索引损坏**：
   ```
   E: Encountered a section with no Package: header
   E: Problem with MergeList /var/lib/apt/lists/mirrors.tuna.tsinghua.edu.cn_..._Packages
   E: The package lists or status file could not be parsed or opened.
   ```
   → `apt-cache policy <包>` 返回空，`apt-get install` 无法进行

2. **dpkg 安装中途崩溃**：
   ```
   dpkg: unrecoverable fatal error, aborting:
      files list file for package 'libgflags-dev' is missing final newline
   E: Sub-process /usr/bin/dpkg returned an error code (2)
   ```
   → 包已经下载完（151MB），但在 `Reading database ... 90%` 处中断

3. **元数据文件内容变成 NUL**：`libgflags-dev.list` 是 956 字节，`cat -A` 显示
   **全是 `^@`（NUL）**，`wc -l` 为 0 —— 根本不是"缺一个换行"，而是**内容整块丢失**

## 根因

磁盘写满时，文件系统的分配已经完成、但数据块没能落盘，于是留下**长度正确、内容全 0**
的文件。这种损坏**没有报错**，只是静默地把文件内容替换成了 NUL。

涉及的文件（本次实测）：

| 文件 | 大小 | 影响 |
|---|---|---|
| `libgflags-dev.list` | 956B | dpkg 直接无法操作（中断安装） |
| `libprotobuf-lite10:arm64.triggers` | 74B | 该包触发器失效 |
| `liburiparser-dev.md5sums` | 569B | `dpkg -V` 无法校验 |
| `python-wxversion.postinst` | 166B | 重装该包时脚本无法执行 |
| `ros-melodic-rqt-bag.md5sums` | 5321B | 同上（ROS 包，无关） |

## 排查方法（可复用）

```bash
# 1) 找出所有"内容全为 NUL"的文件（长度 > 0 但去掉 \0 后为 0）
sudo sh -c 'for f in /var/lib/dpkg/info/*; do
  [ -s "$f" ] || continue
  [ "$(tr -d "\0" < "$f" | wc -c)" -eq 0 ] && echo "全NUL: $(basename $f)"
done'

# 2) 检查 apt 索引健康
sudo apt-get check

# 3) 检查 dpkg 状态机
sudo dpkg --audit
```

> 注意：**不要只看"文件是否缺少末尾换行"**。`libgflags-dev.list` 报的是
> "missing final newline"，但真实病因是整个文件被 NUL 填满；只补一个换行
> 会让 dpkg 的表面检查通过，而内容仍然是坏的。

## 修复

```bash
# 1) 清掉损坏的 apt 索引并重建（同时释放空间，本次释放 231MB）
sudo rm -rf /var/lib/apt/lists/*
sudo apt-get update

# 2) 重装受影响包，让 dpkg 重建元数据
sudo apt-get install --reinstall -y libgflags-dev liburiparser-dev \
  libprotobuf-lite10 python-wxversion
```

修复后 `dpkg --audit` 无输出、`apt-get check` 通过，即可继续装包。
本次剩余 2 个全 NUL 文件（`python-wxversion.postinst`、`libprotobuf-lite10.triggers`）
与本项目无关，巡检只报 WARN。

## 预防

- **保持磁盘 >15% 空闲**；`scripts/healthcheck.sh` 第 1 项在 ≥85% 时 WARN、≥92% 时 FAIL
- 本项目已出现过两类"陈旧大文件"：`logs/wobot.log.2`（**739MB**）、
  `/var/lib/apt/lists`（**231MB**）—— 定期检查
- 磁盘吃紧时**先别装包**（apt 会在最坏的时刻损坏）

## 相关

- [L4T半升级导致CSI绿屏与Argus崩溃.md](L4T半升级导致CSI绿屏与Argus崩溃.md)
  —— 同类叠加伤害，排查时一起出现
