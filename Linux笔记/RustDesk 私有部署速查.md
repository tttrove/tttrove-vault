---
title: RustDesk 私有部署速查：Docker + Nginx + API 三件套
tags: [RustDesk, Docker, Nginx, 腾讯云, 自托管, 远程桌面]
created: 2026-06-21
---

# RustDesk 私有部署速查：Docker + Nginx + API 三件套

## 1. 引言

商业版 RustDesk 中继有流量/隐私顾虑，公网直连又不稳定。本方案用一体化镜像 `rustdesk-server-s6`（hbbs+hbbr+API+Web）配合 Nginx 反代 HTTPS，在腾讯云轻量服务器上搭出一套可控、可审计、带账号体系的私有远程桌面服务，适合个人/小团队长期自托管。

## 2. 核心架构

单容器内集成四个组件：**hbbs**（ID/信令服务器，负责设备注册与打洞）、**hbbr**（中继服务器，NAT 穿透失败时转发流量）、**API**（账号、设备、地址簿管理后端）、**Web Admin**（管理控制台前端）。容器使用 `network_mode: host` 直接复用主机网络栈，省去端口映射的麻烦。Nginx 监听 443，做 TLS 终止后反代到容器内部的 21114 端口，对外只暴露标准 HTTPS 入口，API/Web/信令共用同一域名。

## 3. DNS 与安全组

| 项目 | 配置 |
|---|---|
| 域名 | `rd.tttrove.qzz.io` |
| DNS 记录 | A 记录 → 服务器公网 IPv4；AAAA 记录 → 服务器公网 IPv6 |
| 域名隔离 | `rd.tttrove.qzz.io` 专用 RustDesk；`linux.tttrove.qzz.io` 留给 SSH，避免混用 |

腾讯云安全组放行端口：

| 端口 | 协议 | 用途 |
|---|---|---|
| 80 | TCP | HTTP，证书校验/跳转 |
| 443 | TCP4/TCP6 | HTTPS，Web/API/信令统一入口 |
| 21115-21119 | TCP4/TCP6 | hbbs/hbbr 信令与中继 |
| 21116 | UDP4/UDP6 | hbbs UDP 打洞 |

> ⚠️ 坑2：腾讯云安全组的 IPv4 443 规则有被自动清理的情况，建议定期检查，并用 iptables 做兜底（见第 9 节）。

## 4. 目录结构

```bash
# 部署根目录：/home/ubuntu/Apps/rustdesk
ubuntu@VM-0-6-ubuntu:~/Apps/rustdesk$ tree
.
├── api-data
│   └── rustdeskapi.db          # API 服务账号/设备数据库
├── compose.yml                 # Docker Compose 主配置
├── compose.yml.bak.20260621-141822  # 修改前自动备份，习惯性留痕
└── data
    ├── db_v2.sqlite3            # hbbs 设备注册数据库
    ├── db_v2.sqlite3-shm
    ├── db_v2.sqlite3-wal
    ├── id_ed25519                # 服务器私钥，权限务必收紧
    └── id_ed25519.pub             # 服务器公钥，客户端配置需要

3 directories, 8 files
```

## 5. Docker Compose 速查

```yaml
# compose.yml
services:
  rustdesk:
    container_name: rustdesk
    image: lejianwen/rustdesk-server-s6:latest
    environment:
      - RELAY=rd.tttrove.qzz.io              # 中继服务器域名
      - ENCRYPTED_ONLY=0                     # 先跑通再加固，见坑1
      - MUST_LOGIN=N                         # 先跑通再加固，见坑1
      - TZ=Asia/Shanghai
      - RUSTDESK_API_LANG=zh-CN
      - RUSTDESK_API_APP_REGISTER=false      # 关闭自助注册，账号由管理员创建
      - RUSTDESK_API_APP_SHOW_SWAGGER=0      # 不暴露 API 文档
      - RUSTDESK_API_APP_CAPTCHA_THRESHOLD=3
      - RUSTDESK_API_APP_BAN_THRESHOLD=5
      - RUSTDESK_API_GIN_TRUST_PROXY=127.0.0.1   # 信任 Nginx 反代来源
      - RUSTDESK_API_RUSTDESK_ID_SERVER=rd.tttrove.qzz.io
      - RUSTDESK_API_RUSTDESK_RELAY_SERVER=rd.tttrove.qzz.io
      - RUSTDESK_API_RUSTDESK_API_SERVER=https://rd.tttrove.qzz.io
      - RUSTDESK_API_RUSTDESK_KEY_FILE=/data/id_ed25519.pub
      - RUSTDESK_API_JWT_KEY=${RUSTDESK_API_JWT_KEY}   # 从 .env 注入，不入库
    volumes:
      - ./data:/data            # 信令数据库 + 密钥对
      - ./api-data:/app/data    # API 账号/设备数据库
    network_mode: "host"        # 直接复用主机网络，避免端口映射遗漏
    restart: unless-stopped
```

启动/重建：

