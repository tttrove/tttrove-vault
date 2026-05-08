# Linux 服务器入侵排查手册

> 适用系统：Ubuntu 20.04+ / Debian 11+ / CentOS 7+  
> 前置条件：root 权限

当怀疑服务器被入侵时，按以下章节顺序逐一排查。每步均给出**可直接执行的命令**和**正常输出对照**，无需跳转查询。

---

## 1. 入侵迹象识别

**排查目的：** 在不惊动攻击者的情况下快速确认系统是否确实被入侵。

### 1.1 CPU 和负载异常

```bash
# 查看系统整体负载，关注 load average 是否显著超出 CPU 核心数
uptime

# 实时监控 CPU 占用，按 P 排序
top -bn1 | head -20

# 按 CPU 使用率降序列出进程
ps aux --sort=-%cpu | head -15
```

**正常基线：** `load average` 的三个值应接近 CPU 核心数；`%CPU` 超过 100% 的进程通常是挖矿程序。

**可疑信号：** 陌生进程名（如 `xmrig`、`kdevtmpfsi`、`[kworkerds]`）持续占用高 CPU，且 `/proc/<pid>/exe` 指向 `/tmp` 或 `/dev/shm`。

### 1.2 内存异常

```bash
# 查看内存占用概况
free -h

# 按内存使用率降序列出进程
ps aux --sort=-%mem | head -15
```

### 1.3 网络流量异常

```bash
# 查看网络接口流量统计（对比日常基线）
ip -s link

# 查看所有 TCP 连接状态汇总
ss -s

# 列出所有已建立的连接
ss -antp state established
```

**可疑信号：** 大量 `ESTABLISHED` 连接到同一个陌生 IP 的同一端口；端口号为非标准服务端口（如 4444、8080、9999 等常见 C2 端口）。

### 1.4 登录时间线异常

```bash
# 查看最近 30 条登录记录
last -30

# 查看失败的登录尝试
lastb -30 2>/dev/null || journalctl -u ssh --no-pager | grep 'Failed password' | tail -30

# 查看当前在线用户及其 IP
who
w
```

**可疑信号：** 凌晨时段的 root 登录、来自境外 IP 的登录、同一用户在极短时间内从不同 IP 登录。

### 1.5 Shell 环境劫持识别

SSH 登录后发现 `ls` 没有颜色、常用别名（`ll`、`la`）丢失、文件"看不到"——这些是攻击者劫持 Shell 环境以隐藏恶意文件的典型信号。

**环境被劫持的典型表现：**

| 现象 | 可能原因 |
|------|----------|
| `ls` 无颜色输出 | `alias ls='ls'` 或 `--color=never` 被强制设置 |
| `ll` 命令不存在 | 别名被 `unalias` 移除 |
| 文件"看不见" | `LD_PRELOAD` 劫持了 `readdir()`/`getdents()` |
| `top` 看不到某进程 | `LD_PRELOAD` 劫持了 `/proc` 读取函数 |

**立即排查命令：**

```bash
# 检查当前 shell 的所有别名
alias

# 检查是否设置了 LD_PRELOAD（最常见劫持手段）
env | grep -iE 'LD_PRELOAD|LD_LIBRARY_PATH'

# 检查 PROMPT_COMMAND 是否被注入恶意代码
echo "$PROMPT_COMMAND"

# 如果 env 不显示 LD_PRELOAD，直接检查 /proc
cat /proc/self/environ | tr '\0' '\n' | grep LD_PRELOAD
```

**正常基线：** `alias ls='ls --color=auto'`，`alias ll='ls -alF'`，`LD_PRELOAD` 为空。

**遇到劫持时的自救：**

```bash
# 启动一个干净的非交互 bash，绕过可能被污染的 rc 文件
/bin/bash --norc --noprofile

# 或使用 busybox（攻击者通常不会替换）
busybox ls -la /tmp/
busybox ps aux
```

**劫持的来源排查（SSH 登录环境）：**

攻击者可能通过以下文件在 SSH 登录时注入环境变量或命令：

```bash
# 1. ~/.ssh/rc — SSH 登录后自动执行的脚本
cat ~/.ssh/rc 2>/dev/null

# 2. ~/.ssh/environment — SSH 登录时注入的环境变量
cat ~/.ssh/environment 2>/dev/null

# 3. 检查 sshd 是否允许用户自定义环境
grep 'PermitUserEnvironment' /etc/ssh/sshd_config
# 若输出 "PermitUserEnvironment yes" 且 ~/.ssh/environment 存在 — 已确认被利用
```

---

## 2. 登录审计

**排查目的：** 确认攻击者通过 SSH 使用了哪些用户名、从哪些 IP 登录成功。

> **重要：** 优先使用 `journalctl` 读取日志，因为 **journald 将日志存储于二进制文件中（`/var/log/journal/`），攻击者更难清除或篡改**；而 `/var/log/auth.log` 是纯文本文件，易被 `echo "" > /var/log/auth.log` 一键清空。

### 2.1 提取密码登录成功的记录

```bash
# 提取所有密码认证成功的用户名与源 IP，去重（-E 扩展正则，捕获组无需转义）
journalctl -u ssh --no-pager \
  | sed -En 's/.*Accepted password for (\S+) from (\S+).*/\1 \2/p' \
  | sort -u
```

**正常基线：** 只有你预期的用户名和 IP。

