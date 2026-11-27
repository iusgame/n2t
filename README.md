# ugN2T N2T 引擎 · 完整参数与上手说明

> **这是一个用爱发电的闭源引擎工具。** 作者 tudouni，联系 QQ：601083006。  
> **非营利用途免费下载、运行，并允许在自己的程序里调用、封装、套壳**；盈利 / 商用请先联系我协商分成。  
> 不提供源码，不是开源，也不是 source-available。本目录就是 n2t 仓库的全部内容。

---

## 1. 这是什么

`n2t` 是自研 **N2T1 + MESH1** 双引擎的命令行程序；同一进程自动编排两个引擎，**没有引擎选择参数**。

- `n2t edge [...]`：终端节点，创建虚拟网卡并加入二层虚拟局域网。
- `n2t supernode [...]`：注册 / 中继节点，负责发现与直连失败后的 UDP 转发。
- `n2tctl`：**本地认证 IPC 控制客户端**，只查询/控制已经运行的 `n2t`，本身不开网卡、不跑 UDP/P2P，没有 `n2t` 在跑就什么都做不了。
- **不解析、不翻译、不互通 n2n / EasyTier 线协议**。`-c/-l/-a/-m/-k`、`N2N_KEY`、`N2N_COMPAT`、`internal/n2n`、`n2n.uglife.cn` 只是 CLI / 内部包名 / 部署 DNS 兼容术语。
- macOS / iOS / darwin **不支持**：无构建目标、无原生 TUN、无 IPC 实现。Android 是 VpnService 引擎目标，不提供 APK。

端口约定：N2T1 用 `-p` 指定端口（常见 `27777`），MESH1 中继固定用 `+1000`（`28777`，接近上限时反向偏移），不按社区派生新端口。`-l` 省略端口时用 `7654`。**没有 TCP transport，不要放行 TCP 27777**；旧 UDP `5644`/`5645` 管理口已不存在，控制走本地认证 IPC。

## 2. 文件清单

| 路径 | 说明 |
|---|---|
| `windows-x64/n2t.exe` | Windows 10/11 x64（或 Server 2016+）双引擎 CLI |
| `windows-x64/n2tctl.exe` | Windows x64 本地 IPC 控制工具 |
| `linux-x64/n2t` | Linux x86_64 静态 CLI（CentOS / SteamOS / Ubuntu / Deepin） |
| `linux-x64/n2tctl` | Linux x64 IPC 控制工具 |
| `README.md` | 本说明（完整参数与教程） |
| `LICENSE.txt` | 授权约定，含“允许套壳 / 盈利需协商” |
| `SHA256SUMS` | 本目录全部文件哈希 |

## 3. 快速开始

### Windows：自建房间

需要一个已安装的 TAP-Windows 适配器（`edge` 必须管理员权限；`supernode` 不需要 TAP 或管理员）。本仓库不附带驱动。

```powershell
# 中继 / 注册节点
.\n2t.exe supernode -p 27777 --mesh-network mynet -v

# 两个终端节点（另开两个管理员 PowerShell）
$env:N2N_KEY = '换成很长的随机口令'
.\n2t.exe edge -c mynet -a 10.9.0.2/24 -m 02:09:00:00:00:02 -l 127.0.0.1:27777 -v
$env:N2N_KEY = '换成很长的随机口令'
.\n2t.exe edge -c mynet -a 10.9.0.3/24 -m 02:09:00:00:00:03 -l 127.0.0.1:27777 -v
```

在节点 A 上 `ping 10.9.0.3`，节点 B 上 `ping 10.9.0.2`。

### Linux

Linux `edge` 需要 root 或 `CAP_NET_ADMIN` 加 `/dev/net/tun`：

```bash
chmod +x n2t n2tctl
./n2t supernode -p 27777 --mesh-network mynet -v
N2N_KEY='换成很长的随机口令' ./n2t edge -c mynet -a 10.9.0.2/24 -m 02:09:00:00:00:02 -l 127.0.0.1:27777 -v
```

独立节点（非 GUI 环境）的部署口径是**在启动命令里显式传 `-J` / `-k`**，不再依赖 `EnvironmentFile`；注意口令会出现在 `/proc/<pid>/cmdline`、`systemctl status` 和 shell history，需自行限制进程列表与日志访问。环境变量兜底仍保留以兼容旧脚本。`systemd` unit 建议 `0600 root:root`。

## 4. 语法与解析规则

```text
n2t edge      [参数...]
n2t supernode [参数...]
```

