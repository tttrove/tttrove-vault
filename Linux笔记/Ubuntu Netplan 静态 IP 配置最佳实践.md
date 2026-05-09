# Ubuntu Netplan 静态 IP 配置：五个关键参数与完整模板

在 Linux 服务器装机流程中，配置静态 IP 是最基础的操作之一。Netplan 是 Ubuntu 的默认网络配置工具，大多数装机教程只讲到"配通"为止，却忽略了几个决定网络稳定性的关键参数——它们的缺失不会影响网络"能不能通"，但会在链路波动、网卡切换、系统重启等场景下引发 SSH 断连、路由漂移、开机卡死等问题。

**Netplan** 是一款声明式网络配置工具，通过 YAML 配置文件描述网络拓扑，再由后端渲染器（`systemd-networkd` 或 `NetworkManager`）转换为底层配置。自 Ubuntu 17.10 起成为默认网络配置系统。

Netplan 静态 IP 配置的核心要点：
- **`dhcp6: false` 与 `accept-ra: false`**：彻底封锁 IPv6 干扰，消除链路波动时的配置冲突。
- **`optional: true`**：防止找不到网卡时开机等待两分钟。
- **`metric`**：多网口场景下明确主备路由优先级，实现拔线即切。
- **`renderer: networkd`**：锁定渲染器，避免双头管理导致的配置周期性覆盖。

本文将从配置自查、五个关键参数的原理、三种场景的完整模板以及装后验证流程等方面，帮助读者编写一次到位的 Netplan 静态 IP 配置。

## 目录