**可疑信号：** `root` 出现在列表中；来自不认识 IP 的登录；存在不认识用户名（如 `admin`、`test`、`git`、`postgres` 等）。

### 2.2 按用户名统计独立 IP 数量

```bash
# 每个用户关联了多少不同 IP
journalctl -u ssh --no-pager \
  | sed -En 's/.*Accepted password for (\S+) from (\S+).*/\1 \2/p' \
  | sort -u \
  | awk '{cnt[$1]++} END {for(u in cnt) print cnt[u], u}' \
  | sort -rn
```

### 2.3 提取公钥认证登录记录

```bash
# 公钥登录也可能被利用（如果攻击者上传了自己的公钥）
journalctl -u ssh --no-pager \
  | sed -En 's/.*Accepted publickey for (\S+) from (\S+).*/\1 \2/p' \
  | sort -u
```

### 2.4 查看所有认证成功与失败的时序

```bash
# 按时间线混合排列，看爆破和成功的先后关系
journalctl -u ssh --no-pager \
  | grep -E 'Accepted (password|publickey)|Failed password' \
  | tail -100
```

**解读：** 如果在同一分钟内出现大量 `Failed password` 后紧跟 `Accepted password`，那就是典型的爆破成功。

### 2.5 如果 journalctl 日志已被清空

```bash
# 检查 journald 日志是否被手动清除
journalctl --verify

# 检查日志保留周期是否异常缩短
grep -E 'MaxFileSec|MaxRetentionSec|SystemMaxUse' /etc/systemd/journald.conf 2>/dev/null
```

`journalctl --verify` 输出中如果有 `FAIL`，说明二进制日志已被损坏或篡改。

---

## 3. 账号与权限审计

**排查目的：** 找出攻击者创建的后门账号或被篡改的权限配置。

### 3.1 审计 /etc/passwd 中的非预期用户

```bash
# 列出所有 UID ≥ 1000 的用户（正常用户范围）
awk -F: '($3>=1000){print $1, $3}' /etc/passwd

# 检查是否有 UID 0 的非 root 账号（隐藏超级用户）
awk -F: '($3==0){print $1, $3, $7}' /etc/passwd

# 查看所有能登录的用户（shell 不是 nologin 或 false）
awk -F: '($7!~/(nologin|false)$/){print $1, $3, $7}' /etc/passwd
```

**正常基线：** UID 0 应该只有 `root` 一行。

**可疑信号：** 出现了不认识的高 UID 用户，或 UID 0 不止 `root` 一个。

### 3.2 审计 /etc/shadow

```bash
# 查找空密码账户
awk -F: '($2==""){print $1}' /etc/shadow

# 查找无密码账户（!或*是合法的锁定状态，检查是否有异常）
awk -F: '($2!~/^[!*]/ && $2!=""){print $1, $2}' /etc/shadow | awk '{if(length($2)<13) print}' 
```

### 3.3 审计 sudoers

```bash
# 查看主配置文件
cat /etc/sudoers 2>/dev/null | grep -v '^#' | grep -v '^$'

# 查看 sudoers.d 下所有文件
find /etc/sudoers.d/ -type f -exec echo "--- {} ---" \; -exec grep -v '^#' {} \;

# 查看哪些用户拥有 sudo 权限（用 awk 替代 grep -P，避免 PCRE 依赖）
awk -F: '/^sudo:/{print $NF}' /etc/group
```

**可疑信号：** `ALL=(ALL) NOPASSWD: ALL` 出现在不认识的用户或组上。

### 3.4 审计 SSH authorized_keys

```bash
# 检查所有用户的 authorized_keys
for user_home in /home/* /root; do
    user=$(basename "$user_home")
    keyfile="$user_home/.ssh/authorized_keys"
    if [ -f "$keyfile" ]; then
        key_count=$(grep -c '^ssh-' "$keyfile")
        echo "[$user] $keyfile ($key_count keys)"
        grep '^ssh-' "$keyfile" | awk '{print $1, $3}'
    fi
done
```

**正常基线：** 每个 key 的 `$3`（comment 字段）是你认识的标识。

**可疑信号：** 出现了不认识的公钥；authorized_keys 文件的修改时间晚于你最后一次添加公钥的时间。

> **进阶检查：** 攻击者常用 `chattr +i` 锁定 `authorized_keys`，使其无法被删除或覆盖，同时破坏 `.ssh` 目录的文件创建权限阻止 `known_hosts` 生成。若遇到 `Operation not permitted` 或无法创建文件，先查扩展属性：
>
> ```bash
> # 查看所有文件的扩展属性（i = 不可变，a = 仅追加）
> lsattr /root/.ssh/ /home/*/.ssh/ 2>/dev/null
>
> # 移除不可变属性
> chattr -i /root/.ssh/authorized_keys
> chattr -i /root/.ssh/
> ```

### 3.5 检查 passwd 和 shadow 的修改时间

```bash
ls -la /etc/passwd /etc/shadow /etc/group /etc/gshadow /etc/sudoers
```

**可疑信号：** 修改时间与系统安装或合法配置变更时间不符。

---

## 4. 进程与网络排查

**排查目的：** 找出正在运行的恶意进程及其网络通信。

### 4.1 检查进程树

