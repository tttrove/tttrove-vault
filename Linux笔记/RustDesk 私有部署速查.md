---
title: RustDesk 私有部署速查：Docker + Nginx + API 三件套
tags: [RustDesk, Docker, Nginx, Cloudflare, 自托管, 远程桌面]
created: 2026-06-21
---

# RustDesk 私有部署速查：Docker + Nginx + API 三件套

**换机重部署速查手册**。服务器到期换机时，从上到下照做即可还原整套 RustDesk 私有服务：Docker 一体化镜像 + Nginx HTTPS + Web 控制台 + 设备别名 + 地址簿同步。

域名永远托管在 Cloudflare，证书续期走 Cloudflare DNS 验证，**不依赖 80 端口**，可彻底关闭 80 减少攻击面。

**核心组件**：

- **服务端**：`lejianwen/rustdesk-server-s6`（社区 fork，单容器含 hbbs + hbbr + API + Web Admin）
- **反代**：nginx.org 官方版 + certbot（Cloudflare DNS 验证）
- **域名**：`rd.tttrove.qzz.io` 专给 RustDesk，`linux.tttrove.qzz.io` 留给 SSH
- **部署用户**：`ubuntu`（非 root，sudo 提权）
- **部署路径**：`/home/ubuntu/Apps/rustdesk`

---

## 目录