- [1. 引言：你的 Netplan 配置"能通但不稳"](#1-引言你的-netplan-配置能通但不稳)
- [2. 前置准备](#2-前置准备)
  - [2.1 检查当前 Netplan 配置](#21-检查当前-netplan-配置)
  - [2.2 确认网卡名称与系统环境](#22-确认网卡名称与系统环境)
- [3. 基础用法](#3-基础用法)
  - [3.1 Netplan 配置文件结构](#31-netplan-配置文件结构)
  - [3.2 应用与验证配置](#32-应用与验证配置)
- [4. 核心概念：五个必须写的参数](#4-核心概念五个必须写的参数)
  - [4.1 `dhcp4: false` 与 `dhcp6: false`](#41-dhcp4-false-与-dhcp6-false)
  - [4.2 `accept-ra: false`](#42-accept-ra-false)
  - [4.3 `optional: true`](#43-optional-true)
  - [4.4 `metric`](#44-metric)
  - [4.5 `renderer: networkd`](#45-renderer-networkd)
- [5. 常见场景与最佳实践](#5-常见场景与最佳实践)
  - [5.1 常见使用场景](#51-常见使用场景)
  - [5.2 最佳实践](#52-最佳实践)
- [6. 故障排除](#6-故障排除)
- [7. 总结](#7-总结)
- [8. 参考资料](#8-参考资料)

---

## 1. 引言：你的 Netplan 配置"能通但不稳"

在服务器装机完成后，网络能通只代表配置文件没有语法错误。真正的隐患在于那些"不写也能通，但不写就会在特定条件下出问题"的参数。以下是一个典型的生产环境 Netplan 配置——由装机工具 Subiquity 自动生成，表面正常，实则暗藏四个问题。

### 1.1 典型问题配置

```yaml
# 由 Subiquity 自动生成的网络配置
network:
  ethernets:
    eno1:
      addresses: [10.106.159.6/27]
      routes:
        - to: default
          via: 10.106.159.1
      nameservers:
          addresses: [223.5.5.5, 8.8.8.8]
      dhcp4: false                 # 正确：已禁用 IPv4 DHCP
    eno2:
      dhcp4: true                  # 管理口，DHCP 获取 IP
    eno3np0:
      addresses: [172.67.0.43/24]
      routes:
        - to: 172.67.0.0/16
          via: 172.67.0.1
      dhcp4: false
    eno4np1:
      addresses: [172.67.0.43/24]
      routes:
        - to: 172.67.0.0/16
          via: 172.67.0.1
      dhcp4: false
  version: 2
```

### 1.2 配置中隐藏的四个隐患

| # | 缺失参数 | 位置 | 后果 |
|---|---|---|---|
| 1 | `dhcp6: false` | 所有静态 IP 网卡 | 链路波动时系统疯狂尝试获取 IPv6 地址，拖垮网络重建速度，严重时断 SSH |
| 2 | `accept-ra: false` | 所有静态 IP 网卡 | 交换机可能推送错误的 IPv6 路由，污染路由表 |
| 3 | `metric` | `eno3np0` / `eno4np1` | 两只网口配置完全相同，内核随机选路；断一个口后流量可能继续往死口发 |
| 4 | `renderer: networkd` | 顶层缺失 | `NetworkManager` 与 `systemd-networkd` 可能同时管理同一块网卡，配置周期性被覆盖 |

---

## 2. 前置准备

### 2.1 检查当前 Netplan 配置

```bash
# 列出所有 Netplan 配置文件
ls /etc/netplan/                   # 通常为 00-installer-config.yaml 或 01-netcfg.yaml

# 查看当前生效的网络接口
ip link show                       # 列出所有网络接口及其状态

# 查看当前路由表
ip route                           # 检查 default 路由和 metric 值
```

Netplan 配置文件存放在 `/etc/netplan/` 目录下，文件扩展名为 `.yaml`。常见文件名为 `00-installer-config.yaml`（Subiquity 装机工具生成）或用户自定义的 `01-netcfg.yaml`。

### 2.2 确认网卡名称与系统环境

```bash
# 列出物理网卡（排除 lo 回环口和虚拟接口）
ip -br link show | grep -vE 'lo|docker|veth|br-'

# 确认当前活动的网络管理服务
systemctl is-active systemd-networkd   # 预期输出：active
systemctl is-active NetworkManager     # 服务器场景预期输出：inactive
```

> **提示**：Ubuntu Server 22.04 默认使用 `systemd-networkd` 管理网络。如果 `NetworkManager` 处于 `active` 状态，务必在 Netplan 配置中声明 `renderer: networkd` 以排除双头管理冲突。

---

## 3. 基础用法

### 3.1 Netplan 配置文件结构

Netplan 使用 YAML 格式，所有缩进必须使用**空格**（不允许 Tab 字符）。基本结构如下：

```yaml
# Netplan 配置文件示例
network:
  version: 2                       # 固定值，当前配置格式版本为 2
  renderer: networkd               # 指定后端渲染器
  ethernets:                       # 物理网卡配置块
    eno1:                          # 网卡名称（用 ip link 查看）
      dhcp4: false                 # 禁用 IPv4 DHCP
      addresses:                   # 静态 IP 列表（CIDR 格式）
        - 10.0.0.101/24
      routes:                      # 路由规则列表
        - to: default              # 默认路由目标
          via: 10.0.0.254          # 网关地址
      nameservers:                 # DNS 配置
        addresses:
          - 223.5.5.5
```

### 3.2 应用与验证配置

```bash
# 步骤 1：检查配置文件语法（不实际应用）
sudo netplan generate              # 无输出即为通过，有报错直接打印

# 步骤 2：试运行（120 秒后自动回滚，适合远程操作防断连）
sudo netplan try                   # 按 Enter 确认生效，否则超时自动回滚

# 步骤 3：正式应用配置
sudo netplan apply                 # 立即生效，不自动回滚

# 步骤 4：验证路由表
ip route                           # 检查 default 路由的网关 IP 和 metric 值是否正确
```

---

## 4. 核心概念：五个必须写的参数

### 4.1 `dhcp4: false` 与 `dhcp6: false` —— 关闭不该开的自动配置

`dhcp4: false` 是静态 IP 的基本要求，但 **`dhcp6: false` 同样必须写**。原因在于 Ubuntu 20.04 之后，`systemd-networkd` 默认在每块网卡上启用 IPv6 自动地址配置（SLAAC，Stateless Address Autoconfiguration），**无论配置文件中是否写了 `dhcp6: true`，系统都会在后台寻找 IPv6 路由器**。

链路检测到抖动（`Lost carrier` 事件）时，systemd-networkd 会同时执行两项操作：重建 IPv4 配置，并发起 IPv6 地址获取。在纯 IPv4 环境中，IPv6 请求陷入无限重试循环，日志持续输出：

```
eno1: DHCPv6 lease lost
eno1: DHCPv6 error: No message of desired type
eno1: DHCPv6 error: No message of desired type
```

这些重试严重拖慢网卡重新就绪的速度。在某些内核版本中，IPv6 配置冲突会触发整个网卡接口重新初始化，断开该接口上所有现有 TCP 连接（包括 SSH）。

> **提示**：静态 IP 场景下，`dhcp4: false` 和 `dhcp6: false` 应同时出现在每一块网卡的配置中。

### 4.2 `accept-ra: false` —— 拒绝交换机推送的路由公告

**RA**（Router Advertisement，路由公告）是 IPv6 协议中的一项机制：路由器定期向网段内所有设备广播路由信息，`systemd-networkd` 默认监听并采纳这些公告。

在未规划 IPv6 的机房环境中，交换机可能因配置不规范而推送错误的 RA 报文。如果服务器接受这些报文，路由表被注入无效条目，严重时默认路由被覆盖，导致外部无法访问服务器。

```yaml
# 正确写法：dhcp6 和 accept-ra 配合使用
eno1:
  dhcp6: false                     # 禁止 IPv6 DHCP 客户端
  accept-ra: false                 # 禁止接受路由公告
```

### 4.3 `optional: true` —— 防止找不到网卡时开机卡死

当 Netplan 配置中声明了某块网卡、但该网卡实际不存在时（网线拔掉、驱动未加载、虚拟机切换网卡类型），`systemd-networkd` 会等待该网卡上线，默认超时约**两分钟**。表现是开机时卡在 `A start job is running for Wait for Network`。

```yaml
# 仅对不保证始终在线的网卡添加此参数
eno4np1:
  optional: true                   # 找不到此网卡时直接跳过，不阻塞启动
  dhcp4: false
  dhcp6: false
  accept-ra: false
```

> **提示**：主网口（保证始终在线的网卡）不需要加 `optional: true`，否则可能掩盖真实的网络故障。仅对备用口、管理口等非永久连接的网卡使用。

### 4.4 `metric` —— 明确主备路由的优先级

`metric` 是路由权重值，数字越小优先级越高。当系统存在多条指向同一目标的路由时，内核按 `metric` 值从小到大选择最优路径。

不设置 `metric` 时的双重隐患：

1. **两根网线同时在线**：流量随机走主口或备口，SSH 连接不稳定。
2. **一根网线断开**：内核不知道该路径已失效，继续向死口发包，出现"时通时不通"。

```yaml
# 正确写法：主备关系由 metric 明确区分
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

**效果：** 主口在线时流量走主口 → 主口断开自动切备口 → 主口恢复自动切回。拔线即切，全程无需人工干预。

### 4.5 `renderer: networkd` —— 锁定渲染器避免双头管理

Netplan 支持两种后端渲染器：

| 渲染器 | 适用场景 | 特点 |
|---|---|---|
| `systemd-networkd` | **服务器（推荐）** | 轻量，无 GUI 依赖，适合 headless 环境 |
| `NetworkManager` | 桌面版工作站 | 功能丰富，支持 WiFi、VPN 等动态网络 |

Ubuntu Server 默认使用 `networkd`，但安装 `network-manager` 等软件包后，`NetworkManager` 也会启动并尝试接管网卡。不声明 `renderer` 时，两个服务可能同时管理同一块网卡——即"双头管理"，导致配置周期性被覆盖。

```yaml
# 在配置顶层声明渲染器
network:
  version: 2
  renderer: networkd               # 明确锁定为 systemd-networkd
  ethernets:
    ...
```

---

## 5. 常见场景与最佳实践

### 5.1 常见使用场景

#### 场景 A：单网口服务器，静态 IP

最常见的场景：一台服务器、一块网卡、一个静态 IP。

```yaml
# 单网口静态 IP 完整配置
network:
  version: 2
  renderer: networkd               # 明确渲染器
  ethernets:
    eno1:                          # 替换为实际网卡名（用 ip link 查看）
      dhcp4: false                 # 禁用 IPv4 DHCP
      dhcp6: false                 # 禁用 IPv6 DHCP（必须写）
      accept-ra: false             # 禁用路由公告（必须写）
      addresses:
        - 10.106.159.6/27          # 替换为实际 IP 和子网掩码
      routes:
        - to: default              # 默认路由
          via: 10.106.159.1        # 替换为实际网关
      nameservers:
        addresses:
          - 223.5.5.5              # 阿里 DNS
          - 119.29.29.29           # 腾讯 DNS
```

#### 场景 B：多网口服务器，备用光口热切换

适用于服务器光口富余、用备口做冷备的场景。两口同一时刻只接一根网线，主口故障时把网线换插到备口即可恢复，无需进入终端修改配置。

```yaml
# 双网口主备热切换完整配置
network:
  version: 2
  renderer: networkd
  ethernets:
    eno3np0:
      optional: true               # 找不到此网卡时不阻塞启动
      dhcp4: false
      dhcp6: false
      accept-ra: false
      addresses:
        - 172.67.0.43/24
      routes:
        - to: default
          via: 172.67.0.1
          metric: 100              # 主路由（优先级高）
      nameservers:
        addresses:
          - 223.5.5.5
          - 119.29.29.29
    eno4np1:
      optional: true               # 找不到此网卡时不阻塞启动
      dhcp4: false
      dhcp6: false
      accept-ra: false
      addresses:
        - 172.67.0.43/24
      routes:
        - to: default
          via: 172.67.0.1
          metric: 200              # 备用路由（优先级低）
      nameservers:
        addresses:
          - 223.5.5.5
          - 119.29.29.29
```

> **提示**：两根网线不要同时接入同一个二层交换机。两块网卡持有同一个 IP，同时在线会引起 ARP/MAC 冲突。此方案的设计前提是：同一时刻只有一根网线插入。

#### 场景 C：多网口服务器，链路聚合 Bonding

如果需要两根光纤**同时在线**，且其中任意一根断开时 SSH **完全无感**，应使用链路聚合（Bonding）将两块物理网卡合并为一块逻辑网卡。

```yaml
# 链路聚合 Bonding 完整配置
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
      interfaces: [eno3np0, eno4np1]        # 两块物理网卡合并为 bond0
      parameters:
        mode: active-backup                  # 主备模式，一块断了另一块接管
        fail-over-mac: active                # MAC 地址跟随活动接口，减少 ARP 刷新
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
          - 119.29.29.29
```

Bond 模式下，对外只有一个 IP 和一个 MAC 地址，上层应用感知不到底层切换。`active-backup` 模式下，一块网卡断开时另一块立即接管，中间不产生任何 TCP 中断。

```bash
# 验证 Bond 状态
cat /proc/net/bonding/bond0         # 查看当前活动网卡和链路状态
```

输出示例：`Currently Active Slave: eno3np0`（当前活动网卡为 eno3np0）、`MII Status: up`（链路正常）。

### 5.2 最佳实践

#### 1. 静态 IP 五件套，每块网卡缺一不可

对于纯 IPv4 的静态 IP 服务器，每块网卡的配置必须包含以下参数：

```yaml
eno1:
  dhcp4: false                     # ① 禁用 IPv4 自动获取
  dhcp6: false                     # ② 禁用 IPv6 自动获取
  accept-ra: false                 # ③ 拒绝路由公告
  # addresses / routes ...         # ④ 静态 IP 地址和路由规则
  # optional: true                 # ⑤ 仅对备用网卡使用
```

对于备用网卡额外添加 `optional: true`；配置顶层添加 `renderer: networkd`。

#### 2. 装完必做的四步自查

配置文件写完后，按以下顺序检查再离开服务器：

```bash
# 第一步：语法检查
sudo netplan generate              # 无输出即为通过

# 第二步：路由表验证
ip route                           # 确认 default 路由的网关、metric 值正确

# 第三步：日志检查
sudo journalctl -n 30 -u systemd-networkd  # 不应出现 DHCPv6 相关错误

# 第四步：外部连通性验证
# 从另一台机器 SSH 进来并保持连接数分钟，确认无断连
```

#### 3. 主备网卡不同时接线

主备热切换（场景 B）的前提是两块网卡同一时刻只接一根网线。如需两根同时在线，使用 Bonding（场景 C）。

---

## 6. 故障排除

### 6.1 SSH 周期性断连，日志出现 `DHCPv6 lease lost`

**错误日志：**

```
ens33: Lost carrier
ens33: DHCPv6 lease lost
ens33: Gained carrier
ens33: DHCPv6 error: No message of desired type
```

**原因：** 未配置 `dhcp6: false`，链路波动时系统同步尝试获取 IPv6 地址，拖慢网络恢复速度；严重时触发网卡重新初始化，断开所有 TCP 连接。

**解决：**

```bash
# 1. 编辑 Netplan 配置，为所有网卡添加
#    dhcp6: false
#    accept-ra: false
sudo nano /etc/netplan/00-installer-config.yaml

# 2. 应用配置
sudo netplan apply

# 3. 验证修改生效
sudo journalctl -f -u systemd-networkd   # 持续监控日志，不应再出现 DHCPv6 错误
```

### 6.2 开机卡在 "A start job is running for Wait for Network"

**原因：** 配置文件中声明了某块网卡但该网卡不存在或未连接，且未设置 `optional: true`，systemd-networkd 等待超时（约两分钟）。

**解决：**

```yaml
# 对不保证始终在线的网卡添加 optional: true
eno4np1:
  optional: true                   # 找不到则跳过，不阻塞启动
```

### 6.3 多网口时通时不通

**错误现象：** 两台服务器之间通信时好时断，ping 出现周期性丢包。

**原因：** 多网口配置了相同 IP 且未设置 `metric`，内核在两个网口之间随机选路；网口切换或链路抖动时，数据包被发往已失效的网口。

**解决：**

```bash
# 1. 检查当前路由表
ip route                           # 观察是否出现两条相同 default 路由

# 2. 为每个网口的路由设置不同的 metric 值（主口 ≤ 100，备口 ≥ 200）

# 3. 应用配置后验证
ip route | grep default            # 确认只存在一条最优路由
```

---

## 7. 总结

Netplan 静态 IP 配置的稳定性取决于五个关键参数：`dhcp6: false` 和 `accept-ra: false` 封锁 IPv6 干扰，`optional: true` 防止启动阻塞，`metric` 明确路由优先级，`renderer: networkd` 锁定渲染器。这些参数不影响网络"能不能通"，但直接决定在链路波动、网卡切换、系统重启等场景下网络"稳不稳"。

掌握 Netplan 静态 IP 最佳实践后，你可以：

- 编写一次到位的服务器网络配置，消除 `Lost carrier` 触发的 SSH 断连。
- 在多网口场景中实现拔线即切的主备热切换，无需人工干预。
- 使用链路聚合 Bonding 实现真正的零中断容灾。

建议从单网卡场景的"五件套"起步，逐步扩展到多网口和 Bonding 配置，结合装后四步自查确保每次部署的可靠性。

---

## 8. 参考资料

- [Netplan 官方文档](https://netplan.readthedocs.io/)
- [Netplan YAML 配置参考](https://netplan.readthedocs.io/en/stable/netplan-yaml/)
- [systemd-networkd 手册](https://www.freedesktop.org/software/systemd/man/systemd-networkd.html)
- [Linux Bonding 驱动文档](https://www.kernel.org/doc/Documentation/networking/bonding.txt)
- [Ubuntu Server 网络配置指南](https://ubuntu.com/server/docs/network-configuration)
