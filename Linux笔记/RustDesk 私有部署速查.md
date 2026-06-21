---
title: RustDesk 私有部署速查：Docker + Nginx + API 三件套
tags: [RustDesk, Docker, Nginx, 腾讯云, 自托管, 远程桌面]
created: 2026-06-21
---

# RustDesk 私有部署速查：Docker + Nginx + API 三件套

在远程访问多台电脑时，RustDesk OSS 虽然开源免费，但缺少设备管理界面，只能记忆数字设备 ID（如 `1426193003`）。而 RustDesk Pro 需要付费，TeamViewer 又依赖国外 SaaS。

**本文提供 RustDesk OSS + 社区 API 控制台的快速部署方案**，30 分钟内跑通：Web 管理界面 + 设备别名 + HTTPS + 地址簿同步，实现 TeamViewer 体验的自托管版本。

核心优势：

- **设备管理**：Web 控制台查看设备列表、在线状态、分组管理
- **别名代替 ID**：用 `HOME-PC` / `DELL-LAPTOP` 代替数字 ID
- **地址簿同步**：客户端登录后自动同步，跨设备共享
- **HTTPS 安全**：Let's Encrypt 证书自动续期
- **审计日志**：登录、连接、文件传输全记录

本文将从架构、DNS、部署、客户端配置、故障排查等方面，沉淀这次实战的完整知识。

---

## 目录

