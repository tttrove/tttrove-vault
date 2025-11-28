# hostC 通过 hostB 走 hostA 上互联网（APT 代理方案）
## 1. 网络拓扑与需求
+ **HostA**: 可访问互联网 (网关/出口节点)
+ **HostB**: 仅内网，连接 HostA 与 HostC (中间跳板)
+ **HostC**: 仅内网，需访问互联网更新软件 (目标节点)
+ **目标**: 在 HostC 上配置 `apt` 通过 SOCKS5 代理上网。

## 2. 核心原理
1. **链路建立**: 利用 SSH 的 `-J` (ProxyJump) 参数，从 C 通过 B 连接到 A。
2. **隧道创建**: 利用 SSH 的 `-D` 参数，在 HostC 本地开启一个 SOCKS5 监听端口。
3. **应用代理**: 配置 apt 使用 `socks5h` 协议（支持远程 DNS 解析）。

---

## 3. 操作步骤
### 第一步：建立 SSH SOCKS5 隧道
在 **HostC** 上执行以下命令。为了防止连接断开，建议使用 `-f -N` 参数让其在后台静默运行。

**前提**: 确保 HostC 能 SSH 访问 HostB，HostB 能 SSH 访问 HostA。建议配置 SSH Key 免密登录，否则需要多次输入密码。

```plain
# 语法: ssh -D [本地监听端口] -J [中间跳板用户]@[中间IP] [出口用户]@[出口IP]

# 示例:
# 本地端口: 1080
# 中间跳板: user@192.168.1.2 (HostB)
# 出口节点: user@10.0.0.1 (HostA)

ssh -f -N -D 127.0.0.1:1080 -J user@hostB_IP user@hostA_IP
```

**参数解析：**

+ `-f`: 请求 ssh 在执行命令前转入后台运行。
+ `-N`: 不执行远程命令（专门用于端口转发）。
+ `-D 1080`: 在本地开启 SOCKS5 代理，监听 1080 端口。
+ `-J user@hostB`: 指定 HostB 为跳板机。



### 第二步：添加curl测试，在 hostC 上执行：
```bash
export ALL_PROXY="socks5h://127.0.0.1:1080"
export http_proxy="$ALL_PROXY"
export https_proxy="$ALL_PROXY"
```

**注意关键点**：必须使用 `socks5h` 而不是 `socks5`。

+ `socks5`: 本地解析 DNS（HostC 无法联网，会解析失败）。
+ `socks5h`: 远程解析 DNS（让 HostA 帮忙解析域名，**必须用这个**）。

测试`curl`：

```bash
curl https://www.baidu.com
```

或

```bash
curl --proxy socks5h://127.0.0.1:1080 https://www.baidu.com
```

如果能访问成功，说明 SOCKS5 通道可用。

---

### 第三步：配置 apt 使用 SOCKS5
在 **HostC** 上，创建或编辑 apt 的配置文件。

创建apt配置文件`/etc/apt/apt.conf.d/80proxy`：

```bash
sudo vim /etc/apt/apt.conf.d/80proxy
```

写入以下内容：

```bash
Acquire::http::Proxy "socks5h://127.0.0.1:1080/";
Acquire::https::Proxy "socks5h://127.0.0.1:1080/";
```

---

### 第四步：验证与测试
在 **HostC** 上更新软件源：

Bash

```bash
sudo apt update
```

如果不报错且能拉取到数据，说明配置成功。

---

## 4. 进阶方案：保持隧道稳定 (Autossh)
普通的 SSH 隧道可能会因为网络波动断开。生产环境中推荐使用 `autossh` 自动重连。

**1. 安装 autossh (如果无法联网安装，可先用上面的临时方法连通后安装):**

Bash

```bash
sudo apt install autossh
```

**2. 启动持久化隧道:**

Bash

```bash
autossh -M 0 -f -N \
    -o "ServerAliveInterval 30" \
    -o "ServerAliveCountMax 3" \
    -D 127.0.0.1:1080 \
    -J user@hostB_IP user@hostA_IP
```

---

## 5. 常见问题排查 (Troubleshooting)
| **问题现象**                                     | **可能原因**   | **解决方案**                                                                   |
| -------------------------------------------- | ---------- | -------------------------------------------------------------------------- |
| **apt 报错: "Temporary failure resolving..."** | DNS 解析失败   | 检查 apt.conf 是否写成了 `socks5h` (带有 **h**)。                                    |
| **apt 报错: "Connection refused"**             | SSH 隧道未建立  | 执行 `netstat -tnlp`                                                         |
| **apt 报错: "Protocol not supported"**         | apt 版本过低   | 旧版 apt (如 Ubuntu 16.04 以下) 可能不支持 socks 直接配置。需配合 `tsocks`或 `proxychains`使用。 |
| **连接极慢**                                     | 路由跳数多或加密开销 | SSH 隧道本身有加密开销，属正常现象；可在 SSH 命令中添加 `-C`开启压缩试试。                               |


---

## 6. 总结
1. **开启隧道 (HostC 执行):**

```bash
ssh -qTfnN -D 1080 -J root@HostB root@HostA
```

2. **临时使用 (不修改配置文件):**

```bash
curl --proxy socks5h://127.0.0.1:1080 https://www.baidu.com
sudo apt -o Acquire::http::Proxy="socks5h://127.0.0.1:1080/" update
```

3. **永久配置 (**`**/etc/apt/apt.conf.d/80proxy**`**):**

```bash
Acquire::http::Proxy "socks5h://127.0.0.1:1080/";
Acquire::https::Proxy "socks5h://127.0.0.1:1080/";
```

```bash
curl https://www.baidu.com
sudo apt update
```

