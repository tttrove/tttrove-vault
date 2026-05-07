# Netplan 配置探索笔记

> 起源：排查 VMware Ubuntu 22.04 虚拟机 SSH 周期性断连问题，意外深挖出 Netplan 在生产环境中最容易被忽略的配置细节。

---

## 一、背景与问题现象

**环境：** Windows 11 宿主机（笔记本）+ VMware Workstation + Ubuntu 22.04（NAT 模式，静态 IP 10.0.0.101/24，开机自启）

**故障现象：** 每隔一两天，Xshell 在敲命令途中突然断连，约一分钟后才能从另一个 session 重连。

**排查关键日志：**

```bash
sudo journalctl -n 50 -u systemd-networkd
```

```
ens33: Lost carrier
ens33: DHCPv6 lease lost
ens33: Gained carrier
ens33: Lost carrier
ens33: DHCPv6 lease lost
...
```

**结论：** 不是 IP 层问题，而是链路层（物理层）在频繁抖动。根本原因是笔记本在有线网络与 WiFi 之间切换，触发了 VMware NAT 服务的重绑定机制，导致 VMnet8 虚拟交换机闪断，Ubuntu 内核记录了 `Lost carrier`，进而强制重置 SSH Socket。

---

## 二、核心发现：Netplan 中那些"不起眼"却至关重要的参数

从这次排查延伸出的最大收获，是发现无论是个人虚拟机还是生产环境服务器，都普遍存在 Netplan 配置不完整的问题，以下是最容易被忽略的参数及其原因。

---

### 2.1 `dhcp6: false` —— 禁用 IPv6 DHCP（强烈建议写）

**为什么必须写？**

Ubuntu 20.04+ 的 `systemd-networkd` 默认会开启 IPv6 的 SLAAC（无状态地址自动配置）和 DHCPv6 客户端。**即使你的配置文件里只配了静态 IPv4，没有写 `dhcp6: true`，系统依然会在后台偷偷寻找 IPv6 路由器和 DHCPv6 服务器。**

**后果：**

- 链路波动（`Lost carrier`）时，系统会同时重置静态 IPv4 配置 **和** 发起 IPv6 地址申请。
- 如果机房交换机没有配置 IPv6 或配置了错误的 RA（路由公告），`systemd-networkd` 会陷入无限重试循环。
- 在某些内核版本中，这会触发整个网卡接口被重新初始化，**导致所有现有 TCP 连接（包括 SSH）立即中断**。

**日志证据（生产环境实例）：**

```
Mar 19 16:16:16 eno1: Lost carrier
Mar 19 16:16:16 eno1: DHCPv6 lease lost      ← 没写 dhcp6: false，系统在伤口上撒盐
Mar 19 16:16:22 eno1: Gained carrier
Mar 19 16:16:48 eno1: Lost carrier
Mar 19 16:16:48 eno1: DHCPv6 lease lost
Mar 19 16:19:38 eno1: Gained carrier
Mar 19 16:19:39 eno1: DHCPv6 error: No message of desired type  ← 在不支持IPv6的环境中反复报错
Mar 19 16:21:38 eno1: DHCPv6 error: No message of desired type
```

---

### 2.2 `accept-ra: false` —— 禁用路由公告（配合 dhcp6 一起写）

**什么是 RA（Router Advertisement）？**

IPv6 网络中，路由器会定期广播"路由公告"报文，告诉网络内的设备前缀和默认网关。`systemd-networkd` 默认会监听这些报文并自动配置 IPv6 地址。

**为什么要禁用？**

对于明确使用静态 IP 的服务器，接受 RA 是不必要的行为。在 IPv6 未规划或交换机配置不规范的环境中，接受错误的 RA 甚至可能导致路由表被篡改，影响出向流量。

**结论：** 静态 IP 场景下，`dhcp6: false` 和 `accept-ra: false` 应作为标配组合出现。