- [1. 核心架构](#1-核心架构)
- [2. DNS 与安全组](#2-dns-与安全组)
  - [2.1 DNS 记录](#21-dns-记录)
  - [2.2 腾讯云安全组](#22-腾讯云安全组)
- [3. 目录结构](#3-目录结构)
- [4. Docker Compose 部署](#4-docker-compose-部署)
- [5. Nginx + HTTPS](#5-nginx--https)
  - [5.1 安装 nginx.org 官方版 + certbot DNS 插件](#51-安装-nginxorg-官方版--certbot-dns-插件)
  - [5.2 创建 Cloudflare API Token](#52-创建-cloudflare-api-token)
  - [5.3 签发 Let's Encrypt 证书（DNS 验证）](#53-签发-lets-encrypt-证书dns-验证)
  - [5.4 Nginx 反向代理配置（仅 443，不监听 80）](#54-nginx-反向代理配置仅-443不监听-80)
  - [5.5 腾讯云安全组关闭 80 端口](#55-腾讯云安全组关闭-80-端口)
- [6. Windows 客户端配置](#6-windows-客户端配置)
  - [6.1 GUI 手动配置](#61-gui-手动配置)
  - [6.2 PowerShell CLI 批量配置](#62-powershell-cli-批量配置)
  - [6.3 验证配置](#63-验证配置)
- [7. 控制台使用](#7-控制台使用)
- [8. 故障速查表](#8-故障速查表)
- [9. 长期维护](#9-长期维护)
- [10. 总结](#10-总结)
- [11. 参考资料](#11-参考资料)

---

## 1. 核心架构

```
公网用户 (IPv4/IPv6)
    ↓
腾讯云 Ubuntu 24.04（非 root 用户 ubuntu）
├─ Nginx 1.30 (80/443 → 21114)
│   └─ Let's Encrypt 证书（自动续期）
└─ Docker: lejianwen/rustdesk-server-s6
    ├─ hbbs + hbbr (21115-21119)
    └─ API + Web Admin (21114)
```

**组件清单**：

- **服务端**：`lejianwen/rustdesk-server-s6`（fork 增强版，单容器含 hbbs + hbbr + API + Web Admin）
- **反向代理**：nginx.org 官方版 + certbot
- **域名策略**：`rd.tttrove.qzz.io` 专给 RustDesk，`linux.tttrove.qzz.io` 留给 SSH
- **部署用户**：`ubuntu`（非 root，sudo 提权）
- **部署路径**：`/home/ubuntu/Apps/rustdesk`

---

## 2. DNS 与安全组

### 2.1 DNS 记录

在腾讯云 DNS 解析中添加两条记录：

| 主机记录 | 记录类型 | 记录值 | TTL |
|---------|---------|-------|-----|
| `rd` | A | 服务器 IPv4 | 600 |
| `rd` | AAAA | 服务器 IPv6 | 600 |

> **提示**：双栈记录确保 IPv4-only 和 IPv6-only 客户端都能访问。

### 2.2 腾讯云安全组

入站规则需放行以下端口（TCP4/TCP6 + UDP4/UDP6）：

| 端口 | 协议 | 用途 |
|------|------|------|
| ~~80~~ | ~~TCP4/TCP6~~ | ~~HTTP 跳转 + 证书续期~~ **可关闭**（证书续期走 Cloudflare DNS 验证，不依赖 80） |
| 443 | TCP4/TCP6 | Web Admin / API HTTPS |
| 21115 | TCP4/TCP6 | hbbs NAT 测试 |
| 21116 | TCP4/TCP6 + UDP4/UDP6 | hbbs ID 注册（UDP 必须放行） |
| 21117 | TCP4/TCP6 | hbbr 中继 |
| 21118 | TCP4/TCP6 | hbbs WebSocket |
| 21119 | TCP4/TCP6 | hbbr WebSocket |

> **重要**：`21116` 必须同时放行 TCP 和 UDP，否则客户端无法注册 ID。`21114` 不需要对公网开放（仅 Nginx 本机反代）。

---

## 3. 目录结构

```
/home/ubuntu/Apps/rustdesk
├── api-data/
│   └── rustdeskapi.db          # Web 控制台数据库（用户、设备、地址簿）
├── compose.yml                 # Docker Compose 配置
├── compose.yml.bak.*           # 配置备份
└── data/
    ├── db_v2.sqlite3           # hbbs 设备注册数据库
    ├── db_v2.sqlite3-shm
    ├── db_v2.sqlite3-wal
    ├── id_ed25519              # 服务端身份私钥（核心机密）
    └── id_ed25519.pub          # 服务端公钥（客户端 Key 来源）
```

**关键文件说明**：

- `data/id_ed25519`：RustDesk 服务端身份私钥，**决定服务器身份**，泄露后他人可伪造你的服务器
- `data/id_ed25519.pub`：公钥，客户端配置中的 `Key` 字段填此内容
- `api-data/rustdeskapi.db`：Web 控制台 SQLite 数据库，包含用户、设备、地址簿、审计日志

---

## 4. Docker Compose 部署

### 4.1 备份旧部署（如已部署 RustDesk）

```bash
# 保留 id_ed25519 密钥对（客户端 Key 不变，无需重新配置）
sudo cp -a /home/ubuntu/Apps/rustdesk/data /home/ubuntu/rustdesk-backup-$(date +%Y%m%d)
```

### 4.2 创建 compose.yml

```yaml
# /home/ubuntu/Apps/rustdesk/compose.yml
services:
  rustdesk:
    container_name: rustdesk
    image: lejianwen/rustdesk-server-s6:latest
    environment:
      - RELAY=rd.tttrove.qzz.io
      - ENCRYPTED_ONLY=0              # 兼容模式，生产环境改 1
      - MUST_LOGIN=N                  # 客户端全配好后改 Y
      - TZ=Asia/Shanghai
      - RUSTDESK_API_LANG=zh-CN
      - RUSTDESK_API_APP_REGISTER=false   # 禁止公开注册
      - RUSTDESK_API_APP_SHOW_SWAGGER=0   # 隐藏 API 文档
      - RUSTDESK_API_RUSTDESK_ID_SERVER=rd.tttrove.qzz.io
      - RUSTDESK_API_RUSTDESK_RELAY_SERVER=rd.tttrove.qzz.io
      - RUSTDESK_API_RUSTDESK_API_SERVER=https://rd.tttrove.qzz.io
      - RUSTDESK_API_RUSTDESK_KEY_FILE=/data/id_ed25519.pub
      - RUSTDESK_API_JWT_KEY=${RUSTDESK_API_JWT_KEY}
    volumes:
      - ./data:/data           # 旧密钥挂载此处
      - ./api-data:/app/data
    network_mode: "host"
    restart: unless-stopped
```

### 4.3 创建 .env 文件

```bash
# 生成随机 JWT 密钥
echo "RUSTDESK_API_JWT_KEY=$(openssl rand -hex 32)" > /home/ubuntu/Apps/rustdesk/.env
sudo chmod 600 /home/ubuntu/Apps/rustdesk/.env
```

### 4.4 创建数据目录并启动

```bash
cd /home/ubuntu/Apps/rustdesk
sudo mkdir -p data api-data
sudo docker compose up -d
```

### 4.5 获取初始 admin 密码

```bash
sudo docker logs rustdesk 2>&1 | grep "Admin Password"
# 输出示例：Admin Password Is: xxxxxx
```

**立即登录改密**：`https://rd.tttrove.qzz.io/_admin/`

### 4.6 核心环境变量说明

| 变量 | 作用 | 调整建议 |
|------|------|---------|
| `ENCRYPTED_ONLY` | `0`=兼容模式 `1`=强制加密 | 测试用 `0`，生产用 `1` |
| `MUST_LOGIN` | `N`=可匿名连接 `Y`=强制登录 | **客户端全配好后再改 `Y`** |
| `RUSTDESK_API_LANG` | 控制台语言 | `zh-CN` / `en` |
| `RUSTDESK_API_APP_REGISTER` | 是否允许公开注册 | 建议 `false` |
| `RUSTDESK_API_JWT_KEY` | API JWT 签名密钥 | 随机 32 字节，设定后不可随意改 |

> **警告**：`MUST_LOGIN=Y` 和 `ENCRYPTED_ONLY=1` 不要在部署初期同时启用，会导致所有未配置 API Server 的客户端无法连接。

---

## 5. Nginx + HTTPS

### 5.1 安装 nginx.org 官方版 + certbot DNS 插件

```bash
# 添加 nginx.org 官方仓库
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] https://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" | sudo tee /etc/apt/sources.list.d/nginx.list

# 安装 nginx + certbot + Cloudflare DNS 插件
sudo apt update && sudo apt install -y nginx certbot python3-certbot-dns-cloudflare

# 启动
sudo systemctl enable --now nginx
```

> **为什么用 Cloudflare DNS 验证而非 webroot**：webroot 模式必须开放 80 端口给公网，证书续期时 Let's Encrypt 会访问 `http://域名/.well-known/acme-challenge/`。DNS 验证通过 Cloudflare API 写 TXT 记录验证，**完全不需要 80 端口**，可彻底关闭 80 端口减少攻击面。

### 5.2 创建 Cloudflare API Token

1. 登录 Cloudflare → 右上角头像 → **My Profile** → **API Tokens**
2. 点 **Create Token** → 选模板 **"Edit zone DNS"**
3. 权限：`Zone` → `DNS` → `Edit`（模板默认）
4. Zone Resources：`Include` → `Specific zone` → `tttrove.qzz.io`
5. 创建并复制 Token（只显示一次）

写入凭证文件（权限 600 仅 root 可读）：

```bash
sudo tee /etc/letsencrypt/cloudflare.ini > /dev/null << 'EOF'
dns_cloudflare_api_token = 你的Cloudflare_API_Token
EOF
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
sudo chown root:root /etc/letsencrypt/cloudflare.ini
```

### 5.3 签发 Let's Encrypt 证书（DNS 验证）

```bash
# 签发证书（Cloudflare DNS 验证，不依赖 80 端口）
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d rd.tttrove.qzz.io \
  --non-interactive \
  --agree-tos \
  -m admin@yourdomain.com

# 验证
sudo certbot certificates

# 测试自动续期（注意加 --no-random-sleep-on-renew 避免随机延迟）
sudo certbot renew --dry-run --no-random-sleep-on-renew
```

> **提示**：certbot 已自动注册 systemd timer，证书到期前 30 天自动续期。DNS 验证续期无需 80 端口，可在安全组彻底关闭 80。

### 5.4 Nginx 反向代理配置（仅 443，不监听 80）

```nginx
# /etc/nginx/conf.d/rustdesk.conf
# 证书续期走 Cloudflare DNS 验证，不依赖 80 端口

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

# 443 端口：HTTPS 反代到容器 21114
server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    http2 on;
    server_name rd.tttrove.qzz.io;

    # 非 rd 域名访问 443 时拒绝（证书不匹配，浏览器自动拦截）
    if ($host != rd.tttrove.qzz.io) {
        return 444;
    }

    ssl_certificate     /etc/letsencrypt/live/rd.tttrove.qzz.io/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/rd.tttrove.qzz.io/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=31536000" always;

    client_max_body_size 0;

    location / {
        proxy_pass http://127.0.0.1:21114;
        proxy_http_version 1.1;

        # WebSocket 支持（在线状态 / Web Client 需要）
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 长连接超时
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_connect_timeout 60s;
    }
}
```

```bash
# 测试并重载
sudo nginx -t && sudo systemctl reload nginx
```

**删除残留默认配置（避免 80 端口被默认 server 监听）**：

```bash
sudo rm -f /etc/nginx/conf.d/default.conf
sudo rm -f /etc/nginx/sites-available/default
sudo rm -f /etc/nginx/sites-enabled/default

# 重启 nginx 确保老 worker 退出（reload 不会立即停老 worker）
sudo systemctl restart nginx

# 验证 80 端口已无监听
sudo ss -tlnp | grep ":80 " || echo "✓ 80 端口已无监听"
```

### 5.5 腾讯云安全组关闭 80 端口

证书续期已切换为 Cloudflare DNS 验证，**80 端口不再有任何业务依赖**。去腾讯云控制台安全组删除 80 端口的入站规则（TCP4/TCP6 都删）。

> **验证清单**：关 80 后依次确认
> - `https://rd.tttrove.qzz.io/_admin/` 仍可访问（走 443）
> - RustDesk 客户端设备仍在线（走 21116 UDP）
> - `sudo certbot renew --dry-run` 成功（走 Cloudflare API）

---

## 6. Windows 客户端配置

### 6.1 GUI 手动配置

RustDesk → 设置 → 网络 → ID/中继服务器：

| 字段 | 值 |
|------|------|
| ID 服务器 | `rd.tttrove.qzz.io` |
| 中继服务器 | `rd.tttrove.qzz.io` |
| API 服务器 | `https://rd.tttrove.qzz.io` |
| Key | `YOUR_SERVER_PUBLIC_KEY_HERE` |

> **提示**：填完点应用，**彻底退出 RustDesk 托盘图标再重启**（右键托盘 → Quit → 重新打开）。

### 6.2 PowerShell CLI 批量配置

```powershell
# 必须以管理员身份运行
$rd = "C:\Program Files\RustDesk\rustdesk.exe"
& $rd --option custom-rendezvous-server "rd.tttrove.qzz.io"
& $rd --option relay-server "rd.tttrove.qzz.io"
& $rd --option api-server "https://rd.tttrove.qzz.io"
& $rd --option key "YOUR_SERVER_PUBLIC_KEY_HERE"
```

> **注意**：`--option` 命令执行后**无回显**（GUI 子系统程序 stdout 不输出到 PowerShell）。验证需读配置文件。

### 6.3 验证配置

```powershell
# 读取配置文件确认
Get-Content "$env:APPDATA\RustDesk\config\RustDesk2.toml"
```

检查 `[options]` 段是否包含：

```toml
[options]
custom-rendezvous-server = 'rd.tttrove.qzz.io'
key = 'YOUR_SERVER_PUBLIC_KEY_HERE'
api-server = 'https://rd.tttrove.qzz.io'
relay-server = 'rd.tttrove.qzz.io'
```

### 6.4 客户端登录账号

打开 RustDesk → 右上角 ≡ → **设置 → 账户 → 登录**

输入 Web 控制台的账号密码（默认 `admin` + 初始密码）。

登录后设备会自动注册到 `peers` 表，`user_id` 绑定为当前登录用户。

---

## 7. 控制台使用

### 7.1 Web 控制台地址

```
https://rd.tttrove.qzz.io/_admin/
```

### 7.2 设备管理与别名设置

Web 控制台 → **设备管理** → 编辑设备 `alias` 字段：

| 原主机名 | 建议别名 |
|---------|---------|
| `msi-b660m` | `HOME-PC` |
| `dell-vostro` | `DELL-LAPTOP` |

改完后，所有登录 admin 的客户端，其地址簿会**自动同步别名**，从此不用记数字 ID。

### 7.3 设备分组

Web 控制台 → 设备管理 → 群组管理：

- 创建群组：`家用` / `公司`
- 将设备分配到群组
- 客户端地址簿会同步分组结构

### 7.4 Web Client 浏览器远控

无需安装客户端，直接浏览器访问：

```
https://rd.tttrove.qzz.io/_webclient/
```

输入目标设备 ID 和密码即可远控（适合临时访问场景）。

### 7.5 审计日志

控制台提供三类日志：

- **登录日志**：记录所有账号登录（IP、设备、时间）
- **连接日志**：记录所有远程连接会话
- **文件传输日志**：记录文件传输操作

---

## 8. 故障速查表

| 现象 | 原因 | 解决 |
|------|------|------|
| 客户端显示离线 / 无法连接 | `MUST_LOGIN=Y` 或 `ENCRYPTED_ONLY=1` 过早启用 | 先设 `MUST_LOGIN=N`、`ENCRYPTED_ONLY=0`，等客户端全配好再加固 |
| 客户端账号登录超时 `TimedOut` | 腾讯云安全组未放行 IPv4 443 端口 | 在腾讯云控制台安全组入站规则放行 TCP4 443 |
| `curl -4 https://IP:443` 返回 `000` | Nginx `server_name` 仅匹配 `rd.*` 域名，IP 直接访问被拒 | 用 `curl --resolve rd.tttrove.qzz.io:443:<IP>` 按域名测试 |

### 关键排查命令

```bash
# 查容器状态与日志
sudo docker ps
sudo docker logs rustdesk --tail 50

# 查端口监听
sudo ss -tlnp | grep -E ":(80|443|21114|21116) "

# 抓包确认客户端请求是否到达
sudo tcpdump -i any -n "udp port 21116" -c 20

# 查设备注册情况
sudo sqlite3 /home/ubuntu/Apps/rustdesk/api-data/rustdeskapi.db \
  "SELECT id,hostname,user_id,alias FROM peers;"
```

---

## 9. 长期维护

### 9.1 查看容器状态

```bash
cd /home/ubuntu/Apps/rustdesk
sudo docker compose ps
sudo docker logs rustdesk --tail 50
```

### 9.2 更新容器镜像

```bash
cd /home/ubuntu/Apps/rustdesk
sudo docker compose pull
sudo docker compose up -d
```

### 9.3 证书续期验证

```bash
# 查看证书状态
sudo certbot certificates

# 测试自动续期（不影响现有证书）
sudo certbot renew --dry-run
```

> **提示**：certbot 安装时已自动注册 systemd timer，会在证书到期前 30 天自动续期。`--dry-run` 仅用于验证续期流程是否正常。

### 9.4 备份关键数据

```bash
# 完整备份（密钥 + 数据库 + 配置）
sudo tar -czf rustdesk-backup-$(date +%Y%m%d).tar.gz \
  /home/ubuntu/Apps/rustdesk/data \
  /home/ubuntu/Apps/rustdesk/api-data \
  /home/ubuntu/Apps/rustdesk/compose.yml \
  /home/ubuntu/Apps/rustdesk/.env
```

**密钥备份策略**：

- `id_ed25519` 私钥必须加密备份
- 推荐 Bandizip / 7-Zip **AES-256** 加密 + 强密码 + 文件名加密
- 加密后上传 OneDrive 等云存储
- **不建议**把密钥目录直接实时同步到网盘

### 9.5 重置 admin 密码

```bash
# 忘记密码时的重置命令
sudo docker exec rustdesk ./apimain reset-admin-pwd <新密码>
```

### 9.6 查看登录日志

```bash
sudo sqlite3 /home/ubuntu/Apps/rustdesk/api-data/rustdeskapi.db \
  "SELECT id,user_id,client,ip,platform,created_at FROM login_logs ORDER BY id DESC LIMIT 10;"
```

---

## 10. 总结

RustDesk OSS 本身没有 Pro 控制台，但通过 `lejianwen/rustdesk-api` 社区项目可以补齐设备列表、登录认证、地址簿同步、审计日志等核心能力，实现接近 TeamViewer 的体验。

部署中最容易踩坑的 4 个点：

1. **DNS 双栈**：A + AAAA 记录都要加，确保 IPv4-only 和 IPv6-only 客户端都能访问
2. **443 端口**：腾讯云安全组必须显式放行 TCP4 443，否则 IPv4 客户端登录超时
3. **`MUST_LOGIN`**：不要在客户端未配好 API Server 前启用，否则所有客户端断连
4. **`ENCRYPTED_ONLY`**：不要在初期启用，与旧客户端握手不兼容会导致连接被拒

**稳定部署的顺序**：

```text
先跑通连接（ENCRYPTED_ONLY=0, MUST_LOGIN=N）
  → 配置 API Server
    → 客户端登录 admin 账号
      → 验证设备注册 + 地址簿同步
        → 最后考虑安全加固（ENCRYPTED_ONLY=1, MUST_LOGIN=Y）
```

掌握这套方案后，你可以：

- 在自己的云主机上运行完整的 RustDesk 私有服务
- 用 PowerShell CLI 批量配置多台 Windows 客户端
- 通过 Web 控制台管理设备别名与分组
- 不再依赖 TeamViewer 或 RustDesk 官方 SaaS

---

## 11. 参考资料

- [lejianwen/rustdesk-api（后端 + Web Admin + Web Client）](https://github.com/lejianwen/rustdesk-api)
- [lejianwen/rustdesk-server（fork 增强版服务端）](https://github.com/lejianwen/rustdesk-server)
- [rustdesk/rustdesk-server（官方 OSS 服务端）](https://github.com/rustdesk/rustdesk-server)
- [RustDesk 官方文档](https://rustdesk.com/docs/)
- [nginx.org 官方仓库安装指南](https://nginx.org/en/linux_packages.html)
- [Certbot 官方文档](https://certbot.eff.org/)
