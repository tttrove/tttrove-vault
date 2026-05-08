# SSH 日志审计：提取密码登录用户名与源IP

> 适用场景：服务器被 SSH 爆破后，需要追溯有哪些用户名通过密码登录成功，以及来自哪些 IP  
> 日志来源：`journalctl -u ssh` 或 `/var/log/auth.log`  
> 核心工具：`sed -E`、`grep -E`、`awk`、`sort -u`

---

## 正则模式速查

三种正则模式差异显著，本文一律使用 **ERE 扩展正则**：

| 模式 | 全称 | 工具示例 | 捕获组写法 | `|` 分支 | `+` 量词 | 推荐程度 |
|------|------|---------|-----------|---------|---------|---------|
| BRE | Basic RegEx | `grep`（默认）、`sed`（默认） | `\(...\)` 要反斜杠 | `\|` | `\+` | 避免使用，反直觉 |
| **ERE** | **Extended RegEx** | **`grep -E`、`sed -E`、`awk`** | **`(...)` 无需转义** | **`\|`** | **`+`** | **优先使用** |
| PCRE | Perl Compatible | `grep -P`（GNU 专属） | `(...)` | `\|` | `+` | 仅 GNU，非 POSIX |

> **原则：能用 ERE 就不用 BRE/PCRE。** `sed -E` 和 `grep -E` 在所有 Linux 发行版均可使用，且语法最直观。

---

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

## 方法一：sed -E 扩展正则（推荐）

ERE 语法，捕获组 `( )` 无需反斜杠转义，`\S+` 匹配连续非空白字符：

```bash
journalctl -u ssh --no-pager \
  | sed -En 's/.*Accepted password for (\S+) from (\S+).*/\1 \2/p' \
  | sort -u
```

**拆解：**

- `sed -En` — `-E` 启用扩展正则，`-n` 抑制默认输出，末尾 `p` 仅打印匹配行
- `(\S+)` — ERE 捕获组，匹配连续非空白字符（用户名）
- `(\S+)` — 第二个捕获组，匹配 IP（`from` 后的第一个非空白字段）
- `\1 \2` — 替换为「用户名 空格 IP」格式
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

### 对比：BRE 写法（不推荐，仅作对照）

```bash
# 同样是 sed，默认 BRE 模式需要把括号全加上反斜杠
sed -n 's/.*Accepted password for \([^ ]*\) from \([^ ]*\).*/\1 \2/p'
#                            ^^   ^  ^        ^^   ^  ^
#         BRE 的捕获组必须写成 \( 和 \)  ← 反直觉，容易写错
```

---

## 方法二：grep -oE + awk

用 `grep -E` 提取匹配段，再用 `awk` 取列：

```bash
journalctl -u ssh --no-pager \
  | grep 'Accepted password' \
  | grep -oE 'Accepted password for [^ ]+ from [^ ]+' \
  | awk '{print $4,$6}' \
  | sort -u
```

**拆解：**

- `grep -oE` — `-o` 只输出匹配段，`-E` 启用扩展正则
- `[^ ]+` — ERE 写法的非空白字符匹配（等价于 PCRE 的 `\S+`）
- `awk '{print $4,$6}'` — 输出第 4（用户名）和第 6（IP）列

---

## 方法三：纯 awk

无需 `grep`，POSIX 兼容性最好：

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

---

## 方法四：读取旧日志文件

如果 `journalctl` 只保留近期日志，可以从 `/var/log/auth.log*` 读取历史记录：

```bash
zgrep 'Accepted password' /var/log/auth.log* \
  | sed -En 's/.*Accepted password for (\S+) from (\S+).*/\1 \2/p' \
  | sort -u
```

`zgrep` 可以同时处理 `.log` 和 `.log.gz` 文件。

---

## 方法对比

| 方法 | 工具 | 正则模式 | 优点 | 缺点 |
|------|------|---------|------|------|
| sed -E 捕获组 | `sed -E` | **ERE** | 一条命令，语法简洁直观 | 无 |
| grep -oE + awk | `grep -E` + `awk` | ERE | POSIX 兼容，分步清晰 | 多一次管道 |
| 纯 awk | `awk` | —（字段匹配） | POSIX 兼容，可移植性最好 | 代码稍长 |

**不使用 PCRE（`grep -P`）的原因：** `-P` 是 GNU grep 专属扩展，macOS/BSD 不兼容，且 ERE 完全够用，无需引入 Perl 语法。

生产环境推荐**方法一（sed -E）**，它用 ERE 扩展正则、一行完成、所有 Linux 发行版均可用。

---

## 扩展：按用户分组统计 IP

在去重基础上，可以进一步统计每个用户名关联了多少独立 IP：

```bash
journalctl -u ssh --no-pager \
  | sed -En 's/.*Accepted password for (\S+) from (\S+).*/\1 \2/p' \
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