```bash
# 以树形结构查看所有进程，父进程为 1 的进程可能是守护进程或 orphan
ps auxf

# 查找进程名模仿系统进程的恶意程序（如 sshd 写成 sshhd、cron 写成 crond 等）
ps aux | grep -v grep | awk '{print $11}' | sort | uniq -c | sort -rn | head -20
```

**正常基线：** 关键系统进程应只有唯一实例，且来自标准路径。

**可疑信号：** 同名进程出现多个实例（如多个 `sshd` 但只有一个系统 sshd）；进程名包含大小写混淆（`SShd`、`SSHd`）。

> **高危模式：伪装内核线程的挖矿程序。** 攻击者将恶意进程命名为与内核线程高度相似的名称以躲避排查：

| 伪装名 | 模仿的内核线程 | 辨别方法 |
|--------|--------------|----------|
| `kswapd00` | `kswapd0` | 真正的 `kswapd0` PID 较小且 CPU 占用极低；名字多了个 `0` |
| `kauditd0` | `kauditd` | 真正的 `kauditd` 几乎不占 CPU |
| `kdevtmpfsi` | `kdevtmpfs` | 尾部多了 `i` |
| `[kworkerds]` | `[kworker/*]` | 真正的 kworker 用方括号且名称格式为 `kworker/uX:Y` |

```bash
# 快速比对：找出不是内核线程但名称可疑的进程
ps aux | grep -E 'kswapd|kauditd|kdevtmpfs|kworkerds' | grep -v 'grep'

# 确认是否真的是内核线程（内核线程的 PID 通常很小，且 /proc/<pid>/exe 不存在）
for name in kswapd0 kauditd kdevtmpfs; do
    pid=$(pgrep "$name" 2>/dev/null)
    if [ -n "$pid" ]; then
        for p in $pid; do
            exe=$(readlink /proc/$p/exe 2>/dev/null)
            echo "$name PID=$p EXE=${exe:-(无，内核线程)}"
        done
    fi
done
```
>
> **辨别真伪的关键：** 真正的内核线程 `/proc/<pid>/exe` 不存在（`readlink` 返回空）；伪装的挖矿程序 `exe` 会指向 `/tmp/` 或隐藏目录下的可执行文件，且大量占用 CPU 核心。

### 4.2 检查被删除但仍在运行的进程

攻击者常运行恶意程序后删除文件，这样 `ls` 看不到但进程仍在跑。

```bash
# 列出所有 exe 指向 "(deleted)" 的进程
ls -la /proc/*/exe 2>/dev/null | grep deleted

# 更详细：显示 PID、comm、cmdline
for pid in $(ls /proc/ | grep -E '^[0-9]+$'); do
    exe=$(readlink /proc/$pid/exe 2>/dev/null)
    if echo "$exe" | grep -q 'deleted'; then
        cmdline=$(tr '\0' ' ' < /proc/$pid/cmdline 2>/dev/null)
        echo "PID=$pid CMD=$cmdline (exe deleted)"
    fi
done
```

### 4.3 审计网络连接

```bash
# 列出所有 LISTEN 端口
ss -tlnp

# 列出所有非本地连接的 ESTABLISHED 连接
ss -antp state established | grep -v '127.0.0.1'
```

**可疑信号：** `LISTEN 0.0.0.0:4444` 等非标准端口；进程名是 `/tmp/` 下或隐藏目录的文件；大量连接到同一个外网 IP 的同一个端口。

### 4.4 检查进程的工作目录和完整命令行

```bash
# 对可疑 PID 执行
ls -la /proc/<PID>/cwd       # 工作目录
cat /proc/<PID>/cmdline      # 完整命令行（NUL 分隔）
cat /proc/<PID>/environ      # 环境变量（NUL 分隔）
cat /proc/<PID>/maps         # 内存映射（可能暴露加载的 .so）
```

### 4.5 检查内核模块

```bash
# 列出已加载的内核模块
lsmod

# 模块详情
modinfo <module_name>
```

**可疑信号：** 不认识的模块名、模块签名缺失（`signer:` 为空）、模块加载路径在 `/tmp` 下。

---

## 5. 持久化机制排查

**排查目的：** 找出攻击者为维持访问而设置的自动启动机制。

### 5.1 Cron 任务审计

```bash
# 系统级 crontab
cat /etc/crontab

# 所有用户的 crontab
for user in $(awk -F: '$7!~/(nologin|false)$/{print $1}' /etc/passwd); do
    cron=$(crontab -u "$user" -l 2>/dev/null)
    if [ -n "$cron" ]; then
        echo "=== $user ==="
        echo "$cron"
    fi
done

# cron 系统目录
ls -laR /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.weekly/ /etc/cron.monthly/ 2>/dev/null

# systemd 定时器
systemctl list-timers --all --no-pager
```

**可疑信号：** 定时任务指向 `/tmp/`、`/dev/shm/`、`/var/tmp/` 下的脚本；`curl` 或 `wget` 往陌生 URL 下发命令的任务；每分钟执行的异常任务。

### 5.2 systemd service 审计

```bash
# 列出所有 enabled 的 service
systemctl list-unit-files --state=enabled --no-pager

# 检查 /etc/systemd/system/ 下非系统默认的 service 文件
ls -la /etc/systemd/system/multi-user.target.wants/
ls -la /etc/systemd/system/*.service

# 检查每个自定义 service 的内容
for svc in /etc/systemd/system/*.service; do
    if [ -f "$svc" ]; then
        echo "=== $svc ==="
        grep -E '^(ExecStart|ExecStartPre|ExecStartPost|ExecReload)=' "$svc"
    fi
done
```