---

### 2.3 `optional: true` —— 防止"找不到网卡"导致开机卡住

**场景：** 你的 Netplan 配置文件里写了两块网卡（如 `ens33` 和 `ens160`），但当前系统只有其中一块（比如虚拟机切换了网卡驱动类型）。

**不加 `optional: true` 的后果：** Ubuntu 开机时会**等待约 2 分钟**，尝试让"丢失的网卡"上线，系统启动速度极慢，日志中会出现网络超时等待错误。

**加了之后：** 找不到该网卡时，系统会立即跳过，开机秒进，不影响已有网卡的正常工作。

---

### 2.4 `metric` —— 多网卡路由权重（多网卡必写）

**什么是 metric？**

路由权重，数字越小，优先级越高。当系统有多条可选路由时，内核按 metric 值从小到大选择最优路径。

**不设置 metric 的后果（生产环境真实案例）：**

```yaml
# 危险配置：两块备用网卡，配置完全相同，没有 metric
eno3np0:
  addresses: [172.67.0.43/24]
  routes:
    - to: 172.67.0.0/16
      via: 172.67.0.1
eno4np1:
  addresses: [172.67.0.43/24]  # ← 同一个 IP！
  routes:
    - to: 172.67.0.0/16
      via: 172.67.0.1
```

**双重隐患：**

1. **IP 冲突：** 两块网卡同时持有同一个 IP，如果两根光纤都插着，交换机 ARP 表会发生"MAC 漂移"——发往该 IP 的包一会儿走 A 口，一会儿走 B 口，SSH 连接极度不稳定。

2. **路由失效：** 没有 metric，内核认为两条路由优先级相同，用哈希算法随机选一条。如果其中一根光纤断了，内核依然可能把包往已经死掉的那个接口发，出现"时通时不通"的诡异现象。

**正确做法：** 设置不同的 metric，让系统明确知道谁是"主"、谁是"备"：

```yaml
eno3np0:
  metric: 100   # 主链路
eno4np1:
  metric: 200   # 备用链路（主断了才走这里）
```

---

### 2.5 `renderer: networkd` —— 明确指定渲染器

Ubuntu 22.04 的 Netplan 支持两种后端渲染器：

| 渲染器 | 适用场景 | 说明 |
|---|---|---|
| `networkd` | **服务器（推荐）** | 轻量、无图形界面依赖，适合 headless 环境 |
| `NetworkManager` | 桌面版 | 适合有 GUI 的工作站，支持动态切换网络 |

**为什么要明确写？** 不写时，Netplan 会根据当前运行的服务自动判断，但当 `NetworkManager` 和 `systemd-networkd` 同时运行时（Ubuntu 22.04 安装某些包后常见），两者可能同时管理同一块网卡，形成"双头管理"冲突，导致配置周期性被覆盖。

---

## 三、完整的最佳实践配置模板

### 3.1 单网卡 · 静态 IP（虚拟机 / 单网口服务器）

```yaml
network:
    version: 2
    renderer: networkd          # 明确指定后端
    ethernets:
        ens33:
            dhcp4: false        # 关闭 IPv4 DHCP
            dhcp6: false        # 关闭 IPv6 DHCP（关键！）
            accept-ra: false    # 禁用路由公告（关键！）
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

---

### 3.2 双网卡 · 主备热切换（不用改配置、拔线即切）

**背景：** 生产环境中大批量服务器有多余光口，现场运维希望直接把网线从故障光口拔出插到备用光口，不需要进终端重新配置，即插即用。

```yaml
network:
    version: 2
    renderer: networkd
    ethernets:
        ens33:
            optional: true      # 找不到该网卡时跳过，不阻塞启动
            dhcp4: false
            dhcp6: false
            accept-ra: false
            addresses:
              - 10.0.0.101/24
            routes:
              - to: default
                via: 10.0.0.254
                metric: 100     # 主路由，优先级高
            nameservers:
                addresses:
                  - 223.5.5.5
                  - 119.29.29.29
        ens160:
            optional: true      # 找不到该网卡时跳过
            dhcp4: false
            dhcp6: false
            accept-ra: false
            addresses:
              - 10.0.0.101/24
            routes:
              - to: default
                via: 10.0.0.254
                metric: 200     # 备用路由，优先级低
            nameservers:
                addresses:
                  - 223.5.5.5
                  - 119.29.29.29
