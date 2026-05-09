# Netplan 配置探索笔记：SSH 断连排查与三病灶诊断模型

在日常运维工作中，SSH 会话突然中断是一个令人困扰的问题。排查过程往往涉及物理层、系统层、应用层三个维度的交叉诊断。一次针对 Ubuntu 虚拟机 SSH 周期性断连的排查，意外揭示了 Netplan 配置中普遍被忽略的关键参数，进而归纳出一套可复用的排查模型。

**Netplan** 是 Ubuntu 17.10 引入的声明式网络配置工具，通过 YAML 配置文件描述网络拓扑，再由后端渲染器（`systemd-networkd` 或 `NetworkManager`）转换为底层网络配置。其配置正确性直接影响网络连接的稳定性。

Netplan 配置排查的关键发现：
- **`dhcp6: false` 缺失是 SSH 断连的首要推手**：链路波动时系统反复尝试获取 IPv6 地址，拖垮网络重建。
- **`metric` 缺失导致多网口随机选路**：内核无法区分主备路由，故障切换失败。
- **三病灶排查模型**：将网络故障归因为物理病灶、配置病灶、逻辑病灶三类，覆盖绝大多数场景。

本文将在重现故障现象的基础上，逐一解析五个关键参数的作用，并通过三病灶模型提供系统化的诊断思路。

## 目录