- 第一个参数必须是 `edge` 或 `supernode`，也可用 `help` / `-h` / `--help` 打印简短帮助。
- 支持 `-c mynet`、`--community mynet`、`--community=mynet` 三种写法。
- 布尔参数支持 `-v`、`--relay`、`--relay=false`；`-v 3` 表示提高 3 级日志。
- 未知参数、已移除参数和模式不匹配参数**不会导致启动失败**：输出 `compat: ignoring unsupported compatibility flag ...` 后忽略，不写入配置，也不伪造成“已生效”。
- **不支持配置文件启动**，请逐项写命令行。
- `--help` 只给简短用法；本文件是完整参数文档。

### 默认值

| 项 | 默认值 |
|---|---|
| 模式 | 未指定时报错 |
| edge 虚拟 IP | `-a` 必填；裸 IPv4 `/24`，裸 IPv6 `/64`；`dhcp:` / `auto:` / `0.0.0.0` 回退 `10.128.0.1/24` |
| underlay 地址族 | `auto`：同时开 IPv4/IPv6 套接字，IPv6 直连 → IPv4 直连 → supernode 中继 |
| MAC | 未指定 `-m` 时由 IP 生成稳定本地管理 MAC `02:xx:xx:xx:xx:xx` |
| MTU / 上限 | `1400` |
| MTU 下限 | `576` |
| MTU 自动协商 | 开启 |
| 中继 | 开启 |
| 注册周期 | `20s` |
| 广播限速 | `1000`/秒，上限 `10000` |
| 防重放时间窗 | `60s` |
| supernode 端口 | 必须由 `-p` 或 `-l` 提供，`1..65535` |
| edge 本地 UDP 端口 | 随机 |
| `-l` 省略端口 | `7654` |
| 日志 | `info`；`Verbose > 0` 为 `debug`；默认只写 stderr，文件日志关闭 |
| 文件日志 | `--log-to-exe-dir` 默认关闭 |

## 5. 有效参数总表

`E` = 仅 `edge` 生效；`S` = 仅 `supernode` 生效；`E/S` = 两者均可。

| 参数 | 别名 | 模式 | 取值与默认 | 实际作用 |
|---|---|---|---|---|
| `-c <社区>` | `--community` | E | 1..64 字节，E 必填；可由 `N2N_COMMUNITY` 提供 | 社区隔离与密钥派生；S 模式下告警忽略 |
| `-k <口令>` | `--key` | E/S | 最长 256 字节；可由 `N2N_KEY` 提供 | E：社区密钥；S：联邦 HMAC 与 MESH1 探活密钥来源 |
| `-l <host:port>` | `--supernode`、`--supernode-list`、`--listen` | E/S | 可重复；省略端口 `7654` | E：supernode 地址列表；S：联邦节点地址列表 |
| `-p [<ip>:]<port>` | `--port`、`--local-port` | E/S | `1..65535`；IPv6 写 `[::]:27777` | S：监听地址/端口；E：本地 UDP 绑定；与 `-4/-6` 冲突时报错 |
| `-a <ip[/前缀]>` | `--ip`、`--addr` | E | CIDR 或裸 IP；IPv4 `1..32`，IPv6 `1..128`；支持 `static:`、`dhcp:`、`auto:`、`auto46` | 虚拟 IP 与前缀；与 underlay 地址族独立 |
| `-4` | `--ipv4-only` | E/S | 布尔 | 强制 UDP underlay 走 `udp4` |
| `-6` | `--ipv6-only` | E/S | 布尔 | 强制 UDP underlay 走 `udp6` |
| `--family auto\|ipv4\|ipv6` | — | E/S | `auto` | 只约束 underlay，不改变 `-a` 虚拟地址族 |
| `-m <mac>` | `--mac` | E | 6 字节 MAC，如 `02:09:00:00:00:02` | 固定虚拟 MAC |
| `-I <描述>` | `--description` | E | 截断到 63 字节 | 节点昵称 |
| `-d <名称>` | `--dev`、`--device`、`--tap-device` | E | 设备名/友好名 | Windows 按友好名找 TAP；Linux 作 TUN 名 |
| `--tap-guid <GUID>` | `N2T_TAP_GUID` | E | GUID | Windows 精确指定 TAP，优先于 `-d` |
| `--tun-fd <fd>` | `N2T_TUN_FD` | E | 正整数 | Android / 嵌入式注入已有 TUN FD |
| `-M <mtu>` | `--mtu` | E | `576..65535`，默认 `1400` | MTU 与 MTU 上限；S 模式告警忽略 |
| `--mtu-min <n>` | — | E | `576..65535` 且 `<= --mtu-max` | MTU 协商下限 |
| `--mtu-max <n>` | — | E | `576..65535` 且 `>= --mtu-min` | MTU 协商上限 |
| `--mtu-auto` | — | E | 布尔，默认开启 | 启用 MTU 探测/协商 |
| `--no-mtu-auto` | — | E | 布尔 | 关闭协商，锁定 MTU |
| `-D` | `--pmtu` | E | 布尔 | 等价于开启 MTU 自动协商 |
| `-r` | `--relay` | E | 布尔，默认开启 | 直连失败时允许 supernode 短时兜底转发 |
| `-S1` | — | E | 布尔 | 强制走中继，底层仍是 UDP |
| `-S2` | — | E | 布尔 | 同 `-S1`，不启用 TCP 中继 |
| `-i <秒>` | `--registration-interval` | E | 整数，`<=0` 回退 `20` | 注册/打洞周期 |
| `-J <口令>` | `--user-password`、`N2N_PASSWORD` | S | 字符串 | 联邦同步 HMAC 密钥的优先来源 |
| `-F <名称>` | `--federation` | S | 字符串 | 联邦名，与 `-l` 启用带 HMAC 的同步 |
| `--broadcast-limit <n>` | — | E | `1..10000`，默认 `1000` | 每秒广播限速 |
| `--replay-ttl <时长>` | — | E | `10s..10m`，默认 `60s` | 防重放窗口，如 `90s`、`2m` |
| `--mesh-network <名称>` | 支持 `<名称>:<密钥>` | S | 可重复 | 在固定 MESH1 端口登记社区；冒号内嵌密钥会进 argv，不推荐 |
| `--mesh-secret <口令>` | — | S | 字符串 | 与前/后一个 `--mesh-network` 配对的探活密钥 |
| `--mesh-relay-open-probes` | — | S | 布尔 | 允许未登记社区限流探活；中继不解密用户数据 |
| `-v` | `--verbose` | E/S | 布尔，可重复/带数字 | 提高日志级别到 debug |
| `--log-to-exe-dir` | — | E/S | 布尔，默认关闭 | 在可执行文件目录追加写 `n2t.log`（0600），仍写 stderr |
| `-V <值>` | `--version-string <值>` | E | 值忽略；每出现一次降低一级日志 | 兼容 n2n 的降详细程度；S 模式告警忽略 |
| `-h` | `--help`、`help` | E/S | 布尔 | 打印简短帮助后退出 |

