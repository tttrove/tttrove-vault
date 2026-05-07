# Ubuntu Netplan 静态 IP 配置最佳实践

> **写给谁看：** 负责装机和网络配置的同事。  
> **解决什么问题：** 装完服务器配完 IP 之后，网络偶发抖动、SSH 莫名断连、多网口切换失效的根源往往不在硬件，而在 Netplan 配置文件里几行没写的参数。  
> **文章结构：** 先说我是怎么发现这件事的 → 看你们现在的配置有没有同款问题 → 每个参数是什么、为什么必须写 → 不同场景的完整配置模板直接复制用 → 装完怎么自查。

---

## 一、这篇文章是怎么来的

起因很简单：我在自己的笔记本上用 VMware 跑了一台 Ubuntu 22.04 虚拟机，设置了静态 IP，用 Xshell 连着干活。后来发现每隔一两天，Xshell 就会在我敲命令的时候突然断连，大约一分钟后才能重连上来。

我以为是 IP 冲突、NAT 服务崩溃、网卡驱动问题……排查了一圈，最后在 `systemd-networkd` 的日志里找到了关键线索：

```bash
sudo journalctl -n 50 -u systemd-networkd
```

```
ens33: Lost carrier
ens33: DHCPv6 lease lost
ens33: Gained carrier
ens33: Lost carrier
ens33: DHCPv6 lease lost
ens33: Gained carrier
...
```

`Lost carrier` 说明网卡在频繁掉线重连，`DHCPv6 lease lost` 说明系统在不该找 IPv6 地址的时候一直在找。问题的直接原因是我的笔记本在有线和 WiFi 之间切换触发了链路抖动，但真正让 SSH 断掉的推手，是 **Netplan 配置文件里几个没写的参数**，导致系统在链路稍微波动的时候做了一堆多余的事，最终把 TCP 连接重置了。

排查完自己的问题之后，我翻了一下生产环境几台服务器的 Netplan 配置，发现我们装机时用的配置和网上绝大多数教程给的配置，都有同样的问题。

这篇文章就是把这件事整理清楚，让后面装机的同事不用再踩一遍。

---

## 二、先看你们现在的配置，对号入座

装机工具（Subiquity）自动生成的 Netplan 配置，通常长这样：