**可疑信号：** ExecStart 指向奇怪路径；service 文件名仿冒系统服务（如 `cron.service` vs 系统的 `cron.service`）。

### 5.3 Shell 启动文件

```bash
# 检查所有用户的 rc/profile/bashrc
for user_home in /home/* /root; do
    for rc in .bashrc .profile .bash_profile .zshrc; do
        f="$user_home/$rc"
        if [ -f "$f" ]; then
            suspicious=$(grep -n -E '(curl|wget|nc|bash -i|/dev/tcp)' "$f" 2>/dev/null)
            if [ -n "$suspicious" ]; then
                echo "=== SUSPICIOUS: $f ==="
                echo "$suspicious"
            fi
        fi
    done
done

# 全局 profile
cat /etc/profile /etc/bash.bashrc 2>/dev/null | grep -v '^#' | grep -v '^$'
```

### 5.4 LD_PRELOAD 劫持

```bash
# 检查是否被设置
cat /etc/ld.so.preload 2>/dev/null

# 检查当前 shell 环境
env | grep LD_PRELOAD

# 检查所有内核模块是否有钩子
cat /proc/sys/kernel/tainted
```

**正常基线：** `/etc/ld.so.preload` 不存在或为空；`LD_PRELOAD` 环境变量未设置；`tainted` 值为 0。

**可疑信号：** `ld.so.preload` 指向 `/tmp/` 下的 `.so` 文件（用于劫持 libc 函数如 `readdir`、`open` 等，隐藏进程和文件）。

### 5.5 SSH rc 与环境变量注入

攻击者在获得 root 权限后，常利用 `~/.ssh/rc` 和 `~/.ssh/environment` 在每次 SSH 登录时重新加载恶意环境。

```bash
# 检查 SSH rc 文件（每次 SSH 登录时自动执行）
cat ~/.ssh/rc 2>/dev/null
for h in /home/*; do
    [ -f "$h/.ssh/rc" ] && echo "=== $h/.ssh/rc ===" && cat "$h/.ssh/rc"
done

# 检查 SSH environment 文件（注入环境变量如 LD_PRELOAD）
cat ~/.ssh/environment 2>/dev/null
for h in /home/*; do
    [ -f "$h/.ssh/environment" ] && echo "=== $h/.ssh/environment ===" && cat "$h/.ssh/environment"
done

# 确认 sshd 是否开启了 PermitUserEnvironment（攻击者利用的前提）
grep '^PermitUserEnvironment' /etc/ssh/sshd_config
```

**正常基线：** `~/.ssh/rc` 和 `~/.ssh/environment` 通常不存在；`PermitUserEnvironment` 默认 `no`。

**可疑信号：** `rc` 中包含 `export LD_PRELOAD=...`；`environment` 中设置了指向 `/tmp/` 下 `.so` 文件的变量；`PermitUserEnvironment` 被改为 `yes`。

---

## 6. 文件与隐藏目录

**排查目的：** 找出攻击者留在文件系统上的后门、脚本和隐藏目录。

### 6.1 查找近期修改/创建的可执行文件

```bash
# 近 7 天修改过的文件（排除 /proc 和 /sys）
find / -path /proc -prune -o -path /sys -prune -o -type f -mtime -7 -ls 2>/dev/null | head -50

# 近 1 天内创建的新文件
find / -path /proc -prune -o -path /sys -prune -o -type f -ctime -1 -ls 2>/dev/null | head -50

# 关键目录：/bin /sbin /usr/bin /usr/sbin /usr/local/bin 中近期修改的文件
find /bin /sbin /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -mtime -7 -ls 2>/dev/null
```

### 6.2 查找世界可写的可执行文件

```bash
# 全局可写的 ELF 文件
find / -path /proc -prune -o -path /sys -prune -o -type f -perm -o+w -exec file {} \; 2>/dev/null | grep ELF

# 全局可写的脚本
find / -path /proc -prune -o -path /sys -prune -o -type f -perm -o+w -name '*.sh' 2>/dev/null
find /etc/cron.d -type f -perm -o+w 2>/dev/null
```

### 6.3 查找伪隐藏目录

攻击者常用「点+空格」命名的目录伪装隐藏（如 `.. ` 即两个点加空格，在 `ls` 输出中不易发现）。

```bash
# 查找以 .. 开头的目录（包括 .. 加空格）
find / -maxdepth 3 -type d -name '.. *' -o -name '..' -o -name '...*' 2>/dev/null

# 查找 . 开头且名称中带空格的可疑目录
find /tmp /var/tmp /dev/shm -maxdepth 2 -type d -name '.*' 2>/dev/null
```

### 6.4 排查临时目录中的可执行文件

```bash
# /tmp、/dev/shm、/var/tmp 中的可执行文件
find /tmp /dev/shm /var/tmp -type f -executable 2>/dev/null

# 在这些目录中运行的无主进程
for pid in $(ls /proc/ | grep -E '^[0-9]+$'); do
    cwd=$(readlink /proc/$pid/cwd 2>/dev/null)
    case "$cwd" in
        /tmp*|/dev/shm*|/var/tmp*)
            cmdline=$(tr '\0' ' ' < /proc/$pid/cmdline 2>/dev/null)
            echo "PID=$pid CWD=$cwd CMD=$cmdline"
            ;;
    esac
done
```

