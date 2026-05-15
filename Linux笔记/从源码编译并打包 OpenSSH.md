# 🧰 从源码编译并打包 OpenSSH（自定义安装路径）
适用于 Ubuntu / Debian 系统

## 📦 一、前置准备
```bash
# 安装必要依赖
apt update
apt install -y build-essential libssl-dev zlib1g-dev libpam0g-dev checkinstall
```

---

## 📁 二、下载并解压源码
```bash
cd /mnt
wget https://mirrors.aliyun.com/pub/OpenBSD/OpenSSH/portable/openssh-10.3p1.tar.gz
tar -zxf openssh-10.3p1.tar.gz
cd openssh-10.3p1
```

---

## ⚙️ 三、编译配置参数
我们不覆盖系统原生 `/usr/sbin/sshd`，  
而是安装到 `/usr/local/`，以便独立运行。

```bash
./configure \
  --prefix=/usr/local \
  --sysconfdir=/etc/ssh \
  --with-md5-passwords \
  --with-pam
```

---

## 🏗️ 四、编译
```bash
make -j$(nproc)
```

---

## 📦 五、使用 checkinstall 打包 .deb 文件
checkinstall 会在安装的同时生成一个 `.deb` 包，便于安装、卸载、迁移。

```bash
checkinstall --install=no --pkgname=openssh-local
```

当提示输入信息时建议如下填写：

```plain
软件包将用下面的值来创建：

0 -  Maintainer: [ root@shiyaofei-virtual-machine ]
1 -  Summary: [ Package created with checkinstall 1.6.2 ]
2 -  Name:    [ openssh-local ]
3 -  Version: [ 10.3p1 ]
4 -  Release: [ 1 ]
5 -  License: [ GPL ]
6 -  Group:   [ checkinstall ]
7 -  Architecture: [ amd64 ]
8 -  Source location: [ openssh-10.3p1 ]
9 -  Alternate source location: [  ]
10 - Requires: [  ]
11 - Provides: [ openssh-local ]
12 - Conflicts: [  ]
13 - Replaces: [  ]

```

最终会生成：  
`openssh-local_10.3p1-1_amd64.deb`

---

## 📜 六、添加自定义维护脚本（DEBIAN 目录）


解压deb包，修改`DEBIAN/` 目录下的脚本文件。

```bash
dpkg-deb -R openssh-local_10.3p1-1_amd64.deb ssh-deb-pack/
cd ssh-deb-pack/
```



```plain
tmp
├── DEBIAN
│   ├── conffiles
│   └── control
└── usr
    ├── local
    └── share
```

### 1️⃣`DEBIAN/preinst`（安装前执行）
```bash
#!/bin/bash
set -e

# ============================================================
# 检查目标系统是否为 Ubuntu 18.04
# ============================================================
if [ -f /etc/os-release ]; then
    . /etc/os-release
    DISTRO_ID="${ID}"
    DISTRO_VERSION="${VERSION_ID}"
else
    echo "ERROR: 无法读取 /etc/os-release，无法确认系统版本" >&2
    exit 1
fi

if [ "${DISTRO_ID}" != "ubuntu" ] || [ "${DISTRO_VERSION}" != "18.04" ]; then
    echo "============================================================" >&2
    echo "ERROR: 此软件包仅支持 Ubuntu 18.04 (Bionic Beaver)" >&2
    echo "       当前系统为: ${PRETTY_NAME}" >&2
    echo "============================================================" >&2
    exit 1
fi

# ============================================================
# 停止原有 SSH 服务
# ============================================================
echo "==> 系统版本检查通过：${PRETTY_NAME}"
echo "==> 停止原有 SSH 服务"
systemctl stop ssh || true

exit 0
```

### 2️⃣ `DEBIAN/postinst`（安装后执行）
```bash
#!/bin/bash
set -e
# postinst：在文件安装完成后执行。用于修改 systemd 配置并启动服务。

SERVICE_FILE="/lib/systemd/system/ssh.service"

# 修改 systemd 服务文件路径，指向 /usr/local/sbin/sshd
if [ -f "$SERVICE_FILE" ]; then
    echo "==> 修改 systemd 服务文件以使用 /usr/local/sbin/sshd"
    sed -i 's|/usr/sbin/sshd|/usr/local/sbin/sshd|g' "$SERVICE_FILE"
fi

# 创建安全隔离目录 /var/empty
mkdir -p /var/empty
chown root:root /var/empty
chmod 755 /var/empty

# 重新加载 systemd 并启动新 ssh 服务
systemctl daemon-reload || true
systemctl enable ssh || true
systemctl restart ssh || true

echo "==> OpenSSH 10.2p1 (local) 已启动"
exit 0
```