- [1. 核心架构](#1-核心架构)
- [2. Cloudflare DNS 与安全组](#2-cloudflare-dns-与安全组)
- [3. 目录结构](#3-目录结构)
- [4. Docker Compose 部署](#4-docker-compose-部署)
- [5. Nginx + HTTPS](#5-nginx--https)
- [6. Windows 客户端配置](#6-windows-客户端配置)
- [7. 控制台使用](#7-控制台使用)
- [8. 日常维护](#8-日常维护)
- [9. 参考资料](#9-参考资料)

---

## 1. 核心架构

```
公网用户 (IPv4/IPv6)
    ↓
云主机 Ubuntu 24.04（非 root 用户 ubuntu）
├─ Nginx 1.30 (443 → 21114)
│   └─ Let's Encrypt 证书（Cloudflare DNS 验证，自动续期）
└─ Docker: lejianwen/rustdesk-server-s6
    ├─ hbbs + hbbr (21115-21119)
    └─ API + Web Admin (21114)
```

---

## 2. Cloudflare DNS 与安全组

### 2.1 Cloudflare DNS 记录

域名托管在 Cloudflare，添加 `rd` 子域名双栈记录（**关闭 Cloudflare 代理，灰云直连**）：

| 名称 | 类型 | 内容 | 代理 |
|------|------|------|------|
| `rd` | A | 服务器 IPv4 | 仅 DNS（灰云） |
| `rd` | AAAA | 服务器 IPv6 | 仅 DNS（灰云） |

> **提示**：必须灰云直连，否则 Cloudflare 代理只转发 HTTP/HTTPS，RustDesk 的 TCP/UDP 21115-21119 端口无法穿透。

### 2.2 云主机安全组

入站规则放行以下端口（TCP4/TCP6 + UDP4/UDP6）：

| 端口 | 协议 | 用途 |
|------|------|------|
| 443 | TCP4/TCP6 | Web Admin / API HTTPS |
| 21115 | TCP4/TCP6 | hbbs NAT 测试 |
| 21116 | TCP4/TCP6 + UDP4/UDP6 | hbbs ID 注册（UDP 必须放行） |
| 21117 | TCP4/TCP6 | hbbr 中继 |
| 21118 | TCP4/TCP6 | hbbs WebSocket |
| 21119 | TCP4/TCP6 | hbbr WebSocket |

> **重要**：`21116` 必须同时放行 TCP 和 UDP，否则客户端无法注册 ID。`21114` 不需要对公网开放（仅 Nginx 本机反代）。**80 端口不需要放行**（证书续期走 Cloudflare DNS 验证）。

---

## 3. 目录结构

```
/home/ubuntu/Apps/rustdesk
├── api-data/
│   └── rustdeskapi.db          # Web 控制台数据库（用户、设备、地址簿）
├── compose.yml                 # Docker Compose 配置
├── .env                        # JWT_KEY 环境变量
└── data/
    ├── db_v2.sqlite3           # hbbs 设备注册数据库
    ├── id_ed25519              # 服务端身份私钥（核心机密）
    └── id_ed25519.pub          # 服务端公钥（客户端 Key 来源）
```

**关键文件**：

- `data/id_ed25519`：RustDesk 服务端身份私钥，**决定服务器身份**。换机重部署时**务必从备份恢复此文件**，否则客户端 Key 变化需全部重新配置
- `data/id_ed25519.pub`：公钥，客户端配置中的 `Key` 字段填此内容
- `api-data/rustdeskapi.db`：Web 控制台 SQLite 数据库，含用户、设备、地址簿、审计日志

---

## 4. Docker Compose 部署

### 4.1 创建 compose.yml

```yaml
# /home/ubuntu/Apps/rustdesk/compose.yml
services:
  rustdesk:
    container_name: rustdesk
    image: lejianwen/rustdesk-server-s6:latest
    environment:
      - RELAY=rd.tttrove.qzz.io
      - ENCRYPTED_ONLY=0              # 兼容模式，客户端全配好后再改 1
      - MUST_LOGIN=N                  # 客户端全配好后再改 Y
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
      - ./data:/data           # 密钥挂载此处（从备份恢复 id_ed25519）
      - ./api-data:/app/data
    network_mode: "host"
    restart: unless-stopped
```

### 4.2 创建 .env 文件

```bash
echo "RUSTDESK_API_JWT_KEY=$(openssl rand -hex 32)" > /home/ubuntu/Apps/rustdesk/.env
sudo chmod 600 /home/ubuntu/Apps/rustdesk/.env
```

### 4.3 恢复密钥并启动

```bash
# 恢复备份的 id_ed25519 密钥对（保持客户端 Key 不变）
sudo cp /path/to/backup/id_ed25519 /home/ubuntu/Apps/rustdesk/data/
sudo cp /path/to/backup/id_ed25519.pub /home/ubuntu/Apps/rustdesk/data/
sudo chmod 600 /home/ubuntu/Apps/rustdesk/data/id_ed25519

# 启动
cd /home/ubuntu/Apps/rustdesk
sudo mkdir -p data api-data
sudo docker compose up -d
```

### 4.4 获取初始 admin 密码

```bash
sudo docker logs rustdesk 2>&1 | grep "Admin Password"
```

**立即登录改密**：`https://rd.tttrove.qzz.io/_admin/`

### 4.5 核心环境变量说明

| 变量 | 作用 | 调整建议 |
|------|------|---------|
| `ENCRYPTED_ONLY` | `0`=兼容 `1`=强制加密 | 部署初期用 `0`，稳定后改 `1` |
| `MUST_LOGIN` | `N`=可匿名 `Y`=强制登录 | **客户端全配好后再改 `Y`** |
| `RUSTDESK_API_LANG` | 控制台语言 | `zh-CN` / `en` |
| `RUSTDESK_API_APP_REGISTER` | 是否允许公开注册 | 建议 `false` |
| `RUSTDESK_API_JWT_KEY` | API JWT 签名密钥 | 随机 32 字节，设定后不可随意改 |

---

## 5. Nginx + HTTPS

### 5.1 安装 nginx.org 官方版 + certbot DNS 插件

```bash
# 添加 nginx.org 官方仓库
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] https://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" | sudo tee /etc/apt/sources.list.d/nginx.list

# 安装 nginx + certbot + Cloudflare DNS 插件
sudo apt update && sudo apt install -y nginx certbot python3-certbot-dns-cloudflare

sudo systemctl enable --now nginx
```

### 5.2 创建 Cloudflare API Token

1. Cloudflare → 右上角头像 → **My Profile** → **API Tokens**
2. **Create Token** → 选模板 **"Edit zone DNS"**
3. Zone Resources：`Include` → `Specific zone` → `tttrove.qzz.io`
4. 创建并复制 Token（只显示一次）

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
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d rd.tttrove.qzz.io \
  --non-interactive \
  --agree-tos \
  -m admin@yourdomain.com

# 测试自动续期（加 --no-random-sleep-on-renew 避免随机延迟）
sudo certbot renew --dry-run --no-random-sleep-on-renew
```

> certbot 已自动注册 systemd timer，证书到期前 30 天自动续期，无需 80 端口。

### 5.4 Nginx 反向代理配置（仅 443，不监听 80）

```nginx
# /etc/nginx/conf.d/rustdesk.conf
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

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

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_connect_timeout 60s;
    }
}
```

```bash
# 删除残留默认配置（避免 80 端口被默认 server 监听）
sudo rm -f /etc/nginx/conf.d/default.conf
sudo rm -f /etc/nginx/sites-available/default
sudo rm -f /etc/nginx/sites-enabled/default

# 测试并重启（restart 确保老 worker 退出，不监听 80）
sudo nginx -t && sudo systemctl restart nginx

# 验证 80 端口已无监听
sudo ss -tlnp | grep ":80 " || echo "✓ 80 端口已无监听"
```

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

> 填完点应用，**彻底退出 RustDesk 托盘图标再重启**（右键托盘 → Quit → 重新打开）。

### 6.2 PowerShell CLI 批量配置

```powershell
# 必须以管理员身份运行
$rd = "C:\Program Files\RustDesk\rustdesk.exe"
& $rd --option custom-rendezvous-server "rd.tttrove.qzz.io"
& $rd --option relay-server "rd.tttrove.qzz.io"
& $rd --option api-server "https://rd.tttrove.qzz.io"
& $rd --option key "YOUR_SERVER_PUBLIC_KEY_HERE"
```

> `--option` 命令执行后**无回显**（GUI 子系统程序 stdout 不输出到 PowerShell）。验证需读配置文件。

### 6.3 验证配置

```powershell
Get-Content "$env:APPDATA\RustDesk\config\RustDesk2.toml"
```

检查 `[options]` 段是否包含四项配置（`custom-rendezvous-server` / `key` / `api-server` / `relay-server`）。

### 6.4 客户端登录账号

RustDesk → 右上角 ≡ → **设置 → 账户 → 登录**，输入 Web 控制台账号密码。登录后设备自动注册到 `peers` 表，地址簿跨设备同步。

---

## 7. 控制台使用

### 7.1 Web 控制台地址

```
https://rd.tttrove.qzz.io/_admin/
```

### 7.2 设备别名与分组

控制台 → **设备管理** → 编辑设备 `alias` 字段（如 `HOME-PC` / `OFFICE-LAPTOP`），替代数字 ID。群组管理可创建 `家用` / `公司` 等分组，客户端地址簿自动同步。

### 7.3 Web Client 浏览器远控

```
https://rd.tttrove.qzz.io/_webclient/
```

无需安装客户端，浏览器输入目标设备 ID 和密码即可远控。

### 7.4 审计日志

控制台提供登录日志、连接日志、文件传输日志三类审计记录。

---

## 8. 日常维护

### 8.1 查看容器状态

```bash
cd /home/ubuntu/Apps/rustdesk
sudo docker compose ps
sudo docker logs rustdesk --tail 50
```

### 8.2 更新容器镜像

```bash
cd /home/ubuntu/Apps/rustdesk
sudo docker compose pull
sudo docker compose up -d
```

### 8.3 证书续期验证

```bash
sudo certbot certificates
sudo certbot renew --dry-run --no-random-sleep-on-renew
```

### 8.4 备份关键数据

```bash
# 完整备份（密钥 + 数据库 + 配置）
sudo tar -czf rustdesk-backup-$(date +%Y%m%d).tar.gz \
  /home/ubuntu/Apps/rustdesk/data \
  /home/ubuntu/Apps/rustdesk/api-data \
  /home/ubuntu/Apps/rustdesk/compose.yml \
  /home/ubuntu/Apps/rustdesk/.env
```

**密钥备份策略**：`id_ed25519` 私钥用 7-Zip / Bandizip **AES-256** 加密 + 强密码 + 文件名加密后上传云存储，**不要**把密钥目录直接实时同步到网盘。

### 8.5 重置 admin 密码

```bash
sudo docker exec rustdesk ./apimain reset-admin-pwd <新密码>
```

---

## 9. 参考资料

- [lejianwen/rustdesk-api（后端 + Web Admin + Web Client）](https://github.com/lejianwen/rustdesk-api)
- [lejianwen/rustdesk-server（fork 增强版服务端）](https://github.com/lejianwen/rustdesk-server)
- [rustdesk/rustdesk-server（官方 OSS 服务端）](https://github.com/rustdesk/rustdesk-server)
- [RustDesk 官方文档](https://rustdesk.com/docs/)
- [nginx.org 官方仓库安装指南](https://nginx.org/en/linux_packages.html)
- [Certbot 官方文档](https://certbot.eff.org/)