### 6.5 SUID/SGID 文件审计

```bash
# 列出所有 SUID 文件
find / -path /proc -prune -o -type f -perm -4000 -ls 2>/dev/null

# 列出所有 SGID 文件
find / -path /proc -prune -o -type f -perm -2000 -ls 2>/dev/null
```

**可疑信号：** 非系统标准路径下的 SUID 二进制（如 `/tmp/su`、`/var/tmp/bash`）、攻击者自制的 SUID shell。

### 6.6 chattr +i 文件锁检测

攻击者获取 root 权限后，常用 `chattr +i` 将关键文件设为**不可变（immutable）**，防止被删除或覆盖，同时破坏目录的文件创建能力（如阻止 `known_hosts` 生成，导致每次 SSH 都出现指纹提示）。这是"占坑锁死"战术。

```bash
# 检查 .ssh 目录及其子文件的扩展属性
lsattr /root/.ssh/ /home/*/.ssh/ 2>/dev/null

# 全系统搜索设置了 +i 或 +a 属性的文件
find / -path /proc -prune -o -path /sys -prune -o -type f -exec lsattr {} \; 2>/dev/null | grep -E '^.{4}i'
```

**属性标记解读：**

| 标记 | 含义 | 攻击者的目的 |
|------|------|------------|
| `i` | 不可变（immutable），连 root 也不能删除/修改/重命名 | 锁死 `authorized_keys` 或 `.ssh` 目录 |
| `a` | 仅追加（append-only），只能追加内容不能删除 | 保护日志不被清空的同时阻止管理操作 |
| `ia` | 同时设置不可变+仅追加，双锁组合 | 连 `chattr -i` 单独解都不行，必须 `chattr -ia` 一起解 |

**解锁：**

```bash
chattr -ia /root/.ssh/authorized_keys
chattr -ia /root/.ssh/
```

**常见被锁目标：** `/root/.ssh/authorized_keys`、`/root/.ssh/`（整个目录）、`~/.bash_history`（阻止记录操作）。

**注意：** 攻击者也可能在 `/etc/`、`/usr/bin/` 等关键路径设置 `+i`，阻止你替换被感染的二进制文件。

---

## 7. Rootkit 检测

**排查目的：** 检测内核级或用户级 rootkit 是否存在。

### 7.1 rkhunter

```bash
# 安装（Ubuntu/Debian）
apt install rkhunter -y

# 更新特征库
rkhunter --update

# 运行全量扫描（--sk 跳过按键确认）
rkhunter --check --sk

# 查看报告
cat /var/log/rkhunter.log
```

重点关注 `Warning` 行，特别是文件属性变更、隐藏文件和可疑字符串。

### 7.2 chkrootkit

```bash
# 安装
apt install chkrootkit -y

# 运行扫描
chkrootkit | grep -E 'INFECTED|suspicious|not found|Warning'
```

### 7.3 Diamond 式 rootkit 检测（proc vs ps 对比）

许多 rootkit 通过 hook 系统调用来隐藏进程。直接读 `/proc` 可绕过 hook：

```bash
# 列出 /proc 中所有 PID（这是真实的进程表）
ls /proc/ | grep -E '^[0-9]+$' | sort -n > /tmp/proc_pids.txt

# 列出 ps 看到的所有 PID
ps -eo pid | tail -n +2 | sort -n > /tmp/ps_pids.txt

# 比较差异：ps 中不存在但 /proc 中存在的 PID 就是被隐藏的进程
diff /tmp/ps_pids.txt /tmp/proc_pids.txt
```

### 7.4 校验系统二进制完整性（仅 Debian/Ubuntu 系）

```bash
# 安装 debsums
apt install debsums -y

# 校验所有已安装包的二进制（仅报告变更）
debsums -c 2>&1 | grep -v 'OK$'
```

**可疑信号：** 核心命令（`ls`、`ps`、`ss`、`netstat`、`top`）校验失败，说明已被替换。

### 7.5 内核 taint 检查

```bash
# 检查内核是否被污染（加载了非 GPL 模块等）
cat /proc/sys/kernel/tainted

# 解码 taint 值
# 0 = 干净
# 1 = 专有模块加载
# 4096 = 内核崩溃后未正确处理
```

---

## 8. 日志审计与异常

**排查目的：** 判断日志是否被篡改或清空，获取未被删除的操作痕迹。

### 8.1 检查日志时间空洞

```bash
# 检查 auth.log / syslog 最后一行的时间戳是否与当前时间差距过大
tail -1 /var/log/auth.log 2>/dev/null
tail -1 /var/log/syslog 2>/dev/null

# 查看 syslog 是否在某个时间点之后为空（被 echo 清空）
wc -l /var/log/syslog /var/log/auth.log /var/log/kern.log 2>/dev/null

# journalctl 检查日志覆盖范围
journalctl --no-pager --lines=1           # 最近一条
journalctl --no-pager --reverse --lines=1  # 最旧一条
```

**可疑信号：** `/var/log/auth.log` 只有几百字节或从某个时间点开始全无记录；`journalctl` 没有早于今天的数据。

### 8.2 校验 journald 日志完整性

```bash
journalctl --verify
```