`-c`、`-M`、`-V` 在 S 模式下都会告警忽略。`--mgmt-port` **不是** `-p` 的别名，已被移除。

### 环境变量（兼容兜底）

| 变量 | 模式 | 作用 |
|---|---|---|
| `N2N_KEY` | E/S | 社区密钥；S 下也是联邦 / MESH1 探活密钥来源，不进 argv |
| `N2N_COMMUNITY` | E | 社区名 |
| `N2N_PASSWORD` | S | 联邦 HMAC，等价 `-J`，优先于 `N2N_KEY` |
| `N2T_TAP_GUID` | E | Windows TAP GUID |
| `N2T_DEVICE` | E | TAP/TUN 设备名 |
| `N2T_TUN_FD` | E | Android/嵌入式 TUN FD |
| `N2T_REGION` | E | MESH1 选路区域提示，无 n2n 兼容含义 |

`--log-to-exe-dir` 没有环境变量形式，只能命令行开启。

### 已移除参数（只告警，不生效）

`-T`/`--tos`、`-L`/`--ttl`、`-A1`…`-A5`、`-H`/`--header-encryption`、`-z1`/`-z2`、`-e`/`--prefer-local-ip`、`--select-rtt`/`--select-mac`、`--no-port-forwarding`、`-E`/`--multicast`、`-P`/`--public-key`、`-R`/`--rule`、`-x`/`--metric`、`-f`/`--foreground`、`-t`/`--management-port`/`--mgmt-port`、`--management-password`、`-c-file`、`-a-range`/`--auto-ip-range`、`-u`/`--uid`、`-g`/`--gid`、`--log-file`、`--no-log`、`-n`、`--communities`；以及 S 模式下的 `-c`、`-M`、`-V`。旧脚本能启动，但这些参数不写入配置。需要文件日志用 `--log-to-exe-dir`。

## 6. 更多示例

### 联邦 supernode + 多社区 MESH1

```powershell
$env:N2N_KEY = '换成很长的随机口令'
.\n2t.exe supernode -p 27777 -F uglife-prod-2026 -J '联邦HMAC强口令' `
  --mesh-network room-a --mesh-network room-b --mesh-relay-open-probes -v
