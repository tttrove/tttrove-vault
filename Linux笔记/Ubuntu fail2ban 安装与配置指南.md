# Ubuntu fail2ban 安装与配置指南

> 适用系统：Ubuntu 20.04 / 22.04 / 24.04  
> 最后更新：2025-06

---

## 目录

1. [简介](#简介)
2. [安装](#安装)
3. [核心概念](#核心概念)
4. [配置文件结构](#配置文件结构)
5. [默认配置文件](#默认配置文件)
6. [自定义配置文件](#自定义配置文件)
7. [逐级惩罚（防小人）](#逐级惩罚防小人)
8. [常用管理命令](#常用管理命令)
9. [验证与排错](#验证与排错)
10. [最佳实践](#最佳实践)

---

## 简介

fail2ban 是一个入侵防御框架，通过监控系统日志，自动封禁短时间内多次认证失败的 IP 地址，主要用于防御暴力破解攻击（SSH、HTTP、FTP 等服务）。

**工作原理：**

```
系统日志 → fail2ban 扫描 → 匹配 filter 规则 → 触发 action → iptables/nftables 封禁 IP
```

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

## 核心概念

| 术语 | 说明 |
|------|------|
| **jail（监狱）** | 一组规则的集合，监控某个服务（如 sshd），每个 jail 包含 filter + action + 参数 |
| **filter（过滤器）** | 正则表达式，用于从日志中识别失败尝试 |
| **action（动作）** | 触发封禁后执行的操作，通常是调用 iptables/nftables 封禁 IP |
| **bantime** | 封禁持续时长，`-1` 表示永久 |
| **findtime** | 观察窗口，在此时间内超过 maxretry 次才触发封禁 |
| **maxretry** | 触发封禁的最大失败次数 |
| **ignoreip** | 白名单 IP，永不封禁 |

---

## 配置文件结构

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
jail.conf → jail.d/*.conf（按文件名字母顺序）
```

> **原则：永远不要修改 `.conf` 文件**，所有自定义配置写入 `jail.d/` 目录下的 `.local` 或 `.conf` 文件，升级时不会被覆盖。

---

## 默认配置文件

apt 安装后自动生成，**不需要修改**：

```bash
# 文件路径
/etc/fail2ban/jail.d/defaults-debian.conf
```

```ini
[sshd]
enabled = true
```

此文件仅启用了 sshd jail，具体参数（端口、日志路径等）继承自 `jail.conf` 的内置定义。

---

## 自定义配置文件

创建自己的配置文件以覆盖默认值：

```bash
# 创建自定义配置
vim /etc/fail2ban/jail.d/custom.local
```

### 基础版配置

```ini
[DEFAULT]
# -------------------------------------------------------
# 1. 项目管理机白名单（多个 IP 用空格隔开，支持 CIDR）
#    在不同项目的服务器上，只需修改这一行
# -------------------------------------------------------
project_management_ips = 10.0.0.0/24 10.0.0.1

# 封禁时长
bantime  = 1h

# 观察窗口：30 分钟内超过 maxretry 次则封禁
findtime = 30m

# 最大失败次数
maxretry = 5

# 白名单：本机回环 + IPv6 回环 + 管理机 IP
ignoreip = 127.0.0.1/8 ::1 %(project_management_ips)s
```

**参数说明：**

- `%(project_management_ips)s`：fail2ban 的变量引用语法，等同于把变量值展开拼接在此处。
- `[DEFAULT]` 节的参数会被所有 jail 继承，在各 jail 节中可以单独覆盖。

---

## 逐级惩罚（防小人）

> 对反复尝试的攻击者自动加重封禁力度，每次被封后下次时间翻倍。

### 配置方式：bantime.increment 递增模式

在 `custom.local` 的 `[DEFAULT]` 节中追加以下参数：

```ini
[DEFAULT]
project_management_ips = 10.0.0.0/24 10.0.0.1

bantime  = 1h
findtime = 30m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1 %(project_management_ips)s

# -------------------------------------------------------
# 逐级惩罚配置
# -------------------------------------------------------

# 开启递增封禁功能
bantime.increment = true

# 每次被封后，下次封禁时长乘以该系数
# 第 1 次：1h × 1 = 1h
# 第 2 次：1h × 2 = 2h
# 第 3 次：1h × 4 = 4h
# 第 4 次：1h × 8 = 8h
# ...以此类推
bantime.factor = 2

# 封禁时长上限（防止无限增长）
bantime.maxtime = 5w

# 跨所有 jail 合并计数
# 例如：SSH 失败 + HTTP 失败 共同计入惩罚次数
# 防止攻击者换端口/服务后"洗白"记录
bantime.overalljails = true
```

### 封禁时长演进示例

| 第 N 次被封 | 封禁时长 |
|:-----------:|:-------:|
| 第 1 次 | 1 小时 |
| 第 2 次 | 2 小时 |
| 第 3 次 | 4 小时 |
| 第 4 次 | 8 小时 |
| 第 5 次 | 16 小时 |
| 第 6 次 | 32 小时 ≈ 1.3 天 |
| 第 7 次 | 64 小时 ≈ 2.7 天 |
| ... | 直至上限 5 周 |

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
fail2ban-client -d 2>&1 | grep "custom"

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

5. **bantime.overalljails = true 建议开启**，防止攻击者在不同服务间游走规避计数。

6. **日志轮转问题**：fail2ban 会自动处理 `/var/log/fail2ban.log` 的轮转，无需额外配置。

7. **生产环境建议 maxretry 不低于 3**，避免因手抖或网络抖动把合法用户误封。

8. **修改 SSH 端口后，记得同步更新 `[sshd]` 节的 `port =`**，否则封禁动作仍针对 22 端口，形同虚设。改完执行 `fail2ban-client -t && systemctl reload fail2ban`。

---

## 完整配置文件参考

```ini
# /etc/fail2ban/jail.d/custom.local

[DEFAULT]
# -------------------------------------------------------
# 修改此处以适配不同项目的管理机 IP
# -------------------------------------------------------
project_management_ips = 10.0.0.0/24 10.0.0.1

# 基础封禁参数
bantime  = 1h
findtime = 30m
maxretry = 5

# 白名单
ignoreip = 127.0.0.1/8 ::1 %(project_management_ips)s

# 逐级惩罚
bantime.increment    = true
bantime.factor       = 2
bantime.maxtime      = 5w
bantime.overalljails = true

# -------------------------------------------------------
# SSH 服务配置
# 如果修改了 SSH 端口，同步修改下方 port 值，然后 reload
# -------------------------------------------------------
[sshd]
port = 22   # 默认 22，改过 SSH 端口则改为对应值（如 2222）
```

> **换项目时只需修改 `project_management_ips` 这一行；若改了 SSH 端口，同步修改 `port =` 这一行。**