**正常输出：** 所有条目均为 `PASS`。

**可疑输出：** 出现 `FAIL` 或 `Data object references invalid entry`，说明日志被破坏。

### 8.3 .bash_history 完整性

```bash
# 检查各用户 history
for user_home in /home/* /root; do
    hist="$user_home/.bash_history"
    if [ -f "$hist" ]; then
        echo "=== $hist ==="
        echo "  Size: $(wc -l < "$hist") lines, $(stat --format=%s "$hist" 2>/dev/null || stat -c%s "$hist") bytes"
        echo "  Last 10:"
        tail -10 "$hist"
    fi
done
```

**可疑信号：** `root` 的 `.bash_history` 为空或只有几行；是否被软链接到 `/dev/null`：

```bash
ls -la /root/.bash_history /home/*/.bash_history
# 如果显示为 /dev/null → .bash_history → /dev/null，说明攻击者禁用了历史记录
```

### 8.4 检查日志发送配置是否被篡改

```bash
# rsyslog 配置是否把 auth 日志发送到了别处
grep -r 'auth' /etc/rsyslog.conf /etc/rsyslog.d/ 2>/dev/null | grep -v '^#'

# 检查是否安装/运行了 syslog-ng 等可能被修改的日志服务
systemctl list-units --type=service --state=running --no-pager | grep -iE 'syslog|rsyslog|journal'
```

---

## 9. 应急处置与溯源优先级

**排查目的：** 确认被入侵后，按正确顺序行动，避免误操作导致证据丢失或触发攻击者的反制手段。

### 9.1 处置决策树

```
确认被入侵
    │
    ├─ 保留现场模式 → 用于司法取证 / 深入了解攻击手法
    │   ├─ 不断网
    │   ├─ 不重启、不关机
    │   ├─ 不删除任何文件
    │   ├─ 优先采集 volatile 数据（见 9.2）
    │   └─ 取证后用新机器替换
    │
    ├─ 隔离模式 → 阻止横向移动，保留当前状态
    │   ├─ iptables 封禁所有出站连接（保留 SSH 管理端口除外）
    │   ├─ 通知相关人员
    │   └─ 开始排查（执行第 1-8 章）
    │
    └─ 紧急切断模式 → 立即阻止持续损害
        ├─ 拔网线 / 控制台关闭网卡
        ├─ 或物理关机
        └─ 从只读介质（LiveCD）启动后获取硬盘内容
```

### 9.2 取证采集优先级

按数据易失性排序（从最易失到最稳定）：

| 优先级 | 数据类型 | 采集命令 |
|--------|----------|----------|
| 1 | 内存 | `dd if=/dev/fmem of=/secure/mem.dd` 或 `LiME` 模块采集 |
| 2 | 网络连接状态 | `ss -antp > /secure/net.txt`，`arp -a > /secure/arp.txt` |
| 3 | 进程信息 | `ps auxf > /secure/ps.txt`、`ls -laR /proc > /secure/proc.txt` |
| 4 | 内核和模块 | `lsmod > /secure/lsmod.txt`、`cat /proc/sys/kernel/tainted` |
| 5 | 登录会话 | `who -a > /secure/who.txt`、`last > /secure/last.txt` |
| 6 | 文件系统时间戳 | `find / -ls > /secure/find_ls.txt` |
| 7 | 日志 | `journalctl --no-pager > /secure/journal.txt`、复制 `/var/log/*` |

```bash
# 一键采集 volatile 数据脚本框架
timestamp=$(date +%Y%m%d_%H%M%S)
outdir="/root/forensics_$timestamp"
mkdir -p "$outdir"

date > "$outdir/00_timestamp.txt"
ps auxf > "$outdir/01_ps.txt"
ss -antp > "$outdir/02_net.txt"
who -a > "$outdir/03_who.txt"
last -100 > "$outdir/04_last.txt"
lsmod > "$outdir/05_lsmod.txt"
journalctl --no-pager > "$outdir/06_journal.txt" 2>/dev/null
mount > "$outdir/07_mount.txt"
```

### 9.3 攻击者共存时的隐蔽排查

如果担心攻击者正在终端上监视，使用以下技巧降低暴露风险：

```bash
# 使用 busybox 替代被替换的工具
busybox ps
busybox netstat -antp
busybox cat /proc/1/cmdline

# 命令行中不显示敏感关键词
echo -e '#!/bin/bash\nps auxf' > /tmp/x && chmod +x /tmp/x && /tmp/x

# 将输出静默写入文件而非终端
ps auxf > /tmp/.res 2>&1; scp /tmp/.res user@safehost:
```

---

## 10. 加固检查清单

**排查目的：** 处理完入侵后的安全加固，防止二次沦陷。

### 10.1 SSH 加固

```bash
# /etc/ssh/sshd_config 核心加固项
PasswordAuthentication no      # 禁用密码登录
PermitRootLogin no             # 禁用 root 直接 SSH 登录
PubkeyAuthentication yes       # 仅允许公钥认证
Protocol 2                     # 仅使用 SSH 协议 2
MaxAuthTries 3                 # 单连接最多认证尝试次数
ClientAliveInterval 300        # 空闲超时断开
ClientAliveCountMax 0          
AllowUsers your_username       # 白名单：仅允许指定用户登录
```

修改后：

```bash
sshd -t && systemctl restart sshd
```