```

### IPv6 虚拟网段

虚拟 IP 由 `-a` 决定，与 underlay 独立。老游戏继续用 IPv4 虚拟网段即可。

```powershell
$env:N2N_KEY = '换成很长的随机口令'
.\n2t.exe edge -c v6room -a fd00:1::2/64 -m 02:60:00:00:00:02 -l 1.2.3.4:27777 -v
```

把 underlay 也限定为 IPv6 时，supernode 绑定和 `-l` 都要用 IPv6 字面量并加 `-6`：

```powershell
.\n2t.exe supernode -p '[::]:27777' -6
.\n2t.exe edge -c v6room -a fd00:1::2/64 -l '[::1]:27777' -6
```

### 老游戏强二层流量

不需要额外参数，真实 ARP、非 IP EtherType、VLAN 标签由 TAP/TUN 归一化处理；直连不可靠时 `-S1` 可强制中继。

## 7. `n2tctl` 与套壳（wrapper）接入

`n2t` 启动时会向 stdout 打印：

```text
N2T_IPC_ENDPOINT=127.0.0.1:61078
N2T_IPC_TOKEN_FILE=<程序目录>\cache\ipc\supernode-.token
```

IPC 走回环 TCP（Windows）或 Unix Domain Socket（Linux/Android），首帧带 Token 鉴权；每启动一次生成新 Token。GUI、管理页面、自动化脚本就是通过这条本地认证通道读取状态和控制节点，**不需要外部 UDP 管理口**。

`n2tctl` 用法：

```powershell
n2tctl [--endpoint ADDR] [--token-file PATH] <command> [json-payload]
```

等价环境变量：`N2T_IPC_ENDPOINT`、`N2T_IPC_TOKEN_FILE`。

| 命令 | 说明 | JSON 载荷示例 |
|---|---|---|
| `ping` | 连通性测试 | — |
| `status` | 节点状态 / 统计 / 邻居 | — |
| `peers` | 对端列表（别名 `peerlist`、`getpeerlist`） | — |
| `start-node` | 启动 edge | `{"community":"net","key":"secret","ip":"10.0.0.2/24","supernode":"127.0.0.1:7654"}` |
| `stop-node` | 停止 edge | — |
| `join` | 加入 MESH1 社区 | `{"community":"net","key":"secret"}` |
| `leave` | 离开社区 | — |
| `start-supernode` | 启动 supernode | `{"port":7654}` |
| `stop-supernode` | 停止 supernode | — |
| `events` | 持续输出事件直到中断 | — |

退出码：`0` 成功、`1` 鉴权/连接/命令失败、`2` 未知选项、未知命令、无效 JSON 或缺 endpoint。命令名不区分大小写。

**允许套壳**：你可以在自己的 GUI、控制台、启动器、服务里调用这些二进制或复用 IPC 协议来控制节点；非营利场景免费。商用 / 盈利分成请先联系 QQ：601083006 协商。

## 8. 常见错误

| 错误 | 原因与处理 |
|---|---|
| `unknown mode ... (expected edge\|supernode)` | 第一个参数必须是模式 |
| `supernode requires -l <port> or -p <port>` | 补 `-p 27777` |
| `edge requires -c <community>` / `-k <key>` / `-a <ip>` / `-l <supernode:port>` | 补齐必填参数 |
| `invalid MAC` | `-m` 必须是 6 字节 MAC |
| `invalid MTU range` | MTU 越界或下限大于上限 |
| `invalid replay TTL` | `--replay-ttl` 需在 `10s..10m` |
| `no usable TAP-Windows adapter found` | 安装 TAP 驱动或指定 `--tap-guid` |
| `open TAP adapter ...: Access is denied` | edge 未以管理员运行 |
| `integrity check failed: copyright seal mismatch` | 二进制版权烙印被改，请重新下载并核对哈希 |

## 9. 校验文件

`SHA256SUMS` 覆盖本目录下全部文件。Windows PowerShell：

```powershell
Get-Content .\SHA256SUMS | ForEach-Object {
  $hash, $path = $_ -split '  ', 2
  $actual = (Get-FileHash -LiteralPath $path -Algorithm SHA256).Hash.ToLowerInvariant()
  if ($actual -ne $hash) { throw "SHA256 不匹配: $path" }
}
'SHA256 OK'
```

Linux 终端：

```bash
sha256sum -c SHA256SUMS
```

版权串 `QQ：601083006` 烙印在 `--help`、启动横幅和 PE 元数据里；启动完整性自检失败退出码为 `3`。当前没有 Authenticode 签名，请以发布页 SHA-256 为准，不要关闭安全软件。

## 10. 许可与联系

非营利：免费下载、运行、套壳、原样分发（带上 `SHA256SUMS` 和 `LICENSE.txt`）。  
商用 / 盈利分成：先联系我协商，QQ：601083006。

软件按“现状”提供，无保修、无更新承诺。详细约定见 [`LICENSE.txt`](LICENSE.txt)。
