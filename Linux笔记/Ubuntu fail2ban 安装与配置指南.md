# Ubuntu fail2ban 安装与配置指南

> 适用系统：Ubuntu 20.04 / 22.04 / 24.04  
> 最后更新：2026-05

---

## 目录

1. [简介](#简介)
2. [安装](#安装)
3. [前置知识](#前置知识)
   - [核心概念](#核心概念)
   - [配置文件在哪里](#配置文件在哪里)
4. [从零构建完整配置](#从零构建完整配置)
   - [第一步：先看看系统默认给了什么](#第一步：先看看系统默认给了什么)
   - [第二步：创建自定义文件，设定基础参数](#第二步：创建自定义文件，设定基础参数)
   - [第三步：用变量管理白名单 IP](#第三步：用变量管理白名单-ip)
   - [第四步：切换到 systemd 日志后端](#第四步：切换到-systemd-日志后端)
   - [第五步：逐级惩罚，让攻击者付出递增代价](#第五步：逐级惩罚让攻击者付出递增代价)
   - [第六步：sshd jail 的精细调校](#第六步：sshd-jail-的精细调校)
   - [第七步：架设 recidive 惯犯监狱](#第七步：架设-recidive-惯犯监狱)
5. [完整配置文件一览](#完整配置文件一览)
6. [关键决策说明](#关键决策说明)
   - [为什么行内不能写注释](#为什么行内不能写注释)
   - [为什么显式声明 enabled = true](#为什么显式声明enabled=true)
   - [为什么指定 backend = systemd](#为什么指定backend=systemd)
   - [sshd-session 兼容问题的完整诊断过程](#sshd-session兼容问题的完整诊断过程)
7. [常用管理命令](#常用管理命令)
8. [验证与排错](#验证与排错)
9. [最佳实践](#最佳实践)

---

## 简介

fail2ban 是一个入侵防御框架，通过监控系统日志，自动封禁短时间内多次认证失败的 IP 地址，主要用于防御暴力破解攻击（SSH、HTTP、FTP 等服务）。

**工作原理：**

```
系统日志 → fail2ban 扫描 → 匹配 filter 规则 → 触发 action → iptables/nftables 封禁 IP
```

本文的目标是带你从系统默认状态出发，一步步构建出一份可直接用于生产环境的完整配置文件，并讲清楚每一步「为什么这么做」。

---

## 安装

```bash
# 更新包列表
apt update

# 安装 fail2ban
apt install fail2ban -y

# 启动服务并设置开机自启
systemctl enable --now fail2ban

# 确认运行状态
systemctl status fail2ban
```

---

## 前置知识

在动手写配置之前，先花两分钟搞清楚两个问题：fail2ban 用哪些概念描述封禁规则，以及配置文件放在哪里。

### 核心概念

| 术语 | 说明 |
|------|------|
| **jail（监狱）** | 一组规则的集合，监控某个服务（如 sshd），每个 jail 包含 filter + action + 参数 |
| **filter（过滤器）** | 正则表达式，用于从日志中识别失败尝试 |
| **action（动作）** | 触发封禁后执行的操作，通常是调用 iptables/nftables 封禁 IP |
| **bantime** | 封禁持续时长，`-1` 表示永久 |
| **findtime** | 观察窗口，在此时间内超过 maxretry 次才触发封禁 |
| **maxretry** | 触发封禁的最大失败次数 |
| **ignoreip** | 白名单 IP，永不封禁 |

后续构建配置时会反复用到这些参数，现在有个印象即可。

### 配置文件在哪里

```
/etc/fail2ban/
├── fail2ban.conf          # 主程序配置（日志级别等），不要直接修改
├── fail2ban.d/            # 主程序配置的覆盖目录
├── jail.conf              # 内置 jail 默认配置，不要直接修改
├── jail.d/                # ← 自定义配置放这里
│   ├── defaults-debian.conf   # apt 安装时自动生成
│   └── custom.local           # 自己创建的配置（推荐命名）
├── filter.d/              # 各服务的 filter 正则规则
└── action.d/              # 各种封禁 action 脚本
```

**配置加载优先级（后者覆盖前者）：**

```
jail.conf → jail.d/*.conf（按文件名字母顺序）→ jail.d/*.local
```

> **核心原则：永远不要修改 `.conf` 文件。** 所有自定义配置写入 `jail.d/` 目录下的 `.local` 文件，系统升级时不会被覆盖。

---

## 从零构建完整配置

下面每一步，都在向最终配置文件里添加新的内容。跟着走完七步，你不仅能拿到一份完整的配置，还能理解每行配置是怎么来的、为什么放在那里。

### 第一步：先看看系统默认给了什么

apt 安装 fail2ban 后，系统自动生成了一份最小配置：

**文件路径：** `/etc/fail2ban/jail.d/defaults-debian.conf`

```ini
[sshd]
enabled = true
```

这份文件只做了一件事——把 sshd jail 打开。其余的配置参数（bantime、findtime、maxretry、端口号、日志路径等）全部继承自 `/etc/fail2ban/jail.conf` 的内置定义。

**我们的起点就是：sshd 已启用，但所有参数用的是 fail2ban 的出厂默认值。接下来的每一步，都是在自定义文件里覆盖这些默认值。**

### 第二步：创建自定义文件，设定基础参数

创建我们自己的配置文件：

```bash
vim /etc/fail2ban/jail.d/custom.local
```

先写入最基础的 `[DEFAULT]` 节——这部分参数会被所有 jail 继承：

```ini
[DEFAULT]
# 封禁时长：初次违规封 10 分钟
bantime  = 10m

# 观察窗口：30 分钟内
findtime = 30m

# 最多允许 5 次失败尝试，超过即封禁
maxretry = 5

# 白名单：本机回环 + IPv6 回环
ignoreip = 127.0.0.1/8 ::1
```

**四个参数的含义串起来就是：** 任何一个不在白名单中的 IP，如果在 30 分钟内失败超过 5 次，就封禁 10 分钟。

> **为什么初次 bantime 只设 10 分钟？** 因为我们后面会启用逐级惩罚——初犯只是警告性质，真正让攻击者难受的是反复来犯时翻倍递增的封禁时长。

### 第三步：用变量管理白名单 IP

你需要把自己的管理机 IP 加入白名单，否则配置变更时万一手误输错密码，可能把自己锁在服务器外面。

但不同项目的服务器，管理机 IP 可能不同。如果每台机器都去改 `ignoreip` 行，容易出错。更好的做法是定义一个变量：

```ini
[DEFAULT]
# ==================== 自定义白名单变量 =====================
# 存放项目管理机 IP，支持 CIDR 格式，多个用空格分隔
# 不同项目服务器只需修改这一行
project_management_ips = 10.0.0.0/24 10.0.0.1

bantime  = 10m
findtime = 30m
maxretry = 5

# ignoreip 中引用变量（%(变量名)s 是 fail2ban 的变量展开语法）
ignoreip = 127.0.0.1/8 ::1 %(project_management_ips)s
```

`%(project_management_ips)s` 会被 fail2ban 自动替换为变量的值——等同于把 `10.0.0.0/24 10.0.0.1` 直接拼在 `ignoreip` 行里。换项目时只改 `project_management_ips` 这一行就够了。

### 第四步：切换到 systemd 日志后端

Ubuntu 22.04 及之后的版本，系统日志以 journald 为主。虽然 `/var/log/auth.log` 仍然存在，但它不再是 SSH 认证日志的唯一来源。为了让 fail2ban 准确、实时地获取日志，需要显式指定使用 systemd 后端：

```ini
[DEFAULT]
project_management_ips = 10.0.0.0/24 10.0.0.1

bantime  = 10m
findtime = 30m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1 %(project_management_ips)s

# 从 journald 读取日志，而非轮询文本文件
backend = systemd
```

这样 fail2ban 不再去读 `/var/log/auth.log`，而是直接向 journald 请求日志数据——更实时，也能完整兼容 Ubuntu 22.04 引入的 `sshd-session` 新进程名（详见[关键决策说明](#sshd-session-兼容问题的完整诊断过程)）。

> 如果使用 Ubuntu 20.04 或更早版本，可以省略此行，默认的 `backend = auto` 即可正常工作。

### 第五步：逐级惩罚，让攻击者付出递增代价

默认的封禁策略是「每次违规都封固定时长」。对于真正的攻击者，这可能不够——他们可以在解封后继续尝试。逐级惩罚的思路是：**每次被封，下次封得更久，直到攻击者放弃。**

在 `[DEFAULT]` 节中追加以下参数：

```ini
[DEFAULT]
project_management_ips = 10.0.0.0/24 10.0.0.1

bantime  = 10m
findtime = 30m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1 %(project_management_ips)s

backend = systemd

# ==================== 逐级惩罚配置 ====================

# 开启递增封禁功能
bantime.increment = true

# 每次被封后，下次封禁时长乘以该系数
# 第 1 次：10m × 1 = 10m
# 第 2 次：10m × 2 = 20m
# 第 3 次：10m × 4 = 40m
# 第 4 次：10m × 8 = 80m
# ...以此类推
bantime.factor = 2

# 封禁时长上限（防止无限增长）
bantime.maxtime = 24w

# 每次额外附加 0~10 分钟随机时长，防止攻击者预测精确解封时间
bantime.rndtime = 10m
```

**封禁时长演进示例：**

| 第 N 次被封 | 基础封禁时长 | 加随机后实际范围 |
|:-----------:|:-----------:|:--------------:|
| 第 1 次 | 10 分钟 | 10~20 分钟 |
| 第 2 次 | 20 分钟 | 20~30 分钟 |
| 第 3 次 | 40 分钟 | 40~50 分钟 |
| 第 4 次 | 80 分钟 | 80~90 分钟 |
| 第 5 次 | 160 分钟 ≈ 2.7 小时 | 2.7~2.8 小时 |
| 第 6 次 | 320 分钟 ≈ 5.3 小时 | 5.3~5.5 小时 |
| ... | 直至上限 24 周 | |

`bantime.rndtime` 的作用是让每次封禁时长多一个随机尾巴，攻击者就无法通过精确计时来预测解封时刻，进一步增加了攻击成本。

### 第六步：sshd jail 的精细调校

前面的配置都写在 `[DEFAULT]` 节里，对所有 jail 生效。现在我们需要针对 sshd 这个具体的 jail 做一些精细调整。

```ini
[sshd]
# 显式启用（确保不依赖默认文件的设置）
enabled      = true

# 填写 SSH 实际端口（非 22 需同步修改）
port         = 22

# 精准订阅 SSH 服务的 journald 日志源
journalmatch = _SYSTEMD_UNIT=ssh.service

# 兼容 sshd 和 sshd-session 两种进程名
# Ubuntu 22.04 开始，SSH 进程从 sshd 改成了 sshd-session
# 这条 prefregex 同时匹配两种名称
prefregex    = ^<F-MLFID>(?:\[\])?\s*(?:<[^.]+\.[^.]+>\s+)?(?:\S+\s+)?(?:kernel:\s?\[ *\d+\.\d+\]:?\s+)?(?:@vserver_\S+\s+)?(?:(?:(?:\[\d+\])?:\s+[\[\(]?(?:sshd(?:-session)?)(?:\(\S+\))?[\]\)]?:?|[\[\(]?(?:sshd(?:-session)?)(?:\(\S+\))?[\]\)]?:?(?:\[\d+\])?:?)\s+)?(?:\\[ID \d+ \S+\\]\s+)?</F-MLFID>(?:(?:error|fatal): (?:PAM: )?)?<F-CONTENT>.+</F-CONTENT>$

# 封禁动作：用 iptables 封锁端口
banaction    = iptables-multiport
```

逐行说明：

- **`enabled = true`**：虽然 `defaults-debian.conf` 已经写了，但在这里显式声明可以确保控制权在自己手里，不依赖系统预设（具体原因见[关键决策说明](#为什么显式声明-enabled--true)）。
- **`port = 22`**：如果你的 SSH 监听的不是 22 端口，一定要同步改成实际端口，否则封禁动作仍针对 22 端口，形同虚设。
- **`journalmatch`**：告诉 journald 只取 `ssh.service` 的日志，避免扫描全量日志，提升效率。注意 journald 中的 unit 名是 `ssh.service`，不是 `sshd.service`。
- **`prefregex`**：这是整个 sshd jail 最关键的配置。fail2ban 先用 prefregex 筛选日志行，再用 failregex 匹配失败记录。Ubuntu 22.04 将 SSH 进程名从 `sshd` 改成了 `sshd-session`，默认的 prefregex 只认 `sshd`，导致日志在进入 failregex 之前就被丢弃。通过 `(?:sshd(?:-session)?)` 这个正则片段，同时兼容新旧两种进程名。
- **`banaction`**：指定用 iptables 的多端口封锁动作。

### 第七步：架设 recidive 惯犯监狱

前面六步配置了对单次攻击的封禁和逐级惩罚。但如果一个 IP 在 SSH 被封后换去攻击 HTTP，或在解封后反复来犯——我们需要一个更高层级的监控：**recidive（惯犯）监狱**。

recidive 不监控业务日志（如 SSH 认证日志），而是监控 fail2ban 自身的日志——只要一个 IP 在短时间内被任何 jail 多次封禁，就判定为惯犯，给予长期封禁。

```ini
[recidive]
enabled     = true

# 使用 fail2ban 内置的 recidive 过滤器，无需额外编写
filter      = recidive

# 监控 fail2ban 自身日志
logpath     = /var/log/fail2ban.log

# 统计窗口：1 天内
findtime    = 1d

# 1 天内被任何 jail 封禁 3 次，视为"惯犯"
maxretry    = 3

# 惯犯直接封禁 12 周（约 3 个月）
bantime     = 12w

# 沿用全局白名单，确保管理机不会被误抓
ignoreip    = 127.0.0.1/8 ::1 %(project_management_ips)s
```

**为什么需要 recidive？** 前面 `[DEFAULT]` 中的逐级惩罚需要同一个 IP 在同一个 jail 中反复被封才会升级。如果攻击者打一枪换一个地方——SSH 被封了就换 HTTP，HTTP 被封了等解封再来——每次在新服务上都是「初犯」。recidive 跨 jail 累计封禁次数，堵住了这个漏洞。

> `[sshd]` 节可以加 `bantime.overalljails = true` 让 sshd 的封禁记录参与 recidive 的统计，但这个参数更适合放在业务 jail 而非 `[DEFAULT]` 节，否则 recidive 自己也会被卷入递归统计。

---

## 完整配置文件一览

经过以上七步，最终得到的完整配置文件如下。把它保存到 `/etc/fail2ban/jail.d/custom.local`，执行 `fail2ban-client -t && systemctl reload fail2ban` 即可生效。

```ini
# /etc/fail2ban/jail.d/custom.local
# 生产环境 fail2ban 完整配置
# 换项目时只需修改 project_management_ips 和 port 两行

[DEFAULT]
# ==================== 自定义白名单变量 =====================
# 存放项目管理机 IP，支持 CIDR 格式，多个用空格分隔
# 不同项目服务器只需修改这一行
project_management_ips = 10.0.0.1

# ===================== 基础行为参数 ========================
# 初次封禁时长
bantime  = 10m
# 统计窗口（30 分钟内失败次数）
findtime = 30m
# 最多允许 5 次失败尝试
maxretry = 5
# 白名单 IP（引用上面的变量）
ignoreip = 127.0.0.1/8 ::1 %(project_management_ips)s

# ==================== 日志后端（适配 journald） =============
# 从 journalctl 读取日志
backend = systemd

# ==================== 递增惩罚 + 随机时长 ===================
# 开启递增，再次被封时长翻倍
bantime.increment = true
# 翻倍系数
bantime.factor    = 2
# 最大封禁时长上限（24 周）
bantime.maxtime   = 24w
# 每次额外附加 0~10 分钟随机时长，防攻击者预测
bantime.rndtime   = 10m

# ==================== sshd jail 专用覆盖 ===================
[sshd]
enabled      = true
# 填写 SSH 实际端口
port         = 22
# journal 中 ssh 服务的 unit 名称是 ssh.service，不是 sshd.service
journalmatch = _SYSTEMD_UNIT=ssh.service
# 兼容 sshd 和 sshd-session 两种进程名（Ubuntu 22.04+）
prefregex    = ^<F-MLFID>(?:\[\])?\s*(?:<[^.]+\.[^.]+>\s+)?(?:\S+\s+)?(?:kernel:\s?\[ *\d+\.\d+\]:?\s+)?(?:@vserver_\S+\s+)?(?:(?:(?:\[\d+\])?:\s+[\[\(]?(?:sshd(?:-session)?)(?:\(\S+\))?[\]\)]?:?|[\[\(]?(?:sshd(?:-session)?)(?:\(\S+\))?[\]\)]?:?(?:\[\d+\])?:?)\s+)?(?:\\[ID \d+ \S+\\]\s+)?</F-MLFID>(?:(?:error|fatal): (?:PAM: )?)?<F-CONTENT>.+</F-CONTENT>$
banaction    = iptables-multiport

# ==================== recidive 监狱（惯犯监狱） ===============
[recidive]
enabled     = true
# 过滤器使用内置的 recidive，无需额外编写
filter      = recidive
# 监控 fail2ban 自身日志
logpath     = /var/log/fail2ban.log
# 统计窗口：1 天内
findtime    = 1d
# 1 天内被任何 jail 封禁 3 次，视为"惯犯"
maxretry    = 3
# 惯犯直接封禁 12 周（约 3 个月）
bantime     = 12w
# 沿用全局白名单，确保管理机不会被误抓
ignoreip    = 127.0.0.1/8 ::1 %(project_management_ips)s

```

> **换项目时只需修改 `project_management_ips` 这一行；若改了 SSH 端口，同步修改 `port =` 这一行。其余配置跨项目通用。**

---

## 关键决策说明

以下是对配置中几个关键选择的详细解释——适合想深入了解「为什么这么做」的读者。

### 为什么行内不能写注释

fail2ban 使用类 INI 格式解析器，**配置值中不支持行内注释**。如果写成：

```ini
port = 22   # ssh port
```

解析器会将 `"22   # ssh port"` 整体作为端口值传递给 iptables，导致生成非法命令：

```bash
iptables -A f2b-sshd -p tcp --dport "22   # ssh port" -j REJECT
```

iptables 报错拒绝执行，结果是 fail2ban 认为已封禁，防火墙实际没有生效。**所有配置值后面不要写行内注释，注释单独写一行。**

### 为什么显式声明 enabled = true

虽然 `defaults-debian.conf` 已经写了 `enabled = true`，但在 `custom.local` 中重复声明基于两个原因：

1. **控制权保证**：`.local` 文件优先级高于 `.conf`，显式声明后，即使系统更新重置了 `defaults-debian.conf` 的内容，sshd jail 也不会意外关闭。

2. **不依赖系统预设**：在某些发行版的最小化安装中，sshd jail 默认是关闭的。在自定义文件里显式声明，保证配置在任何机器上都能直接生效。

### 为什么指定 backend = systemd

在 Ubuntu 22.04+ 中，日志系统以 journald 为主。`/var/log/auth.log` 仍然存在但不再是唯一来源。同时 Ubuntu 22.04 引入了 `sshd-session` 新标识符——fail2ban 默认的文本文件轮询模式有时无法正确匹配这种新格式。

设置 `backend = systemd` 后，fail2ban 直接向 journald 请求日志数据，而非读取文本文件。好处有两个：日志获取更实时；能完整兼容 `sshd-session` 标识符。

> 如果使用 Ubuntu 20.04 或更早版本，可以省略此行，默认的 `backend = auto` 即可正常工作。

### sshd-session 兼容问题的完整诊断过程

#### 问题现象

即使攻击者多次输错密码，`fail2ban-client status sshd` 的 `Total failed` 始终为 0，封禁从不触发。

用以下命令可以复现诊断过程：

```bash
# 喂日志给 filter，看匹配结果
journalctl -u ssh --since "1 hour ago" | fail2ban-regex - /etc/fail2ban/filter.d/sshd.conf
```

输出中 `matched = 0`，missed 行全是 `sshd-session` 开头，说明 filter 认不出这个进程名。

#### 根本原因

我们将服务器的 OpenSSH 升级到了10.3+版本，OpenSSH 9.8 及之后的版本 将 SSH 的进程名从 `sshd` 改成了 `sshd-session`，而 fail2ban 内置的 prefregex（预匹配正则）只认 `sshd`，导致所有日志行在进入 failregex 匹配之前就已被过滤掉——卫兵连敌人的面都见不到。

```
日志行：sshd-session[1234]: Failed password for root ...
prefregex：只认 sshd ← 在这一关就拦住了，failregex 根本不执行
```

#### 解决方案

分两步：

**第一步**，在 `[sshd]` 节中覆盖 `prefregex`，加入 `(?:sshd(?:-session)?)` 同时兼容新旧两种进程名：

```ini
prefregex = ^<F-MLFID>(?:\[\])?\s*(?:<[^.]+\.[^.]+>\s+)?(?:\S+\s+)?(?:kernel:\s?\[ *\d+\.\d+\]:?\s+)?(?:@vserver_\S+\s+)?(?:(?:(?:\[\d+\])?:\s+[\[\(]?(?:sshd(?:-session)?)(?:\(\S+\))?[\]\)]?:?|[\[\(]?(?:sshd(?:-session)?)(?:\(\S+\))?[\]\)]?:?(?:\[\d+\])?:?)\s+)?(?:\\[ID \d+ \S+\\]\s+)?</F-MLFID>(?:(?:error|fatal): (?:PAM: )?)?<F-CONTENT>.+</F-CONTENT>$
```

**第二步**，加上 `journalmatch` 让 fail2ban 精准订阅 SSH 服务的日志源：

```ini
journalmatch = _SYSTEMD_UNIT=ssh.service
```

#### 验证方法

配置生效后，从另一台机器故意输错密码，然后：

```bash
fail2ban-client status sshd
# Banned IP list 中出现对方 IP 即为成功
```

---

## 常用管理命令

### 服务管理

```bash
# 重载配置（修改配置后执行，不中断现有封禁）
systemctl reload fail2ban

# 重启服务
systemctl restart fail2ban

# 检查配置语法
fail2ban-client -t
```

### 查看状态

```bash
# 查看所有启用的 jail 列表
fail2ban-client status

# 查看指定 jail 的状态（当前封禁 IP 等）
fail2ban-client status sshd

# 查看 fail2ban 实时日志
tail -f /var/log/fail2ban.log
```

### 手动操作 IP

```bash
# 手动封禁某个 IP（在 sshd jail 中）
fail2ban-client set sshd banip 1.2.3.4

# 手动解封某个 IP
fail2ban-client set sshd unbanip 1.2.3.4

# 查询某个 IP 是否被封禁
fail2ban-client get sshd banned | grep 1.2.3.4
```

### 查询封禁历史

```bash
# 在日志中查询某个 IP 的历史记录（含已轮转日志）
zgrep "1.2.3.4" /var/log/fail2ban.log*
```

---

## 验证与排错

### 配置修改后的标准流程

```bash
# 第一步：检查语法
fail2ban-client -t && echo "语法正确"

# 第二步：重载配置
systemctl reload fail2ban

# 第三步：确认 jail 已启用
fail2ban-client status
```

### 常见问题

**问：修改配置后没有生效？**

```bash
# 确认配置文件被正确加载
fail2ban-client -d

# 检查文件扩展名（必须是 .conf 或 .local，不能是 .local.bak 等）
ls /etc/fail2ban/jail.d/
```

**问：服务启动失败？**

```bash
# 查看详细错误信息
journalctl -u fail2ban -n 50 --no-pager
```

**问：如何确认某个 IP 被正确忽略（白名单）？**

```bash
# 查看当前 DEFAULT 配置中 ignoreip 的实际值
fail2ban-client get sshd ignoreip
```

---

## 最佳实践

1. **白名单一定要加管理机 IP**，防止手误把自己封掉，尤其是配置变更时。

2. **不要直接修改 `jail.conf`**，系统升级会覆盖它，所有变更写入 `jail.d/custom.local`。

3. **修改配置前先备份**：
   ```bash
   cp /etc/fail2ban/jail.d/custom.local /etc/fail2ban/jail.d/custom.local.bak
   ```

4. **先用 `fail2ban-client -t` 检查语法**，再执行 reload。

5. **生产环境建议 maxretry 不低于 3**，避免因手抖或网络抖动把合法用户误封。

6. **修改 SSH 端口后，记得同步更新 `[sshd]` 节的 `port =`**，否则封禁动作仍针对 22 端口，形同虚设。改完执行 `fail2ban-client -t && systemctl reload fail2ban`。

7. **recidive 的 findtime 要大于 sshd 的 findtime**，否则惯犯统计窗口太短，起不到累积效果。本文配置中 sshd 的 findtime 为 30 分钟、recidive 的 findtime 为 1 天，是一个合理的搭配。