### 10.2 fail2ban

```bash
# 安装
apt install fail2ban -y

# 基本 SSH jail 配置 /etc/fail2ban/jail.local
cat > /etc/fail2ban/jail.local << 'EOF'
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = %(sshd_log)s
maxretry = 3
findtime = 10m
bantime = 1h
EOF

systemctl restart fail2ban
fail2ban-client status sshd
```

### 10.3 内核参数加固

```bash
# /etc/sysctl.d/99-security.conf
cat > /etc/sysctl.d/99-security.conf << 'EOF'
# 防止 SYN Flood
net.ipv4.tcp_syncookies = 1
# 关闭 IP 源路由
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
# 开启反向路径过滤（防 IP 欺骗）
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
# 不响应 ICMP 重定向
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
# 不发送 ICMP 重定向
net.ipv4.conf.all.send_redirects = 0
# 忽略 ICMP 广播请求
net.ipv4.icmp_echo_ignore_broadcasts = 1
# 记录 martian 包（异常源地址包）
net.ipv4.conf.all.log_martians = 1
# 禁止非 GPL 内核模块加载
kernel.modules_disabled = 0
EOF

sysctl -p /etc/sysctl.d/99-security.conf
```

### 10.4 防火墙基线

```bash
# ufw 最小化规则示例
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp comment 'SSH'          # 仅允许你的管理端口
# ufw allow 80/tcp comment 'HTTP'       # 按需开放
# ufw allow 443/tcp comment 'HTTPS'
ufw enable
ufw status verbose
```

### 10.5 定期自查脚本

```bash
cat > /root/security_check.sh << 'EOF'
#!/bin/bash
{
    echo "=== $(date) ==="
    echo "--- Users with UID 0 ---"
    awk -F: '$3==0 {print $1}' /etc/passwd
    echo "--- Recently modified binaries ---"
    find /bin /sbin /usr/bin /usr/sbin -type f -mtime -1 -ls 2>/dev/null
    echo "--- Suspicious crontabs ---"
    grep -rE "curl|wget|/tmp|/dev/shm" /var/spool/cron/crontabs/ /etc/crontab /etc/cron.*/ 2>/dev/null
    echo "--- Listening ports ---"
    ss -tlnp
    echo "--- /tmp executables ---"
    find /tmp -type f -executable 2>/dev/null
    echo "--- Unauthorized SSH keys ---"
    find /home /root -name authorized_keys -exec ls -la {} \; 2>/dev/null
    echo "--- Immutable files (chattr +i) ---"
    lsattr /root/.ssh/ /home/*/.ssh/ 2>/dev/null | grep -E '^.{4}i'
    find /etc/ssh -type f -exec lsattr {} \; 2>/dev/null | grep -E '^.{4}i'
    echo "=== END ==="
} >> /var/log/security_check.log 2>&1
EOF

chmod +x /root/security_check.sh

# 配置每天凌晨 3 点自动执行
(crontab -l 2>/dev/null; echo "0 3 * * * /root/security_check.sh") | crontab -
```

---

## 11. 真实案例：SSH 爆破 → Crypto Miner 全链路

> 以下内容脱敏自生产环境真实入侵事件，攻击者通过 SSH 密码爆破获得 root 权限后部署挖矿木马，并实施多层反侦察手段。

### 11.1 攻击链路总览

```
SSH 端口暴露 → 密码爆破 → root 登录成功
    → 上传 Miner 二进制（~/.configrc7/a/kswapd00、/tmp/.kswapd00）
    → 部署持久化（crontab + @reboot + .ssh/rc）
    → 隐藏痕迹（LD_PRELOAD 劫持 + chattr +i 锁文件 + 伪装内核线程名）
```

### 11.2 第一阶段：爆破登录

攻击者从多个 IP 对 `root` 账号发动密码爆破，历史日志显示连续爆破后多次 `Accepted password for root` 成功：

```bash
# 提取入侵期间所有成功登录的 IP
journalctl -u ssh --no-pager \
  | sed -En 's/.*Accepted password for (\S+) from (\S+).*/\1 \2/p' \
  | sort -u
```

输出看到 `root` 对应了 7 个不同 IP：`58.213.84.194`、`117.89.134.58`、`180.98.144.80`、`180.111.229.146`、`183.209.135.173`、`223.104.104.48`、`1.203.166.134`。

> **教训：** root + 密码 + 22 端口暴露，必然被爆。正确的做法是强制公钥登录、禁用 root 登录、配合 fail2ban。

### 11.3 第二阶段：环境劫持（反侦察）

登录后第一时间感到异常：`ls` 没有颜色、`ll` 命令不存在、`ls` 看不到恶意文件。

**根因：** 攻击者通过 `~/.ssh/rc` 或 `LD_PRELOAD` 劫持了 Shell 环境，使得 `ls` 输出不区分颜色（隐藏文件与普通文件混在一起），且阻断了 `readdir()` 系统调用，使隐藏目录不可见。

**自救方法：**

```bash
# 重新启动干净的 bash（绕过被污染的 rc 文件）
/bin/bash --norc --noprofile
```

执行后 `ls` 恢复颜色，`ll` 可用，隐藏的挖矿文件 `.kswapd00` 立即可见。

### 11.4 第三阶段：挖矿进程伪装

受害服务器上发现的恶意进程：