```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    eno1:
      addresses: [10.106.159.6/27]
      routes:
        - to: default
          via: 10.106.159.1
      nameservers:
          addresses: [223.5.5.5, 8.8.8.8]
      dhcp4: false
    eno2:
      dhcp4: true
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

这是我们生产环境某台服务器的真实配置，乍一看没什么问题，网络也能通，但里面至少藏着 **4 个隐患**：

| # | 问题 | 位置 | 后果 |
|---|---|---|---|
| 1 | 没有禁用 IPv6 | 所有网卡都缺 `dhcp6: false` | 链路波动时系统疯狂尝试获取 IPv6 地址，拖垮网络重建速度，严重时断 SSH |
| 2 | 没有禁用路由公告 | 所有网卡都缺 `accept-ra: false` | 交换机可能推来错误的 IPv6 路由，污染路由表 |
| 3 | 两个备用网口配置完全一样，没有权重 | `eno3np0` 和 `eno4np1` 都缺 `metric` | 两口同时在线时内核随机选路；断一个口后流量可能还往死口发 |
| 4 | 没有声明渲染器 | 顶层缺 `renderer: networkd` | `NetworkManager` 和 `systemd-networkd` 可能同时管同一块网卡，周期性互相覆盖配置 |

---

## 三、五个必须写的参数，逐一说清楚

### 3.1 `dhcp4: false` 和 `dhcp6: false` —— 关掉不该开的开关

`dhcp4: false` 大家都知道要写，静态 IP 嘛，不让它自动获取 IPv4。但 **`dhcp6: false` 很多人没写**，因为我们根本没在用 IPv6。

问题在于：Ubuntu 20.04 之后，`systemd-networkd` 默认会在每块网卡上开启 IPv6 的自动地址配置，**不管你有没有在配置文件里写 `dhcp6: true`，它都会悄悄去找**。

表现在日志里就是：只要网卡有一点点抖动，比如交换机重启了某个端口、服务器重启过程中链路瞬断，你就会看到：

```
eno1: DHCPv6 lease lost
eno1: DHCPv6 error: No message of desired type
eno1: DHCPv6 error: No message of desired type
```

这些重试会拖慢网卡重新就绪的速度，严重时会触发网卡接口重新初始化，把现有的所有 TCP 连接（包括 SSH）一起干掉。

**结论：静态 IP 场景下，`dhcp4: false` 和 `dhcp6: false` 是标配，一个都不能少。**

---

### 3.2 `accept-ra: false` —— 不接受交换机推过来的路由广播

RA 是 IPv6 里的"路由公告"机制，路由器会定期广播告诉网段内的设备怎么走网络。`systemd-networkd` 默认会监听并采纳这些公告。

在我们的机房环境里，交换机不一定有规范的 IPv6 配置，有时候会推出去错误的或者混乱的 RA 报文。如果服务器默默接受了，轻则路由表多几条没用的条目，重则默认路由被修改，服务器从外面连不进来。

**和 `dhcp6: false` 配合使用，彻底封锁 IPv6 的干扰。**

---

### 3.3 `optional: true` —— 防止找不到网卡导致服务器开机卡两分钟

这个参数和多网口配置紧密相关。

我们给服务器配了备用网口（比如 `eno3np0` 和 `eno4np1`），在 Netplan 里两个都写上了。但某一天，你把一根网线拔掉了，或者因为某种原因那个网口没认到，启动时系统会看到："配置文件里有 `eno4np1`，但这个网卡找不到，一定是出问题了！"然后它就开始等，足足等 **将近两分钟**，才放弃继续启动。

加上 `optional: true` 之后，意思是告诉系统："这块网卡可选，找不到就直接跳过，别等。"

**凡是配置文件里写了但不保证一定在线的网卡，都应该加这一行。**

---

### 3.4 `metric` —— 告诉内核谁是主路由、谁是备用路由

我们装服务器的时候，经常遇到这种情况：服务器有多余的光口，就把备用光口也配上，万一主口坏了，把网线拔出来插到备口上就能继续用，不用进终端重配。

**这个想法是对的，但配置文件如果写成两个网口完全一样，会出严重问题。**

`metric` 是路由权重，数字越小优先级越高。如果两个网口都指向同一个网关，没有 metric 区分，内核会认为两条路优先级完全一样，然后用哈希算法随机选一条发包。

结果是：

- 两根线都插着 → 流量一会走这口一会走那口，SSH 连接不稳定
- 其中一根断了 → 内核不知道这条路死了，可能还往那个死口发包，出现"时通时不通"

加上 metric 之后，主备关系就明确了：

```yaml
eno3np0:
    routes:
      - to: default
        via: 172.67.0.1
        metric: 100   # 主路由，优先走这里

eno4np1:
    routes:
      - to: default
        via: 172.67.0.1
        metric: 200   # 备用路由，主口断了才走这里
```

**主口在线走主口，主口断了自动切备口，备口恢复后自动切回。拔线即切，完全无需人工干预。**

---

### 3.5 `renderer: networkd` —— 明确告诉系统谁来管网络

Netplan 本身只是个配置翻译层，实际管理网络的是后端渲染器，有两个选择：

- **`networkd`（即 `systemd-networkd`）**：轻量，适合服务器，没有图形界面依赖
- **`NetworkManager`**：功能更丰富，适合桌面工作站，会自动扫描 WiFi 等

Ubuntu Server 22.04 安装完之后理论上默认用 `networkd`，但装了某些软件包（比如 `network-manager`）之后，`NetworkManager` 也会同时启动并尝试接管网卡。不声明 renderer 的话，两个服务可能同时管同一块网卡，配置会周期性互相覆盖。

在配置文件顶层加一行，明确声明，排除隐患：

```yaml
network:
    version: 2
    renderer: networkd
    ethernets:
      ...
```

---

## 四、不同场景的完整配置模板

### 场景 A：单网口服务器，静态 IP

最常见的情况，直接复制，改 IP、网关、DNS 地址：

```yaml
network:
    version: 2
    renderer: networkd
    ethernets:
        eno1:                              # 改成你的网卡名，用 ip link 查
            dhcp4: false
            dhcp6: false                   # 必须写
            accept-ra: false               # 必须写
            addresses:
              - 10.106.159.6/27            # 改成你的 IP 和掩码
            routes:
              - to: default
                via: 10.106.159.1          # 改成你的网关
            nameservers:
                addresses:
                  - 223.5.5.5
                  - 119.29.29.29