---

### 3️⃣ `DEBIAN/prerm`（卸载前执行）
```bash
#!/bin/bash
set -e
# prerm：卸载前执行，停止当前 ssh 服务

echo "==> 停止 openssh-local 服务"
systemctl stop ssh || true
exit 0
```

---

### 4️⃣ `DEBIAN/postrm`（卸载后执行）
```bash
#!/bin/bash
set -e
# postrm：卸载后执行，恢复系统默认 sshd 路径并重启系统 ssh 服务

SERVICE_FILE="/lib/systemd/system/ssh.service"

if [ -f "$SERVICE_FILE" ]; then
    echo "==> 恢复 systemd 服务文件使用 /usr/sbin/sshd"
    sed -i 's|/usr/local/sbin/sshd|/usr/sbin/sshd|g' "$SERVICE_FILE"
fi

systemctl daemon-reload || true
systemctl restart ssh || true
echo "==> 已恢复系统默认 OpenSSH 服务"
exit 0
```

---

## 🔐 七、设置脚本权限
```bash
chmod 755 DEBIAN/{preinst,postinst,prerm,postrm}
```



## 📦 八、重新打包deb文件
```bash
dpkg-deb -b tmp/ openssh-local_10.2p1-1_amd64.deb
```

---

## 🚀 九、安装自定义 OpenSSH 包
```bash
dpkg -i openssh-local_10.2p1-1_amd64.deb
```

**安装输出示例：**

```plain
root@shiyaofei-virtual-machine:~# dpkg -i openssh-local_10.2p1-1_amd64.deb 
正在选中未选择的软件包 openssh-local。
(正在读取数据库 ... 系统当前共安装有 233372 个文件和目录。)
准备解压 openssh-local_10.2p1-1_amd64.deb  ...
==> 停止原有 SSH 服务
正在解压 openssh-local (10.2p1-1) ...
正在设置 openssh-local (10.2p1-1) ...
==> 修改 systemd 服务文件以使用 /usr/local/sbin/sshd
Synchronizing state of ssh.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable ssh
==> OpenSSH 10.2p1 (local) 已启动
正在处理用于 man-db (2.10.2-1) 的触发器 ...
```

---

## 🧹 十、卸载并恢复系统原生 OpenSSH
```bash
dpkg -r openssh-local
```

输出示例：

```plain
root@shiyaofei-virtual-machine:~# dpkg --remove openssh-local 
(正在读取数据库 ... 系统当前共安装有 233418 个文件和目录。)
正在卸载 openssh-local (10.2p1-1) ...
==> 停止 openssh-local 服务
==> 恢复 systemd 服务文件使用 /usr/sbin/sshd
==> 已恢复系统默认 OpenSSH 服务
正在处理用于 man-db (2.10.2-1) 的触发器 ...
```

---

## 🔍 十一、附加说明
| 内容 | 说明 |
| --- | --- |
| `/usr/local/sbin/sshd` | 你的新编译版 OpenSSH 运行路径 |
| `/usr/sbin/sshd` | 系统默认版本 |
| `/var/empty` | SSH 特权分离安全目录，必须存在且属主为root:root，权限为755 |
| `DEBIAN/control` | 定义包名、版本、依赖关系等信息 |
| `checkinstall` | 自动生成 `.deb`包并更新 dpkg 数据库 |


---

## 十二、验证
```bash
which sshd
# 输出应为 /usr/local/sbin/sshd

sshd -V
# OpenSSH_10.2p1, OpenSSL ...
```

---

## ✅ 📖十三、目录结构参考
```plain
.
├── DEBIAN
│   ├── conffiles
│   ├── control
│   ├── postinst
│   ├── postrm
│   ├── preinst
│   └── prerm
└── usr
    ├── local
    └── share
```