- [1. 引言：一次 SSH 断连引发的 Netplan 深挖](#1-引言一次-ssh-断连引发的-netplan-深挖)
- [2. 背景与问题复现](#2-背景与问题复现)
- [3. 核心概念：易被忽略的关键参数](#3-核心概念易被忽略的关键参数)
  - [3.1 `dhcp6: false` —— IPv6 静默干扰](#31-dhcp6-false--ipv6-静默干扰)
  - [3.2 `accept-ra: false` —— 路由公告污染](#32-accept-ra-false--路由公告污染)
  - [3.3 `optional: true` —— 启动等待问题](#33-optional-true--启动等待问题)
  - [3.4 `metric` —— 多网口路由权重](#34-metric--多网口路由权重)
  - [3.5 `renderer: networkd` —— 渲染器双头管理](#35-renderer-networkd--渲染器双头管理)
- [4. 常见场景与最佳实践](#4-常见场景与最佳实践)
  - [4.1 单网卡静态 IP 配置](#41-单网卡静态-ip-配置)
  - [4.2 双网卡主备热切换](#42-双网卡主备热切换)
  - [4.3 链路聚合 Bonding](#43-链路聚合-bonding)
- [5. 故障排除：三病灶排查模型](#5-故障排除三病灶排查模型)
  - [5.1 物理病灶](#51-物理病灶)
  - [5.2 配置病灶](#52-配置病灶)
  - [5.3 逻辑病灶](#53-逻辑病灶)
  - [5.4 排查诊断表](#54-排查诊断表)
- [6. SSH 连接稳定性补充优化](#6-ssh-连接稳定性补充优化)
- [7. Netplan 操作速查](#7-netplan-操作速查)
- [8. 总结](#8-总结)
- [9. 参考资料](#9-参考资料)

---

## 1. 引言：一次 SSH 断连引发的 Netplan 深挖

在 VMware 虚拟化环境下运行 Ubuntu 22.04 时，观察到以下现象：使用 Xshell 通过 SSH 连接虚拟机，每隔一两天，会话在敲命令途中突然中断，约一分钟后才能从另一个 session 重连。初步排查排除了 IP 冲突、NAT 服务崩溃、网卡驱动问题，最终在 `systemd-networkd` 日志中找到了关键线索。

---

## 2. 背景与问题复现

**测试环境：**

| 组件 | 配置 |
|---|---|
| 宿主机 | Windows 11（笔记本），有线 / WiFi 双网络 |
| 虚拟化平台 | VMware Workstation |
| 虚拟机 | Ubuntu 22.04，NAT 模式，静态 IP `10.0.0.101/24` |
| 连接方式 | Xshell SSH 连接到虚拟机 |

**故障现象：**

- 每 1-2 天出现一次 SSH 突然断开，断开时敲击键盘无响应。
- 约 60 秒后可从另一个 SSH 终端重新连接。
- 宿主机网络正常，互联网访问未中断。

**关键排查命令：**

```bash
# 查看 networkd 最近 50 条日志
sudo journalctl -n 50 -u systemd-networkd
```

**关键日志输出：**

```
ens33: Lost carrier                 # 网卡检测到链路丢失
ens33: DHCPv6 lease lost            # IPv6 租约丢失
ens33: Gained carrier               # 网卡检测到链路恢复
ens33: Lost carrier                 # 再次丢失
ens33: DHCPv6 lease lost            # 再次丢失 IPv6 租约
ens33: Gained carrier               # 再次恢复
...
ens33: DHCPv6 error: No message of desired type  # IPv6 DHCP 请求持续失败
```

**根因分析：**

故障不在 IP 层，而是**链路层**在频繁抖动。根本原因是笔记本在有线网络与 WiFi 之间切换，触发了 VMware NAT 服务的重绑定机制，导致 VMnet8 虚拟交换机闪断。Ubuntu 内核记录 `Lost carrier` 后，`systemd-networkd` 在重建 IPv4 配置的同时，**因未禁用 IPv6 而并发发起了 DHCPv6 请求**。IPv6 请求在纯 IPv4 环境中持续失败，拖慢了整个网络恢复流程，最终导致 SSH Socket 被强制重置。

---

## 3. 核心概念：易被忽略的关键参数

### 3.1 `dhcp6: false` —— IPv6 静默干扰

**背景：** Ubuntu 20.04+ 的 `systemd-networkd` 默认在每块网卡上启用 IPv6 的 SLAAC（Stateless Address Autoconfiguration，无状态地址自动配置）和 DHCPv6 客户端。即使 Netplan 配置中仅声明了静态 IPv4、未写 `dhcp6: true`，系统仍会在后台自动寻找 IPv6 路由器和 DHCPv6 服务器。

**触发条件与后果：**

- 链路抖动（`Lost carrier` 事件）触发网卡重新初始化。
- 系统在重建静态 IPv4 配置的同时，自动发起 IPv6 地址获取流程。
- 在纯 IPv4 环境中，DHCPv6 请求陷入无限重试循环。

**生产环境日志实例：**

```
Mar 19 16:16:16 eno1: Lost carrier
Mar 19 16:16:16 eno1: DHCPv6 lease lost           # 链路波动时系统在伤口上撒盐
Mar 19 16:16:22 eno1: Gained carrier
Mar 19 16:16:48 eno1: Lost carrier
Mar 19 16:16:48 eno1: DHCPv6 lease lost
Mar 19 16:19:38 eno1: Gained carrier
Mar 19 16:19:39 eno1: DHCPv6 error: No message of desired type
Mar 19 16:21:38 eno1: DHCPv6 error: No message of desired type
```

**影响：** 在某些内核版本中，IPv6 配置冲突会触发整个网卡接口被重新初始化，导致该接口上所有现有 TCP 连接（包括 SSH）立即中断。

> **提示**：静态 IP 场景下，`dhcp4: false` 和 `dhcp6: false` 应作为标配组合，同时出现在每一块网卡的配置中。

### 3.2 `accept-ra: false` —— 路由公告污染

**RA**（Router Advertisement，路由公告）是 IPv6 网络中路由器定期向网段内所有设备广播的报文，用于通告网络前缀和默认网关信息。`systemd-networkd` 默认监听 RA 报文并自动配置 IPv6 地址。

在 IPv6 未规划或交换机配置不规范的机房环境中，RA 报文可能包含错误的路由信息。接受这些报文后：

- 路由表被注入无效条目，出向流量可能被导向错误路径。
- 默认路由被覆盖，服务器从外部无法访问。

```yaml
# 正确的封锁组合
eno1:
  dhcp6: false                     # 禁止 IPv6 DHCP 客户端
  accept-ra: false                 # 禁止接受路由公告
```

### 3.3 `optional: true` —— 启动等待问题

当 Netplan 配置中声明了某块网卡、但该网卡实际不存在时（网线拔掉、驱动未安装、虚拟机切换网卡类型），`systemd-networkd` 默认等待该网卡上线，超时约**两分钟**。

**表现：** 开机时卡在 `A start job is running for Wait for Network`，系统启动速度极慢。

```yaml
# 对不保证始终在线的网卡添加此参数
eno4np1:
  optional: true                   # 找不到此网卡时直接跳过，不阻塞启动
```

> **提示**：主网口（保证始终在线的网卡）不需要添加 `optional: true`，避免掩盖真实的网络故障。仅对备用口、管理口等非永久连接的网卡使用。

### 3.4 `metric` —— 多网口路由权重

`metric` 是路由权重值，数字越小优先级越高。当系统存在多条指向同一目标的路由时，内核按 `metric` 值从小到大选择最优路径。

**真实案例：** 生产环境两台服务器各配了两块光口，指向同一网关，IP 相同、路由相同、没有 `metric`。结果是：

1. **IP 冲突：** 两块网卡同时持有同一个 IP，两根光纤都插入交换机时，交换机的 ARP 表发生"MAC 漂移"——发往该 IP 的数据包一会儿走 A 口、一会儿走 B 口，SSH 连接极度不稳定。
2. **路由失效：** 内核认为两条路由优先级相同，使用哈希算法随机选择发包路径。一根光纤断开后，内核仍可能将数据包发往已失效的网口，出现"时通时不通"。

**正确配置：**

```yaml
eno3np0:
  routes:
    - to: default
      via: 172.67.0.1
      metric: 100                  # 主路由（数值小，优先级高）
eno4np1:
  routes:
    - to: default
      via: 172.67.0.1
      metric: 200                  # 备用路由（数值大，优先级低）
```

**效果：** 主口在线走主口 → 主口断开自动切备口 → 主口恢复自动切回。拔线即切，全程无需人工干预。

### 3.5 `renderer: networkd` —— 渲染器双头管理

Netplan 支持两种后端渲染器：

| 渲染器 | 适用场景 | 说明 |
|---|---|---|
| `systemd-networkd` | **服务器（推荐）** | 轻量、无 GUI 依赖，适合 headless 环境 |
| `NetworkManager` | 桌面版工作站 | 适合有 GUI 的工作站，支持 WiFi、VPN 等动态网络 |

Ubuntu Server 22.04 默认使用 `networkd`，但安装 `network-manager` 等软件包后，`NetworkManager` 也会启动并尝试接管网卡。不声明 `renderer` 时，两个服务可能同时管理同一块网卡——即"双头管理"，导致配置周期性被覆盖，出现"配置明明是对的，但过一段时间就失效了"的现象。

```yaml
# 在配置顶层声明渲染器
network:
  version: 2
  renderer: networkd               # 明确锁定为 systemd-networkd
```

---

## 4. 常见场景与最佳实践

### 4.1 单网卡静态 IP 配置

```yaml
# 单网卡静态 IP — 虚拟机 / 单网口服务器通用模板
network:
  version: 2
  renderer: networkd               # 锁定渲染器
  ethernets:
    ens33:
      dhcp4: false                 # 禁用 IPv4 DHCP
      dhcp6: false                 # 禁用 IPv6 DHCP（关键）
      accept-ra: false             # 禁用路由公告（关键）
      addresses:
        - 10.0.0.101/24
      routes:
        - to: default
          via: 10.0.0.254
      nameservers:
        addresses:
          - 223.5.5.5
          - 119.29.29.29
```

### 4.2 双网卡主备热切换

适用于服务器有多余光口的场景。运维人员可直接将网线从故障口拔出插入备用口，无需进入终端修改配置。

```yaml
# 双网卡主备热切换模板
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      optional: true               # 找不到该网卡时跳过启动
      dhcp4: false
      dhcp6: false
      accept-ra: false
      addresses:
        - 10.0.0.101/24
      routes:
        - to: default
          via: 10.0.0.254
          metric: 100              # 主路由（优先级高）
      nameservers:
        addresses:
          - 223.5.5.5
          - 119.29.29.29
    ens160:
      optional: true               # 找不到该网卡时跳过启动
      dhcp4: false
      dhcp6: false
      accept-ra: false
      addresses:
        - 10.0.0.101/24
      routes:
        - to: default
          via: 10.0.0.254
          metric: 200              # 备用路由（优先级低）
      nameservers:
        addresses:
          - 223.5.5.5
          - 119.29.29.29
```

> **提示**：两块网卡不能同时物理接入同一个二层交换机（同一 IP 会引起 ARP/MAC 冲突），主备网卡同一时刻只接一根网线。应用后用 `ip route` 验证路由表，确认 metric 值正确生效。

### 4.3 链路聚合 Bonding

如果希望两根光纤**同时在线**，断任意一根时 SSH **完全无感知**，应使用 Bonding 将两块物理网卡合并为一块逻辑网卡。

```yaml
# 链路聚合 Bonding 模板
network:
  version: 2
  renderer: networkd
  ethernets:
    eno3np0:
      dhcp4: false
      dhcp6: false
      accept-ra: false
    eno4np1:
      dhcp4: false
      dhcp6: false
      accept-ra: false
  bonds:
    bond0:
      interfaces: [eno3np0, eno4np1]       # 两块物理网卡合并为 bond0
      parameters:
        mode: active-backup                 # 主备模式（也可用 802.3ad 做 LACP）
        fail-over-mac: active               # MAC 跟随活动接口，减少 ARP 刷新
      dhcp4: false
      dhcp6: false
      accept-ra: false
      addresses:
        - 172.67.0.43/24
      routes:
        - to: default
          via: 172.67.0.1
      nameservers:
        addresses:
          - 223.5.5.5
          - 8.8.8.8
```

**优势：** 两个物理网卡共用一套逻辑 MAC 和 IP，对上层应用完全透明，断一根光纤不产生任何 TCP 连接中断。`active-backup` 模式下，一块网卡断开后另一块立即接管。

---

## 5. 故障排除：三病灶排查模型

通过对此次排查过程的归纳，将 Netplan 相关的网络故障归因为三类"病灶"，每类病灶对应特定的日志特征、根本原因和解决方案。

### 5.1 物理病灶

**症状日志：**

```
ens33: Lost carrier
ens33: Gained carrier
```

**根本原因：** 网线接触不良、WiFi/有线切换、交换机端口重启、网卡驱动兼容性问题（如 VMware 默认的 e1000 驱动在 Ubuntu 22.04 下偶发不稳定）。

**解决方向：**

```bash
# VMware 虚拟机中更换为 vmxnet3 驱动（需安装 open-vm-tools）
sudo apt install open-vm-tools -y

# 物理机中检查网线连接和交换机端口状态
sudo ethtool eno1                   # 查看链路协商速率和双工模式
dmesg | grep -i eno1                # 查看内核级别的网卡事件
```

### 5.2 配置病灶

**症状日志：**

```
eno1: DHCPv6 lease lost
eno1: DHCPv6 error: No message of desired type
```

**根本原因：** 静态 IP 配置中未禁用 IPv6（缺 `dhcp6: false` 和 `accept-ra: false`），链路波动时系统在纯 IPv4 环境中反复寻找 IPv6 地址，拖慢网络恢复流程。

**解决方向：**

```yaml
# 为每块网卡添加以下两行
eno1:
  dhcp6: false                     # 禁用 IPv6 DHCP
  accept-ra: false                 # 禁用路由公告
```

### 5.3 逻辑病灶

**症状：** 多网口服务器通信时通时不通，ping 出现周期性丢包，无对应的特定错误日志。

**根本原因：** 多网口配置了相同 IP 但未设置 `metric`，内核在多条等价路由之间随机选路；某一网口断开后路由失效未被感知，系统继续向死口发包。

**解决方向：**

```yaml
# 为每条路由设置不同的 metric 值
eno3np0:
  routes:
    - to: default
      via: 172.67.0.1
      metric: 100                  # 主路由
eno4np1:
  routes:
    - to: default
      via: 172.67.0.1
      metric: 200                  # 备用路由
```

同时对非永久在线的网卡添加 `optional: true`，防止启动阻塞。

### 5.4 排查诊断表

| 病灶类型 | 日志特征 | 根本原因 | 解决参数 / 方法 |
|---|---|---|---|
| **物理病灶** | `Lost carrier` / `Gained carrier` 频繁交替 | 网线/WiFi 切换、交换机端口抖动、驱动 Bug | 更换稳定驱动（vmxnet3）；检查物理连接 |
| **配置病灶** | `DHCPv6 lease lost` / `DHCPv6 error` | 未禁用 IPv6，链路波动时系统反复尝试 IPv6 获取 | `dhcp6: false` + `accept-ra: false` |
| **逻辑病灶** | SSH 时通时不通；无特定错误日志 | 多网卡未设 `metric`，内核随机选路 | `metric` 区分主备；`optional: true` 防卡启动 |

> **核心原则：** 静态 IP 场景下，`dhcp4: false` / `dhcp6: false` / `accept-ra: false` 为三件套，缺一不可。多网卡场景再加上 `metric` 和 `optional: true`，五个参数构成完整防护。

---

## 6. SSH 连接稳定性补充优化

即使 Netplan 配置完美，移动办公环境（笔记本在多个网络之间切换）仍可能导致 SSH 断连。建议在服务端配置 SSH 心跳检测，减少无数据交互时的误断。

编辑 `/etc/ssh/sshd_config`：

```bash
# 开启 TCP KeepAlive
TCPKeepAlive yes

# 每 30 秒向客户端发送一次心跳包
ClientAliveInterval 30

# 最多容忍 3 次无响应后才断开连接
ClientAliveCountMax 3
```

```bash
# 重启 sshd 使配置生效
sudo systemctl restart ssh
```

**终极方案 —— tmux：** 在 Ubuntu 中安装 `tmux`（Terminal Multiplexer，终端复用器），每次 SSH 连接后先进入 `tmux` 会话。

```bash
# 安装 tmux
sudo apt install tmux -y

# 新建会话并命名
tmux new -s work

# 断连后重连，恢复原会话
tmux attach -t work
```

即使 Xshell 断开，命令仍在 tmux 会话中后台执行，重连后立即找回现场，彻底消除断连焦虑。

---

## 7. Netplan 操作速查

```bash
# 检查配置文件语法（不实际应用）
sudo netplan generate              # 无输出即为通过

# 试运行（120 秒超时自动回滚，适合远程操作防断连）
sudo netplan try                   # 按 Enter 确认生效，否则超时自动回滚

# 正式应用配置
sudo netplan apply                 # 立即生效

# 查看当前路由表（验证 metric 是否生效）
ip route

# 查看网卡状态
ip link

# 查看网卡详细信息（IP、MAC 等）
ip addr show

# 实时监控 networkd 日志（排查断连）
sudo journalctl -f -u systemd-networkd

# 查看最近 50 条 networkd 日志
sudo journalctl -n 50 -u systemd-networkd

# 验证 Bond 状态
cat /proc/net/bonding/bond0
```

---

## 8. 总结

Netplan 配置的稳定性隐患往往隐藏在"不写也能通"的参数中。`dhcp6: false` 和 `accept-ra: false` 封锁 IPv6 干扰，`optional: true` 防止启动阻塞，`metric` 明确多网口路由优先级，`renderer: networkd` 锁定渲染器。三病灶排查模型（物理病灶 → 配置病灶 → 逻辑病灶）提供了一套系统化的诊断思路：看日志 → 对号入座 → 修复参数。

掌握 Netplan 配置探索的思路后，你可以：

- 通过 `systemd-networkd` 日志快速定位网络断连的病灶类型。
- 在不支持 IPv6 的环境中彻底消除 DHCPv6 重试导致的连接中断。
- 在多网口服务器上实现拔线即切的热备切换或零中断的链路聚合。
- 结合 SSH 心跳和 `tmux` 会话，在移动办公环境中保持远程会话的连续性。

建议从单网卡静态 IP 的"三件套"起步，逐步实践多网口 metric 配置和 Bonding 部署，遇到故障时使用三病灶模型进行分步排查。

---

## 9. 参考资料

- [Netplan 官方文档](https://netplan.readthedocs.io/)
- [Netplan YAML 配置参考](https://netplan.readthedocs.io/en/stable/netplan-yaml/)
- [systemd-networkd 手册](https://www.freedesktop.org/software/systemd/man/systemd-networkd.html)
- [Linux Bonding 驱动文档](https://www.kernel.org/doc/Documentation/networking/bonding.txt)
- [Ubuntu Server 网络配置指南](https://ubuntu.com/server/docs/network-configuration)
