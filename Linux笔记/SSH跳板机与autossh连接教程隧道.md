## SSH 跳板机完全教程：`-J` 参数与 autossh 持久隧道

先用一张图说清楚整体架构，再逐层讲解。---

### 一、前置条件

三台机器的角色分工：

- **hostA**：你的本地机器，发起 SSH 连接
- **hostB**：跳板机（bastion），公网可访问，端口通常为 22 或自定义
- **hostC**：目标机器，只在私有网络内可达，hostA 无法直连

确保 hostA 的公钥已分别添加到 hostB 和 hostC 的 `~/.ssh/authorized_keys`。

---

### 二、`ssh -J` 一键跳板（推荐）

#### 基础用法

```bash
ssh -J hostB hostC
```

这一条命令做了两件事：先 SSH 连到 hostB，再从 hostB 内部 SSH 连到 hostC。整个过程端对端加密，hostB 看不到明文内容。

#### 自定义端口与用户

```bash
# hostB 用户 admin、端口 2222；hostC 用户 ubuntu、端口 22
ssh -J admin@hostB:2222 ubuntu@hostC
```

#### 多级跳板（跳两次）

```bash
# hostA → hostB → hostD → hostC
ssh -J hostB,hostD hostC
```

#### 写进 `~/.ssh/config`（强烈推荐）

每次手打参数太繁琐，配置文件一次写好，之后只需 `ssh hostC`：

```
Host hostB
    HostName 1.2.3.4          # hostB 的公网 IP
    User     youruser
    Port     22
    IdentityFile ~/.ssh/id_ed25519

Host hostC
    HostName 192.168.1.100    # hostC 的内网 IP
    User     ubuntu
    ProxyJump hostB            # 核心：声明跳板
    IdentityFile ~/.ssh/id_ed25519
```

配置完成后：

```bash
ssh hostC          # 自动经 hostB 跳板
scp file hostC:~   # scp / rsync 同样生效
```

---

### 三、端口转发进阶

#### 本地端口转发（把 hostC 的服务拉到本地）

```bash
# 把 hostC:8080 映射到本地 localhost:8080，通过 hostB 跳板
ssh -J hostB -L 8080:hostC:8080 hostC -N
```

`-N` 表示不执行远程命令，只维持隧道。之后浏览器访问 `http://localhost:8080` 即可。

#### 动态 SOCKS5 代理

```bash
# 在本地 1080 端口起一个 SOCKS5 代理，流量经 hostB 出去
ssh -J hostB -D 1080 hostC -N
```

---

### 四、autossh：让隧道永不断线

`ssh -N` 断线后不会自动重连，生产环境要用 autossh 守护隧道。

#### 安装

```bash
# macOS
brew install autossh

# Ubuntu / Debian
sudo apt install autossh
```

#### 基本用法

```bash
autossh -M 0 \
  -o "ServerAliveInterval 30" \
  -o "ServerAliveCountMax 3" \
  -o "ExitOnForwardFailure yes" \
  -J hostB \
  -L 8080:hostC:8080 \
  hostC -N
```

参数说明：

|参数|含义|
|---|---|
|`-M 0`|关闭 autossh 自带的监控端口，改用 SSH 心跳检测|
|`ServerAliveInterval 30`|每 30 秒发一次心跳|
|`ServerAliveCountMax 3`|连续 3 次无响应则断开并重连|
|`ExitOnForwardFailure yes`|端口转发失败时立即退出让 autossh 重启|

#### 配置为 systemd 服务（开机自启）

创建 `/etc/systemd/system/ssh-tunnel-hostC.service`：

```ini
[Unit]
Description=SSH tunnel to hostC via hostB
After=network.target
Wants=network-online.target

[Service]
User=youruser
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -M 0 \
    -o "ServerAliveInterval=30" \
    -o "ServerAliveCountMax=3" \
    -o "ExitOnForwardFailure=yes" \
    -o "StrictHostKeyChecking=accept-new" \
    -J hostB \
    -L 8080:hostC:8080 \
    hostC -N -T

Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

启用并启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable ssh-tunnel-hostC
sudo systemctl start  ssh-tunnel-hostC
sudo systemctl status ssh-tunnel-hostC
```

---

### 五、常见问题排查

**连接超时**：先单独测试 `ssh hostB` 能否成功，再测 `ssh -J hostB hostC`，缩小问题范围。

**"Permission denied"**：检查 hostC 的 `~/.ssh/authorized_keys` 是否包含 hostA 的公钥（不是 hostB 的）。用 `-v` 参数看详细握手日志：

```bash
ssh -vJ hostB hostC
```

**端口被占用**（`bind: Address already in use`）：先 `lsof -i :8080` 找到占用进程，或换一个本地端口号。

**hostB 禁止 TCP 转发**：联系管理员检查 hostB 的 `/etc/ssh/sshd_config` 中 `AllowTcpForwarding` 是否为 `yes`。

---

### 六、安全加固建议

在 hostB 的 `sshd_config` 中限制跳板机权限，防止被滥用：

```
# 只允许转发，禁止登录执行命令
Match User jumpuser
    ForceCommand /usr/sbin/nologin
    AllowTcpForwarding yes
    X11Forwarding no
    PermitTTY no
```

同时给跳板机账号设置专用的受限密钥，在 `authorized_keys` 中加前缀限制来源 IP：

```
from="1.2.3.4",no-pty,no-X11-forwarding ssh-ed25519 AAAA...
```