| 伪装进程名 | 模仿对象 | 真实路径 | 行为 |
|-----------|---------|---------|------|
| `kauditd0` | 内核审计线程 `kauditd` | `~/.configrc7/a/kswapd00` | 占用几十个 CPU 线程，每个 100% |
| `.kswapd00` | 内核交换线程 `kswapd0` | `/tmp/.kswapd00` | 2.2MB ELF 可执行文件，Miner 主程序 |

**排查要点：** 伪装的进程名与内核线程仅差 1-2 个字符（`kswapd0` → `kswapd00`，`kauditd` → `kauditd0`），不仔细看极难发现。

```bash
# 确认真伪：内核线程的 exe 不存在
readlink /proc/$(pgrep kswapd0)/exe   # 无输出 → 真正的内核线程
readlink /proc/$(pgrep kswapd00)/exe  # /tmp/.kswapd00 → 伪装的挖矿程序

# 内核线程的 PID 普遍较小，且括号包裹
ps aux | grep -E '\[.*\]'
```

### 11.5 第四阶段：多层持久化

攻击者部署了三套独立的持久化机制，确保即使清理不彻底也能自动复活：

**1. crontab（6 个恶意条目）：**

```
*/30 * * * * /tmp/.kswapd00                                     # 每 30 分钟执行 Miner
*/30 * * * * /root/.configrc7/a/kswapd00                        # 备用副本
5 6 */2 * 0 /root/.configrc7/a/upd                              # 每两周周日更新
@reboot /root/.configrc7/a/upd                                   # 开机自启
5 8 * * 0 /root/.configrc7/b/sync                               # 每周日同步
@reboot /root/.configrc7/b/sync                                  # 开机自启
0 0 */3 * * /tmp/.X291-unix/.rsync/c/aptitude                   # 每 3 天一次
```

**关键模式：**
- 多个目录存放副本（`/tmp/`、`~/.configrc7/a/`、`~/.configrc7/b/`、`/tmp/.X291-unix/`）
- `@reboot` 保证重启后复活
- 更新器和同步器保证版本迭代

**2. 目录命名伪装：**

| 路径 | 伪装对象 |
|------|---------|
| `~/.configrc7/` | 仿 `~/.configrc`（bash 配置目录） |
| `/tmp/.X291-unix/` | 仿 `/tmp/.X11-unix/`（X Window 合法目录） |

**3. authorized_keys 植入 + 文件锁：**

```bash
# authorized_keys 中被追加的攻击者公钥（comment 为 "mdrfckr"）
cat ~/.ssh/authorized_keys
# ssh-rsa AAAAB3NzaC1... mdrfckr

# 尝试清空 → Operation not permitted
> ~/.ssh/authorized_keys
# bash: authorized_keys: Operation not permitted

# 原因：被 chattr +ia 双锁（immutable + append-only）
lsattr ~/.ssh/authorized_keys
# ----ia-------e-- ~/.ssh/authorized_keys

# 连 .ssh 目录本身也被双锁，导致 known_hosts 无法创建
lsattr -d ~/.ssh
# ----ia-------e-- /root/.ssh
touch ~/.ssh/known_hosts
# touch: setting times of 'known_hosts': No such file or directory
```

**解锁操作（必须 `-ia` 一起解，单独 `-i` 不够）：**

```bash
chattr -ia ~/.ssh/authorized_keys
chattr -ia ~/.ssh/
> ~/.ssh/authorized_keys
```

### 11.6 清理步骤

按以下顺序执行，避免残留：

```bash
# 1. 停止伪装的内核线程
kill -9 $(pgrep -f kswapd00) 2>/dev/null
kill -9 $(pgrep kauditd0) 2>/dev/null

# 2. 清理 crontab 恶意条目（确认最后一行正常脚本后注释或删除）
crontab -e
# 删除或注释所有非预期行，保留 @reboot /bin/bash /root/video-api-pre/startup.sh start

# 3. 解除文件双层锁并清空 hostile key
chattr -ia /root/.ssh/authorized_keys /root/.ssh/
> /root/.ssh/authorized_keys
rm -f /root/.ssh/rc /root/.ssh/environment

# 4. 删除所有恶意文件与目录
rm -rf /root/.configrc7
rm -f /tmp/.kswapd00
rm -rf /tmp/.X291-unix
# 注意：先打包保留证据
tar -czf /root/configrc7.tar.gz /root/.configrc7

# 5. 检查并移除 LD_PRELOAD
cat /etc/ld.so.preload && > /etc/ld.so.preload
unset LD_PRELOAD

# 6. 重建 known_hosts（修复跳板机指纹问题）
touch /root/.ssh/known_hosts

# 7. 立即加固 SSH（禁用密码 + root 登录）
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
systemctl restart sshd
```

### 11.7 未处理时的持续风险

如果仅删除文件但不加固 SSH，攻击者在以下时机可以重新入侵：

1. **crontab 未清理** → Miner 每 30 分钟从 `/tmp/.kswapd00` 复活
2. **authorized_keys 未清空** → 攻击者通过公钥免密登录
3. **SSH 密码登录未禁用** → 爆破继续，二次沦陷
4. **LD_PRELOAD 未清除** → 即使清理了文件，`ls` 和 `top` 仍然被劫持

> **结论：** 手动清理只是临时措施。被 root 权限入侵过的机器，唯一安全的选择是**重装系统**。

