你是一名 Linux 安全运维专家。请以「Linux 服务器入侵排查手册」为主题，生成一份结构完整的 Markdown 笔记。笔记采用教程风格，实操导向，每一项排查手段都附带可直接执行的命令与解读。

要求覆盖以下主线，每条主线作为一个 H2 章节：

## 1. 入侵迹象识别
- CPU 异常飙高、陌生进程、网络流量异常、crontab 被篡改、日志被清空等典型表象
- 给出排查命令和正常基线对照

## 2. 登录审计
- `last` / `lastb` / `lastlog` 核查登录记录
- `who` / `w` 查看当前在线用户
- 优先从 `journalctl` 提取 SSH 日志（journald 日志更难被篡改，而 `/var/log/auth.log` 易被反侦察清除）
- 从 journalctl 提取密码登录成功的用户名与源 IP 并去重

## 3. 账号与权限审计
- `/etc/passwd` 中非预期用户、UID 0 的隐藏超级用户
- `/etc/shadow` 中空密码、无密码账户
- `sudoers` 及 `/etc/sudoers.d/` 中异常授权
- SSH authorized_keys 是否被追加未知公钥

## 4. 进程与网络排查
- `ps auxf` 查进程树，识别伪装成系统进程的恶意程序
- `ls -la /proc/*/exe` 找出被删除但仍在运行的可执行文件
- `netstat -antp` / `ss -antp` 查异常外连和监听端口
- `/proc/*/cwd` 和 `/proc/*/cmdline` 追溯进程起源

## 5. 持久化机制排查
- `/etc/crontab`、`/var/spool/cron/crontabs/`、`/etc/cron.*/` 下的异常任务
- `systemctl list-timers` 和 `/etc/systemd/system/` 下的恶意 service/timer
- `/etc/rc.local`、`/etc/init.d/`、`~/.bashrc` / `~/.profile` 中的后门启动项
- LD_PRELOAD、`/etc/ld.so.preload` 劫持

## 6. 文件与隐藏目录
- `find` 查找近期修改的脚本、可执行文件（`-mtime`、`-perm` 等）
- 以 `.` 或 `.. `（点+空格）开头的伪隐藏目录
- `/tmp`、`/dev/shm`、`/var/tmp` 中的可疑文件
- SUID/SGID 异常文件审计

## 7. Rootkit 检测
- `rkhunter` 和 `chkrootkit` 的安装与使用
- 比较 `/proc` 和 `ps` 输出是否一致（Diamond 式 rootkit 检测）
- `debsums` 校验系统二进制完整性（Debian/Ubuntu 系）

## 8. 日志审计与异常
- `/var/log/syslog`、`/var/log/auth.log`、`/var/log/kern.log` 的时间空洞
- `history` 文件是否被篡改、`HISTFILE` / `HISTSIZE` 变量是否被修改
- `journalctl --verify` 校验日志完整性

## 9. 应急处置与溯源优先级
- 处置决策树：断网保留现场 vs 隔离 vs 关机封存
- 取证优先顺序（volatile → non-volatile 数据）
- 与攻击者共存时的隐蔽排查策略

## 10. 加固检查清单
- SSH 相关：禁用密码登录、禁用 root 登录、禁用弱加密算法、更换端口、fail2ban
- 内核参数：`net.ipv4.tcp_syncookies`、`net.ipv4.conf.all.rp_filter` 等
- 防火墙基线：`iptables` / `nftables` / `ufw` 最小化规则
- 定期审计：`lynis` 或自建 cron 巡检脚本

---

格式要求：
- 使用 H1 主标题「Linux 服务器入侵排查手册」
- 每个 H2 章节内，按「排查目的 → 命令 → 正常输出 vs 可疑输出对照 → 解读」的结构展开
- 所有命令放在 \`\`\`bash 代码块内
- 重要字段用**加粗**标注
- 关键工具或配置文件用反引号标注路径
- 禁止添加 YAML frontmatter
- 禁止添加「参考」「延伸阅读」等凑数章节，只写实操内容
- 全文中文叙述，技术术语保留英文原文
- 命令行中的正则表达式**必须使用 ERE 扩展正则**（`sed -E`、`grep -E`、`awk`），
  禁止使用 BRE 基础正则（sed 默认模式的反斜杠转义括号 `\(`、`\)`、`\|`）和 PCRE（`grep -P`），
  ERE 语法最直观（`(...)` 无需转义、`|` 直接使用），且在所有 Linux 发行版均可使用