```bash
cd /home/ubuntu/Apps/rustdesk
sudo docker compose up -d        # 首次启动或配置变更后重建
sudo docker compose logs -f      # 查看启动日志，确认四组件均正常
```

## 6. Nginx + HTTPS 速查

```bash
# 1. 安装官方源 Nginx（Ubuntu 24.04）
sudo apt update
sudo apt install -y curl gnupg2 ca-certificates lsb-release
curl -fsSL https://nginx.org/keys/nginx_signing.key | sudo gpg --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" \
  | sudo tee /etc/apt/sources.list.d/nginx.list
sudo apt update && sudo apt install -y nginx   # 装出 1.30.x

# 2. 申请证书（webroot 方式，先放行 80 端口）
sudo apt install -y certbot
sudo certbot certonly --webroot -w /var/www/html -d rd.tttrove.qzz.io
```

核心反代配置：

```nginx
# /etc/nginx/conf.d/rustdesk.conf
server {
    listen 443 ssl;
    server_name rd.tttrove.qzz.io;

    ssl_certificate     /etc/letsencrypt/live/rd.tttrove.qzz.io/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/rd.tttrove.qzz.io/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:21114;       # 转发到容器内 Web/API 端口
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;  # 配合 GIN_TRUST_PROXY
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name rd.tttrove.qzz.io;
    location /.well-known/acme-challenge/ { root /var/www/html; }  # 续期校验
    location / { return 301 https://$host$request_uri; }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx   # 校验配置并热加载
```

## 7. Windows 客户端配置

GUI 手填（设置 → 网络 → 取消勾选"使用 ID/中继服务器"后填）：

| 字段 | 值 |
|---|---|
| ID 服务器 | `rd.tttrove.qzz.io` |
| 中继服务器 | `rd.tttrove.qzz.io` |
| API 服务器 | `https://rd.tttrove.qzz.io` |
| Key | `YOUR_SERVER_PUBLIC_KEY_HERE` |

PowerShell 批量配置脚本：

```powershell
# 以管理员身份运行，按需替换 exe 路径
$rd = "C:\Program Files\RustDesk\rustdesk.exe"
& $rd --option id-server "rd.tttrove.qzz.io"
& $rd --option relay-server "rd.tttrove.qzz.io"
& $rd --option api-server "https://rd.tttrove.qzz.io"
& $rd --option key "YOUR_SERVER_PUBLIC_KEY_HERE"
```

验证方法：

```bash
# 客户端本地：右下角状态应显示已连接到自建服务器，而非默认公共服务器
# 服务器端：观察容器日志中是否有设备心跳/注册记录
sudo docker compose logs -f --tail=50 rustdesk
```

## 8. 控制台使用

- Web Admin 地址：`https://rd.tttrove.qzz.io`，用管理员账号登录（密码自行设置，不要使用默认值）。
- 设备列表中可为每台机器设置**别名**，方便区分；支持按标签分组。
- 地址簿支持团队共享，登录同一账号的客户端会自动同步地址簿与权限分组，适合多人协作运维。

## 9. 故障速查表

| 现象 | 可能原因 | 处理方式 |
|---|---|---|
| 客户端全部离线 | 坑1：过早开启 `MUST_LOGIN=Y`，旧客户端未登录被拒 | 先确认所有终端已配置账号登录，再切换为 `Y`；紧急情况下改回 `N` 并 `docker compose up -d` 重建 |
| 连接超时/打洞失败 | 安全组 UDP 21116 或 TCP 21115-21119 未放行/被清理 | 重新检查腾讯云安全组规则，必要时用 `sudo iptables -L -n` 核对本机防火墙兜底规则 |
| `curl` 测试 443 返回拒绝 | 坑3：直接用 IP 测试触发 Nginx 的 Host 头校验失败 | 改用 `curl -4 --resolve rd.tttrove.qzz.io:443:<服务器IP> https://rd.tttrove.qzz.io` 模拟真实域名请求 |

## 10. 长期维护

- **容器更新**：`sudo docker compose pull && sudo docker compose up -d`，更新前建议备份 `data/` 和 `api-data/`。
- **证书续期**：Let's Encrypt 90 天有效，配置 cron 定时任务：`sudo certbot renew --quiet && sudo systemctl reload nginx`。
- **备份策略**：定期打包 `/home/ubuntu/Apps/rustdesk/{data,api-data}` 异地存放，密钥对（`id_ed25519*`）丢失会导致所有客户端需要重新配置 Key。

## 11. 总结

通过 Docker 一体化镜像 + Nginx 反代 + Let's Encrypt，在腾讯云上低成本搭建了私有 RustDesk 服务，兼顾易维护与可控性，适合长期自托管。

## 12. 参考资料

- [RustDesk 官方文档](https://rustdesk.com/docs/)
- [lejianwen/rustdesk-server-s6 镜像仓库](https://github.com/lejianwen/rustdesk-server-s6)
- [Nginx 官方安装文档](https://nginx.org/en/linux_packages.html)
- [Certbot 官方文档](https://certbot.eff.org/)
