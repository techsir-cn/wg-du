# wg-du

WireGuard 动态更新（Dynamic Update）

![Shell Script](https://img.shields.io/badge/shell-script-green?logo=gnu-bash)
![License](https://img.shields.io/github/license/techsir-cn/wg-du)

**WireGuard 动态更新**  
**说明：自动检测公网 IP 变动，动态更新 WireGuard Peer Endpoint，无需重启接口**  
**版本：v2.1**  
**最后更新：2026-06-16**

---

## 1. 目标

wireguard-tools 的 `wg-quick` 仅在接口启动时解析 Endpoint 一次。当对端使用 DDNS 域名且 IP 变更后，连接中断，必须重启接口。v2.0 彻底解决了这个问题。

v2.0 新增 **server 角色**——不仅能监控对端域名 IP（client），还能检测**本机公网 IP 变化**并在变化后通知所有 server 类 Peer 重连，形成一个完整的双向 IP 变动闭环。

---

## 2. 核心特性

| 特性 | 说明 |
|------|------|
| **双角色支持** | 带 Endpoint 的 Peer → **client**（监控域名 IP）/ 无 Endpoint 的 Peer → **server**（监控本机 IP） |
| **自动配置解析** | 自动从 `wg0.conf` 或 OpenWrt `/etc/config/network` 解析 Peer，通过 `#name` 标记识别 |
| **交互式配置向导** | `wg-du --setup` 一键交互配置，自动检测、自动写入 |
| **配置检查** | `wg-du --check` 检查配置错误与遗漏 |
| **权威 DNS 直连** | 可选权威 DNS，规避 TTL 缓存延迟 |
| **IP 版本严格匹配** | IPv4 → A 记录；IPv6 → AAAA 记录 |
| **公网 IP 检测** | 多 URL 轮询获取本机公网 IPv4/IPv6 |
| **零中断更新** | `wg set` 动态更新，无需重启接口 |
| **端口 roaming 防护** | 自动检测并修正 NAT 导致的端口漂移 |
| **日志压缩** | 连续相同状态的日志仅保留首尾两条，不重复追加 |

---

## 3. 角色说明

```
有一个 WireGuard 网络：

┌──────────────┐         ┌──────────────┐
│  服务器节点   │ ◄─────► │  客户端节点   │
│  有公网 IP    │         │  动态 IP     │
│  (server 角色)│         │  (client 角色)│
└──────────────┘         └──────────────┘

client：对端有域名 → 监控域名 IP 变化，更新本地 Endpoint
server：对端无域名 → 监控本机公网 IP 变化，触发对端重建连接
```

### 判断规则

| 条件 | 角色 |
|------|------|
| `[Peer]` 中有 `Endpoint = domain:port` | **client** |
| `[Peer]` 中没有 Endpoint | **server** |

配置文件中无需手动指定角色，脚本根据是否有 Endpoint 自动判断。

---

## 4. 配置方法（二选一）

### 方式 A：交互式配置（推荐）

```sh
sudo wg-du --setup
```

向导会自动完成：
1. 检测 `wg0.conf` 或 OpenWrt `/etc/config/network`
2. 列出所有配置了 `#name` 的 Peer
3. 对每个 client Peer 询问 DNS 服务器和 Ping IP
4. 将配置写入脚本头部
5. 可选配置 cron 定时任务

### 方式 B：手动配置

编辑脚本头部配置区：

```sh
# === wg-du 配置区 ===
INTERFACE="wg0"
LOG_FILE="/etc/wireguard/wg-du.log"
CHECK_IP_URLS_V4="https://ddns.oray.com/checkip http://v4.66666.host:66/ip https://4.ipw.cn https://ip.3322.net"
CHECK_IP_URLS_V6="http://v6.66666.host:66/ip https://myip.ipip.net"

PEER_LD_ROLE="server"
PEER_MT_ROLE="client"
PEER_MT_DNS_SERVER="dns17.hichina.com"
PEER_MT_PING_IP="10.10.10.3"
# === wg-du 配置区结束 ===
```

然后在 `wg0.conf` 中为每个 Peer 添加 `#name` 标记：

```ini
[Peer]
#name = MT
PublicKey = xxxx
Endpoint = your.domain.com:51820
AllowedIPs = 10.10.10.3/32
```

---

## 5. 处理逻辑

### 5.1 Client 模式

1. 从 `wg show` 获取当前 Endpoint，**优先检测端口是否被 roaming 漂移**——若端口与预期不一致，先用 `wg set` 修正端口再继续
2. 判定 IP 版本，用 `dig` 查询对应 A/AAAA 记录
3. 若 IP 不同，执行 `wg set` 更新 Endpoint
4. 如果配置了 PING_IP，更新后 Ping 触发握手

### 5.2 Server 模式

1. 通过多 URL 轮询获取本机公网 IP
2. 与 `/tmp/wg-du-ip.${INTERFACE}` 中记录的上次 IP 比较
3. 若不同，执行 `wg set ... endpoint 0.0.0.0:0` 使对端重建连接

### 5.3 更新条件

仅当新 IP 与当前 IP **字符串不相等**且非空时执行更新。

---

## 6. 日志规范

```
YYYY-MM-DD_HH:MM:SS|PeerName|domain|ago=IP:PORT|aft=IP:PORT|same/diff
```

- `|diff` 表示执行了更新（IP 变化或端口漂移修正）
- `|same` 表示无变化——连续相同状态的日志不重复追加，仅更新末条时间戳（同一 peer 同一状态最多保留首条和末条两行）

---

## 7. 命令说明

| 命令 | 说明 |
|------|------|
| `wg-du` | 执行 DDNS 更新（用于 cron 定时任务） |
| `wg-du --setup` | 交互式配置向导 |
| `wg-du --check` | 检查配置错误 |
| `wg-du -h` | 显示帮助 |
| `wg-du -p` | 查看日志 |

---

## 8. 兼容性要求

| 环境 | 要求 |
|------|------|
| **Shell** | POSIX `/bin/sh`（兼容 OpenWrt BusyBox 和 Linux bash） |
| **依赖工具** | `wg`, `dig`, `curl`, `ping`, `awk`, `cut`, `head` |
| **配置文件** | `wg0.conf`（标准 WireGuard 格式）或 OpenWrt `/etc/config/network` |

---

## 9. 部署方法

### 步骤 1：下载脚本

```sh
wget https://github.com/techsir-cn/wg-du/releases/download/v2.1/wg-du -O wg-du && chmod +x wg-du
```

加速下载：

```sh
wget https://ghfast.top/https://github.com/techsir-cn/wg-du/releases/download/v2.1/wg-du -O wg-du && chmod +x wg-du
```

### 步骤 2：在 wg0.conf 中添加 #name

为每个要监控的 `[Peer]` 段落添加 `#name = 名称`：

```ini
[Peer]
#name = MyServer
PublicKey = xxxx
Endpoint = example.com:51820
AllowedIPs = 10.0.0.2/32
```

### 步骤 3：运行配置向导

```sh
sudo ./wg-du --setup
```

按提示完成配置。

### 步骤 4：安装 dig（如需要）

```sh
# Debian/Ubuntu
sudo apt install dnsutils

# OpenWrt
opkg update && opkg install bind-dig
```

### 步骤 5：查看日志与状态

```sh
wg-du -p        # 查看日志
cat /etc/wireguard/wg-du.log
```

---

## 10. 版本记录

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| v2.1 | 2026-06-16 | 新增端口 roaming 防护（`run_client` 自动检测端口漂移并修正）；日志压缩（`log_same`：连续相同状态仅更新末条时间戳，不追加）；`-p` 简化为直接查看日志；删除 `print_log` 函数和 `-p -a` 参数 |
| v2.0 | 2026-06-13 | 重构为双角色架构（client/server）；自动从 wg0.conf 解析 Peer（#name 标记）；新增 `--setup` 交互式配置向导；新增 `--check` 配置检查；新增 server 模式公网 IP 检测；新增 OpenWrt `/etc/config/network` 支持；依赖新增 curl/ping |
| v1.2 | 2026-02-27 | 日志格式优化：去除空格和标识，新增端口记录，ago/aft 字段 |
| v1.1 | 2026-02-19 | 项目更名为 wg-du；日志标识简化为 same/diff；支持命令行选项 |
| v1.0 | 2026-02-12 | 初始定稿 |

---

## License

MIT
