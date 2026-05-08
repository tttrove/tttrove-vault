# SSH 日志审计：提取密码登录用户名与源IP

> 适用场景：服务器被 SSH 爆破后，需要追溯有哪些用户名通过密码登录成功，以及来自哪些 IP  
> 日志来源：`journalctl -u ssh` 或 `/var/log/auth.log`  
> 核心工具：`sed`、`grep`、`awk`、`sort -u`

## 日志格式

SSH 密码登录成功的日志行格式如下：

```
May 01 11:42:40 ubuntu-lwzx-154 sshd[2249288]: Accepted password for root from 58.213.84.194 port 8997 ssh2
```

需要提取的两个字段：

| 字段 | 位置 | 示例值 |
|------|------|--------|
| 用户名 | `Accepted password for` 之后 | `root` |
| 源 IP | `from` 之后 | `58.213.84.194` |

---

## 方法

### 方法一：sed 捕获组（推荐）

最简洁，一行搞定，无需管道串联多个命令：

```bash
journalctl -u ssh --no-pager \
  | sed -n 's/.*Accepted password for \([^ ]*\) from \([^ ]*\).*/\1 \2/p' \
  | sort -u
```

**拆解：**

- `sed -n` — 默认不打印，只有匹配成功才输出（靠末尾的 `p` 标志）
- `\([^ ]*\)` — 捕获组，匹配连续非空格字符，即用户名
- `\([^ ]*\)` — 第二个捕获组，匹配 IP（`from` 后的第一个非空格字段）
- `\1 \2` — 替换为「用户名 空格 IP」格式输出
- `sort -u` — 排序并去重

输出示例：

```
root 1.203.166.134
root 117.89.134.58
root 180.111.229.146
root 180.98.144.80
root 183.209.135.173
root 223.104.104.48
root 58.213.84.194
```

### 方法二：grep -oP + awk

适合对 PCRE 正则更熟悉的场景：

```bash
journalctl -u ssh --no-pager \
  | grep 'Accepted password' \
  | grep -oP 'Accepted password for \S+ from \S+' \
  | awk '{print $4,$6}' \
  | sort -u
```

**拆解：**

- `grep -oP` — `-o` 只输出匹配部分，`-P` 启用 PCRE 正则
- `\S+` — 匹配任意非空白字符
- `awk '{print $4,$6}'` — 输出第 4（用户名）和第 6（IP）列

### 方法三：纯 awk

无需 GNU grep 的 `-P` 选项，兼容性更好：

```bash
journalctl -u ssh --no-pager \
  | grep 'Accepted password' \
  | awk '{
      for(i=1;i<=NF;i++){
          if($i=="for") u=$(i+1);
          if($i=="from"){print u,$(i+1); break}
      }
  }' \
  | sort -u
```

**拆解：**

- 遍历行中每个字段（`for` 循环）
- 遇到 `for` → 记录下一个字段为用户名 `u`
- 遇到 `from` → 输出 `u` 和下一个字段（IP），然后 `break` 结束该行处理

### 方法四：读取旧日志文件

如果 `journalctl` 只保留近期日志，可以从 `/var/log/auth.log*` 读取历史记录：

```bash
zgrep 'Accepted password' /var/log/auth.log* \
  | sed -n 's/.*Accepted password for \([^ ]*\) from \([^ ]*\).*/\1 \2/p' \
  | sort -u
```

`zgrep` 可以同时处理 `.log` 和 `.log.gz` 文件。

---

## 方法对比

| 方法 | 工具 | 优点 | 缺点 |
|------|------|------|------|
| sed 捕获组 | `sed` | 一条命令，简洁高效 | 需要理解 sed 替换语法 |
| grep -oP + awk | `grep` + `awk` | 正则直观 | 需要 GNU grep（`-P` 非 POSIX） |
| 纯 awk | `awk` | POSIX 兼容，可移植性最好 | 代码稍长 |

生产环境中推荐**方法一（sed）**，日常排查时**方法二（grep -oP）**更直观。

---

## 扩展：按用户分组统计 IP

在去重基础上，可以进一步统计每个用户名关联了多少独立 IP：

```bash
journalctl -u ssh --no-pager \
  | sed -n 's/.*Accepted password for \([^ ]*\) from \([^ ]*\).*/\1 \2/p' \
  | sort -u \
  | awk '{cnt[$1]++} END {for(u in cnt) print cnt[u], u}' \
  | sort -rn
```

输出：

```
7 root
3 ubuntu
1 www-data
```

由此可以快速判断哪些用户被成功登录的次数最多，优先排查该用户的会话记录。

---

## 排查思路总结

当怀疑服务器被 SSH 爆破后，按以下顺序排查：

1. **提取密码登录成功记录** → 确认来源 IP 和用户名（本文方法）
2. **检查 authorized_keys** → `cat ~/.ssh/authorized_keys` 有无被追加未知公钥
3. **检查 last 和 wtmp** → `last -20` 或 `lastlog` 查看登录历史
4. **审计当前进程** → `ps auxf` 查可疑进程，`netstat -antp` 查异常连接
5. **检查 crontab 和 systemd timer** → `/var/spool/cron/crontabs/`、`systemctl list-timers` 查持久化
6. **加固 SSH**：
   - 禁用密码登录 → `PasswordAuthentication no`
   - 禁用 root 登录 → `PermitRootLogin no`
   - 更换端口 → 非必要不暴露，推荐配合 `fail2ban` 使用
   - 配置 `fail2ban` → 自动封禁爆破 IP（另见《Ubuntu fail2ban 安装与配置指南》）

---

## 参考

- `man sshd_config` — SSH 服务端配置手册
- `man journalctl` — systemd 日志查询工具
- `man sed` — 流编辑器
- 《Ubuntu fail2ban 安装与配置指南》— 同目录