```

---

### 场景 B：多网口服务器，备用光口热切换（拔线即用）

适合"光口有富余，用备口做冷备"的场景。两口**同一时刻只接一根网线**，主口坏了直接把线换插到备口。

```yaml
network:
    version: 2
    renderer: networkd
    ethernets:
        eno3np0:
            optional: true                 # 找不到网卡不阻塞启动
            dhcp4: false
            dhcp6: false
            accept-ra: false
            addresses:
              - 172.67.0.43/24
            routes:
              - to: default
                via: 172.67.0.1
                metric: 100                # 主路由
            nameservers:
                addresses:
                  - 223.5.5.5
                  - 119.29.29.29
        eno4np1:
            optional: true
            dhcp4: false
            dhcp6: false
            accept-ra: false
            addresses:
              - 172.67.0.43/24
            routes:
              - to: default
                via: 172.67.0.1
                metric: 200                # 备用路由，主口断了才走这里
            nameservers:
                addresses:
                  - 223.5.5.5
                  - 119.29.29.29
```

> ⚠️ **注意：** 两根网线不要同时插进同一个二层交换机。两个口持有同一个 IP，会引起 ARP/MAC 冲突，反而更乱。这个方案的设计前提是：同一时刻只有一根线插着。

---

### 场景 C：多网口服务器，链路聚合 Bonding（最稳方案）

如果你希望两根光纤**同时在线**，其中一根断了 SSH **完全无感**，用链路聚合（Bond）。

Bond 把两块物理网卡合并成一块逻辑网卡，对外只有一个 IP 和一个 MAC 地址，上层应用感知不到底层切换。`active-backup` 模式下，一块网卡断了，另一块立即接管，中间不产生任何 TCP 中断。

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
            interfaces: [eno3np0, eno4np1]  # 两块物理网卡合成一个 bond0
            parameters:
                mode: active-backup          # 主备模式，一块断了另一块接管
                fail-over-mac: active        # MAC 地址跟随活动接口，减少 ARP 刷新
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

**配置完成后验证 bond 状态：**

```bash
cat /proc/net/bonding/bond0
```

输出里找 `Currently Active Slave`，显示当前活动的物理网卡；`MII Status: up` 说明链路正常。可以拔掉其中一根网线，再执行一次，确认 Active Slave 切换了且网络不断。

---

## 五、装完之后，用这几条命令自查

配置文件写完，`sudo netplan apply` 之后，按以下顺序检查一遍再离开：

**第一步：检查配置文件语法**

```bash
sudo netplan generate
```

没有输出就是没问题；有报错或警告会直接打出来。

**第二步：看路由表是否正确**

```bash
ip route
```

正常输出示例：

```
default via 10.106.159.1 dev eno1 proto static metric 100
10.106.159.0/27 dev eno1 proto kernel scope link src 10.106.159.6
```

检查要点：有没有 `default` 路由，网关 IP 是否正确；如果配了多网口，确认主口的 metric 值更小（数字小 = 优先级高）；有没有出现两条 metric 相同的 `default` 路由（说明 metric 没生效）。

**第三步：看网络日志有没有异常**

```bash
sudo journalctl -n 30 -u systemd-networkd
```

正常情况下只会看到网卡上线的记录：

```
eno1: Gained carrier
eno1: Link UP
```

如果看到 `DHCPv6 lease lost` 或 `DHCPv6 error`，说明 `dhcp6: false` 没有生效，重新检查配置文件缩进格式（YAML 对缩进非常敏感，必须用空格，不能用 Tab）。

**第四步：从另一台机器 SSH 进来验证连通性**

不要只在本机 ping 自己，要从外部机器 SSH 连进来并保持几分钟，确认没有断连。

---

## 六、一张表总结

| 参数 | 适用场景 | 不写的后果 |
|---|---|---|
| `dhcp4: false` | 所有静态 IP | 系统自动获取 IP，覆盖你写的静态地址 |
| `dhcp6: false` | 所有静态 IP | 链路抖动时系统反复尝试获取 IPv6，拖慢网络恢复，严重时断 SSH |
| `accept-ra: false` | 所有静态 IP | 可能接受交换机推送的错误 IPv6 路由，污染路由表 |
| `renderer: networkd` | 所有服务器 | NetworkManager 与 networkd 双头管理，配置周期性被覆盖 |
| `optional: true` | 配置了但不保证一定在线的网卡 | 找不到网卡时开机等待约两分钟 |
| `metric` | 多网口配置了相同路由 | 内核随机选路，故障口不自动切换，流量时通时不通 |

---

> 我从自己虚拟机的一次断连排查开始，发现无论是教程示例还是我们自己的装机配置，这几个参数几乎普遍缺失。它们不影响网络"能不能通"，只影响网络"稳不稳"。
>
> 生产服务器跑着业务，"能通"只是及格线，"稳定"才是目标。希望这篇文章能让后面的装机少走一些弯路。

---

  *2026年5月  |  基于 Ubuntu 22.04 + systemd-networkd 验证*