```

**效果：**
- 两根网线都在时：流量走 metric 100（ens33），ens160 待命
- ens33 断线后：系统自动切到 metric 200（ens160），无需人工干预
- 切回 ens33 后：系统自动切回优先链路

**注意事项：**
- 两块网卡**不能同时物理插线接入同一个二层交换机**（会引起 ARP/MAC 冲突），主备网卡同一时刻只接一根
- `sudo netplan apply` 后用 `ip route` 验证路由表，确认 metric 值正确生效

---

### 3.3 双网卡 · 链路聚合 Bonding（真正的生产标准，无感切换）

如果希望两根光纤**同时在线**，断一根 SSH 完全无感，应使用链路聚合（Bonding）：

```yaml
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
            interfaces: [eno3np0, eno4np1]
            parameters:
                mode: active-backup   # 主备模式（也可用 802.3ad 做 LACP 聚合）
                fail-over-mac: active
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

**优势：** 两个物理网卡共用一个逻辑 MAC 和一个 IP，对上层应用完全透明，断一根光纤不产生任何 TCP 连接中断。

---

## 四、Netplan 操作速查

```bash
# 检查配置文件语法（不实际应用）
sudo netplan generate

# 试运行（应用 120 秒，超时自动回滚）
sudo netplan try

# 正式应用配置
sudo netplan apply

# 查看当前路由表（验证 metric 生效）
ip route

# 查看网卡状态
ip link

# 实时查看 networkd 日志（排查断连）
sudo journalctl -f -u systemd-networkd
```

---

## 五、SSH 连接稳定性补充优化

即使 Netplan 配置完美，移动办公环境（笔记本切换网络）仍可能导致 SSH 断连。以下是服务端配置建议：

**编辑 `/etc/ssh/sshd_config`：**

```bash
TCPKeepAlive yes
ClientAliveInterval 30   # 每30秒向客户端发送心跳
ClientAliveCountMax 3    # 最多容忍3次无响应后才断开
```

```bash
sudo systemctl restart ssh
```

**终极方案 —— tmux：** 在 Ubuntu 中安装 `tmux`，每次连接后先进入 `tmux` 会话。即使 Xshell 断开，命令仍在后台执行，重连后 `tmux attach` 立即找回现场，彻底消除断连焦虑。

---

## 六、总结：Netplan 配置的"三病灶"模型

| 病灶类型 | 症状日志 | 根本原因 | 解决参数 |
|---|---|---|---|
| **物理病灶** | `Lost carrier` / `Gained carrier` | 网线/WiFi 切换、网卡驱动问题 | 更换 vmxnet3 驱动；调整物理网卡电源管理 |
| **配置病灶** | `DHCPv6 lease lost` / `DHCPv6 error` | 未禁用 IPv6，系统在链路波动时"撒盐" | `dhcp6: false` + `accept-ra: false` |
| **逻辑病灶** | 时通时不通；路由冲突警告 | 多网卡未设 metric，内核随机选路 | `metric` 区分主备；`optional: true` 防卡启动 |

> **核心原则：** 静态 IP 场景下，`dhcp4: false` / `dhcp6: false` / `accept-ra: false` 是三件套，缺一不可。多网卡场景再加上 `metric` 和 `optional: true`，五个参数构成完整防护。

---

*笔记来源：本机虚拟机断连排查 → 生产环境 Netplan 审查，2026年5月*
