# Nginx 学习与高级配置指南

## 核心概念

Nginx 采用**主进程 + 工作进程**模型，主进程负责读取配置和管理工作进程，工作进程处理实际请求。配置文件采用层级块结构：`main → events → http → server → location`。

---

## 配置文件结构总览

```nginx
# 全局块（main context）
user  nginx;
worker_processes  auto;          # 自动匹配 CPU 核数
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;    # 每个工作进程最大连接数
    use epoll;                   # Linux 推荐使用 epoll
    multi_accept on;             # 一次接受多个连接
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # 日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    sendfile        on;
    tcp_nopush      on;
    keepalive_timeout  65;
    gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

---

## 高级配置示例

### 1. 反向代理 + 负载均衡

```nginx
http {
    # 上游服务器组
    upstream backend_pool {
        least_conn;                        # 最少连接算法（默认 round-robin）
        
        server 192.168.1.10:8080 weight=3; # 权重高，分配更多请求
        server 192.168.1.11:8080 weight=1;
        server 192.168.1.12:8080 backup;   # 备用服务器
        
        keepalive 32;                      # 保持连接池
    }

    server {
        listen 80;
        server_name api.example.com;

        location / {
            proxy_pass         http://backend_pool;
            proxy_http_version 1.1;
            
            # 传递真实客户端信息
            proxy_set_header   Host              $host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
            proxy_set_header   Connection        "";
            
            # 超时设置
            proxy_connect_timeout  5s;
            proxy_send_timeout     60s;
            proxy_read_timeout     60s;
            
            # 缓冲设置
            proxy_buffering        on;
            proxy_buffer_size      4k;
            proxy_buffers          8 32k;
        }
    }
}
```

---

### 2. HTTPS + HTTP/2 + HSTS

```nginx
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # 证书配置
    ssl_certificate     /etc/ssl/certs/example.com.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;
    
    # 现代安全协议
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # SSL 会话缓存（提升性能）
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;
    
    # OCSP Stapling（证书在线验证）
    ssl_stapling        on;
    ssl_stapling_verify on;
    resolver            8.8.8.8 8.8.4.4 valid=300s;
    
    # 安全响应头
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header Referrer-Policy "strict-origin-when-cross-origin";
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'";
}

# HTTP 强制跳转 HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

---

### 3. 缓存配置（代理缓存）

```nginx
http {
    # 定义缓存路径和参数
    proxy_cache_path /var/cache/nginx 
        levels=1:2                   # 两级目录结构
        keys_zone=my_cache:10m       # 共享内存区域名:大小
        max_size=10g                 # 最大磁盘占用
        inactive=60m                 # 60分钟未访问则删除
        use_temp_path=off;

    server {
        location / {
            proxy_pass         http://backend;
            proxy_cache        my_cache;
            proxy_cache_key    "$scheme$request_method$host$request_uri";
            proxy_cache_valid  200 302  10m;   # 200/302 缓存10分钟
            proxy_cache_valid  404      1m;
            proxy_cache_use_stale error timeout updating; # 后端故障时使用旧缓存
            
            add_header X-Cache-Status $upstream_cache_status; # 显示缓存命中状态
        }
    }
}
```

---

### 4. 限流与防护

```nginx
http {
    # 限制请求速率（按 IP）
    limit_req_zone  $binary_remote_addr zone=req_limit:10m rate=10r/s;
    
    # 限制并发连接数
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    server {
        location /api/ {
            limit_req  zone=req_limit burst=20 nodelay; # 突发允许20个，不延迟
            limit_conn conn_limit 10;                   # 同一 IP 最多10个并发连接
            
            limit_req_status  429;
            limit_conn_status 429;
        }
        
        # 防止恶意扫描：屏蔽常见攻击路径
        location ~* \.(php|asp|aspx|jsp)$ {
            deny all;
        }
        
        # 禁止访问隐藏文件
        location ~ /\. {
            deny all;
            return 404;
        }
    }
}
```

---

### 5. 静态资源优化

```nginx
server {
    root /var/www/html;
    
    # Gzip 压缩
    gzip on;
    gzip_vary on;
    gzip_min_length  1024;
    gzip_comp_level  6;
    gzip_types text/plain text/css application/json application/javascript 
               text/xml application/xml image/svg+xml;

    # 静态文件长期缓存
    location ~* \.(jpg|jpeg|png|gif|ico|svg|webp)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        add_header Vary Accept-Encoding;
    }

    location ~* \.(css|js)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # 开启 sendfile 零拷贝传输
    sendfile    on;
    tcp_nopush  on;
    tcp_nodelay on;
    
    # 前端 SPA 路由支持（React/Vue）
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

### 6. WebSocket 代理

```nginx
server {
    location /ws/ {
        proxy_pass          http://websocket_backend;
        proxy_http_version  1.1;
        
        # 关键：升级协议
        proxy_set_header    Upgrade    $http_upgrade;
        proxy_set_header    Connection "upgrade";
        proxy_set_header    Host       $host;
        
        # WebSocket 长连接，不要超时太短
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
    }
}
```

---

## 常用内置变量速查

|变量|含义|
|---|---|
|`$remote_addr`|客户端 IP|
|`$request_uri`|完整请求 URI（含参数）|
|`$uri`|请求路径（不含参数）|
|`$host`|请求的 Host 头|
|`$scheme`|协议（http / https）|
|`$upstream_cache_status`|缓存状态（HIT / MISS / BYPASS）|
|`$http_user_agent`|客户端 User-Agent|
|`$status`|响应状态码|

---

## location 匹配优先级

```
= 精确匹配     （最高优先级）
^~ 前缀匹配    （匹配后停止正则搜索）
~  区分大小写正则
~* 不区分大小写正则
/  普通前缀    （最低优先级）
```

```nginx
location = /favicon.ico  { ... }   # ① 只匹配这个确切路径
location ^~ /static/     { ... }   # ② 以 /static/ 开头，不再匹配正则
location ~* \.(jpg|png)$ { ... }   # ③ 图片正则
location /               { ... }   # ④ 兜底
```

---

## 性能调优要点

```nginx
# worker_processes 设为 CPU 核数或 auto
worker_processes auto;

# 单个 worker 最大文件描述符
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    # 关闭慢客户端 DoS
    client_body_timeout   12s;
    client_header_timeout 12s;
    send_timeout          10s;
    
    # 减小响应头缓冲，节省内存
    proxy_busy_buffers_size 64k;
    
    # 关闭版本号暴露
    server_tokens off;
}
```
