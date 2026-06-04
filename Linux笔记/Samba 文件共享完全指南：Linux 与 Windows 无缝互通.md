# Samba 文件共享完全指南：Linux 与 Windows 无缝互通

在混合操作系统环境中，Linux 服务器与 Windows 客户端之间共享文件是最常见的需求之一。无论你是搭建家庭 NAS、开发环境还是企业文件服务器，都需要一个稳定、高性能的跨平台文件共享方案。

**Samba** 是一款开源的 SMB/CIFS 协议实现，允许 Linux/Unix 系统与 Windows 系统无缝共享文件和打印机。它不仅是 Linux 接入 Windows 网络的“桥接器”，更是众多 NAS 系统的核心组件。

Samba 的核心优势：

- **原生 Windows 集成**：Windows 无需安装任何额外软件，即可通过“网络驱动器”或“网络位置”直接访问。
- **灵活的权限模型**：支持 Linux 文件权限与 Windows ACL 的双向映射，可精细控制读写、执行权限。
- **高性能与稳定性**：作为成熟的企业级方案，Samba 支持多线程、大文件传输和并发连接。
- **身份认证兼容**：可作为域控制器（Domain Controller），与 Active Directory 集成。

本文将从安装、基础配置、共享管理、权限控制、安全加固、性能调优及故障排除等方面，帮助读者系统掌握 Samba 的使用方法。

---

## 目录

- [1. 引言：什么是 Samba？](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#1-引言什么是-samba)
- [2. 安装与前置准备](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#2-安装与前置准备)
  - [2.1 检查是否已安装](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#21-检查是否已安装)
  - [2.2 在主流 Linux 发行版中安装](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#22-在主流-linux-发行版中安装)
- [3. 基础配置：创建第一个共享](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#3-基础配置创建第一个共享)
  - [3.1 配置文件结构](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#31-配置文件结构)
  - [3.2 最简单的共享示例](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#32-最简单的共享示例)
  - [3.3 设置 Samba 用户](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#33-设置-samba-用户)
  - [3.4 从 Windows 访问](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#34-从-windows-访问)
- [4. 高级配置详解](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#4-高级配置详解)
  - [4.1 共享参数速查表](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#41-共享参数速查表)
  - [4.2 权限控制进阶：force user / create mask](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#42-权限控制进阶force-user--create-mask)
  - [4.3 隐藏共享与访问限制](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#43-隐藏共享与访问限制)
  - [4.4 多用户与来宾访问](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#44-多用户与来宾访问)
- [5. 常见场景与最佳实践](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#5-常见场景与最佳实践)
  - [5.1 场景 1：家庭媒体中心共享](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#51-场景-1家庭媒体中心共享)
  - [5.2 场景 2：开发团队共享代码目录](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#52-场景-2开发团队共享代码目录)
  - [5.3 场景 3：备份存储专用共享](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#53-场景-3备份存储专用共享)
  - [5.4 最佳实践](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#54-最佳实践)
- [6. 故障排除](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#6-故障排除)
  - [6.1 无法从 Windows 访问共享](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#61-无法从-windows-访问共享)
  - [6.2 登录时反复提示密码](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#62-登录时反复提示密码)
  - [6.3 写入文件提示权限不足](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#63-写入文件提示权限不足)
  - [6.4 中文文件名显示乱码](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#64-中文文件名显示乱码)
- [7. 总结](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#7-总结)
- [8. 参考资料](Samba%20文件共享完全指南：Linux%20与%20Windows%20无缝互通.md#8-参考资料)

---

## 1. 引言：什么是 Samba？

在 Linux 系统管理或企业运维中，经常需要将 Linux 服务器上的文件共享给 Windows 客户端使用。传统的 FTP 或 NFS 要么配置繁琐，要么在 Windows 上兼容性差。而 Windows 原生支持 SMB（Server Message Block）协议，Linux 需要一套能够“模拟”Windows 文件共享服务的软件。

**Samba** 正是这样一款开源软件套件，它在 Linux/Unix 系统上实现了 SMB/CIFS 协议，使 Linux 服务器可以像 Windows 服务器一样提供文件共享和打印服务。

Samba 的核心优势：

- **原生 Windows 集成**：Windows 客户端通过“\\服务器IP\共享名”即可访问，无需安装任何插件。
- **灵活的权限模型**：支持用户级别共享、来宾访问，并可精确控制新建文件和目录的默认权限。
- **高性能与稳定性**：作为企业级方案，Samba 在生产环境中久经考验，支持多连接、大文件并发读写。
- **域控支持**：可作为主域控制器（PDC）或加入 Active Directory 域。

本文将从安装、基础共享配置、高级权限控制、常见场景及故障排除等方面，帮助读者完整掌握 Samba 的应用。

---

## 2. 安装与前置准备

### 2.1 检查是否已安装

通过以下命令确认系统中是否已安装 Samba：

```bash
smbd --version   # 输出版本号如 Version 4.15.13-Ubuntu
```

若提示 `command not found`，则需要手动安装。

### 2.2 在主流 Linux 发行版中安装

#### Debian / Ubuntu

```bash
sudo apt update && sudo apt install -y samba
```

#### RHEL / CentOS 8+ / Fedora

```bash
sudo dnf install -y samba
```

#### Arch Linux

```bash
sudo pacman -S samba
```

#### openSUSE

```bash
sudo zypper install -y samba
```

安装完成后，Samba 服务会默认启用。核心服务包括：
- `smbd`：提供文件和打印共享服务（监听 139/445 端口）
- `nmbd`：NetBIOS 名称解析服务（监听 137/138 端口）

可通过以下命令检查服务状态：

```bash
sudo systemctl status smbd
```

---

## 3. 基础配置：创建第一个共享

### 3.1 配置文件结构

Samba 的主配置文件为 `/etc/samba/smb.conf`，采用类似 INI 的格式：

- `[global]`：全局配置段，定义工作组、网络安全、日志等。
- `[共享名]`：每个共享目录单独成段，包含路径、访问权限等。

修改配置前，建议备份原文件：

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

### 3.2 最简单的共享示例

假设要共享 `/srv/share` 目录，允许所有已验证用户读写。

```bash
# 创建共享目录并设置 Linux 权限
sudo mkdir -p /srv/share
sudo chmod 777 /srv/share          # 简单测试，生产中应限制权限
```

编辑 `/etc/samba/smb.conf`，在末尾添加：

```ini
[myshare]
   comment = My First Samba Share
   path = /srv/share
   browseable = yes        # 在网络上可见
   writable = yes          # 允许写入
   read only = no
   guest ok = no           # 禁止匿名访问
```

### 3.3 设置 Samba 用户

Samba 使用独立的密码数据库（与系统 `/etc/shadow` 分离）。需要为每个系统用户添加 Samba 密码：

```bash
sudo smbpasswd -a 用户名
```

> **注意**：用户名必须是系统中已存在的用户。例如添加当前登录用户：
> ```bash
> sudo smbpasswd -a $USER
> ```

### 3.4 从 Windows 访问

1. 在 Windows 资源管理器中，输入地址：
   ```
   \\Linux服务器IP\myshare
   ```
   例如 `\\192.168.1.100\myshare`

2. 系统会弹出身份验证窗口，输入 Samba 用户名和密码。

3. 可选：右键“此电脑” → “映射网络驱动器”，将共享固定为某个盘符（如 `Z:`）。

> **提示**：若无法访问，请检查 Linux 防火墙是否开放 139/445 端口：
> ```bash
> sudo ufw allow samba        # Ubuntu 默认防火墙
> ```

---

## 4. 高级配置详解

### 4.1 共享参数速查表

| 参数 | 取值 | 说明 |
|------|------|------|
| `comment` | 字符串 | 共享描述，Windows 中可见 |
| `path` | 绝对路径 | 共享的本地目录 |
| `browseable` | yes/no | 是否在“网上邻居”中可见 |
| `read only` | yes/no | 是否只读（`yes` 时不能写入） |
| `writable` | yes/no | 是否可写（`read only = no` 的同义） |
| `guest ok` | yes/no | 是否允许匿名（来宾）访问 |
| `valid users` | 用户列表（空格分隔） | 仅允许这些用户访问 |
| `invalid users` | 用户列表 | 禁止这些用户访问 |
| `create mask` | 八进制数（如 0644） | 新建文件的默认权限（减去 umask） |
| `directory mask` | 八进制数（如 0755） | 新建目录的默认权限 |
| `force user` | 用户名 | 所有写入操作映射为该用户 |
| `force group` | 组名 | 所有写入操作映射为该组 |
| `hosts allow` | IP/网段 | 仅允许这些 IP 访问 |
| `hosts deny` | IP/网段 | 拒绝这些 IP 访问 |

### 4.2 权限控制进阶：force user / create mask

在混合环境中，常常希望所有通过 Samba 创建的文件具有统一的属主和权限（例如开发目录中所有文件均为 `www-data` 用户，权限 `644/755`）。

示例配置：

```ini
[devshare]
   path = /var/www/html
   writable = yes
   force user = www-data
   force group = www-data
   create mask = 0644
   directory mask = 0755
```

- `force user`：忽略客户端传递的用户身份，强制使用 `www-data` 作为文件属主。
- `create mask`：新建文件权限为 `0666 & ~create mask`。例如 `create mask = 0644` 意味着去掉 group 和 other 的写权限，最终 `644`。
- `directory mask`：同理，`0755` 表示目录权限为 `rwxr-xr-x`。

### 4.3 隐藏共享与访问限制

- **隐藏共享**：在共享名后加 `$` 符号（如 `[secret$]`），使该共享在浏览列表中不可见，但仍可通过完整路径 `\\IP\secret$` 访问。
- **按 IP 限制**：

```ini
[internal]
   path = /srv/internal
   hosts allow = 192.168.1.0/24 127.0.0.1
   hosts deny = 0.0.0.0/0
```

### 4.4 多用户与来宾访问

- **多用户共享**：使用 `valid users` 限制访问人员列表：

```ini
[teamshare]
   path = /srv/team
   valid users = alice, bob, charlie
   write list = alice, bob      # 仅 alice 和 bob 可写，其他人只读
```

- **来宾共享（无密码访问）**：

```ini
[public]
   path = /srv/public
   guest ok = yes
   read only = no
```

同时需要在 `[global]` 中设置：

```ini
[global]
   map to guest = Bad User
   guest account = nobody
```

---

## 5. 常见场景与最佳实践

### 5.1 场景 1：家庭媒体中心共享

在家庭局域网中，将 Linux 主机上的电影、音乐分享给 Windows 电脑和电视盒子。

```ini
[media]
   comment = Home Media
   path = /srv/media
   browseable = yes
   read only = yes          # 只读，防止误删
   guest ok = yes           # 无需密码
   force user = media_user
```

确保 `/srv/media` 对 `media_user` 有读取权限。

### 5.2 场景 2：开发团队共享代码目录

团队所有成员写入的文件应强制属于 `developer` 组，并自动设置为 `755/644`。

```ini
[code]
   path = /srv/code
   valid users = @developers
   force group = developers
   create mask = 0664
   directory mask = 2775
```

- `directory mask = 2775`：其中的 `2` 表示设置 SGID 位，新文件和目录继承父目录的组。

### 5.3 场景 3：备份存储专用共享

大型备份共享，建议限制最大连接数、记录日志。

```ini
[backup]
   path = /backup
   writable = yes
   max connections = 10
   veto files = /.*/               # 隐藏隐藏文件（.开头）
   delete veto files = no
```

### 5.4 最佳实践

#### 1. 始终使用专用目录，避免共享 `/home` 或 `/root`
共享用户家目录容易引发安全隐患。应创建独立目录（如 `/srv/shares/`）。

#### 2. 为 Samba 创建独立系统用户
不要直接使用 `root` 作为 `force user`。创建专用用户并赋予最小权限：

```bash
sudo useradd -r -s /sbin/nologin sambauser
sudo chown -R sambauser /srv/share
```

#### 3. 定期检查 Samba 日志
日志默认位于 `/var/log/samba/`。故障时先查看 `log.smbd` 和针对客户端的 `log.<IP>`。

#### 4. 对生产环境启用 SMB 加密
在 `[global]` 中添加：

```ini
smb encrypt = required
```

防止中间人攻击（会略微影响性能）。

#### 5. 配合 SELinux/AppArmor
如果启用了 SELinux（如 CentOS），必须设置正确的上下文：

```bash
sudo semanage fcontext -a -t samba_share_t "/srv/share(/.*)?"
sudo restorecon -Rv /srv/share
```

---

## 6. 故障排除

### 6.1 无法从 Windows 访问共享

**错误示例**：Windows 提示“找不到网络路径”或“0x80070035”。

**原因**：防火墙未开放 Samba 端口，或 SMB 协议版本不兼容（Windows 10 默认禁用 SMB1）。

**解决**：
1. 检查 Linux 防火墙：
   ```bash
   sudo ufw allow 'Samba'
   sudo ufw reload
   ```
2. 确保 `smbd` 正在运行：
   ```bash
   sudo systemctl restart smbd
   sudo systemctl status smbd
   ```
3. 在 Windows 中启用 SMB1（不推荐）或确保服务器支持 SMB2/3。新版 Samba 默认支持 SMB3。

### 6.2 登录时反复提示密码

**原因**：Samba 用户密码未设置，或客户端缓存了错误的凭据。

**解决**：
- 使用 `smbpasswd -a 用户名` 重置密码。
- 在 Windows 中清除凭据：`控制面板 → 凭据管理器 → Windows 凭据 → 删除相关条目`。

### 6.3 写入文件提示权限不足

**错误提示**：在 Windows 中复制文件时报“需要权限”。

**原因**：Linux 目录的本地权限不允许 Samba 进程写入。

**解决**：
- 确保共享目录的属主与 `force user` 一致，或开放写权限。
- 检查目录权限示例：
  ```bash
  sudo chown -R nobody:nogroup /srv/share   # 若 guest 使用 nobody
  sudo chmod 777 /srv/share                 # 临时测试
  ```
  生产环境根据 `force user` 精确设置。

### 6.4 中文文件名显示乱码

**原因**：Windows 使用 GBK 编码，Linux 使用 UTF-8，Samba 默认自动转码。若出现乱码，通常是因为客户端连接时未协商字符集。

**解决**：在 `[global]` 中指定统一字符集：

```ini
unix charset = UTF-8
display charset = UTF-8
dos charset = CP936      # 中国大陆 Windows 使用 CP936（GBK）
```

重启 Samba 后重新访问。

---

## 7. 总结

Samba 是一款成熟、稳定且功能强大的跨平台文件共享解决方案，通过实现 SMB/CIFS 协议，让 Linux 服务器能够无缝融入 Windows 网络环境。它的核心优势在于原生 Windows 集成、灵活细致的权限控制以及企业级的并发性能。

掌握 Samba 后，你可以：

- 在家庭局域网中快速搭建媒体共享服务器，无需在每台设备上安装 FTP 客户端。
- 为开发团队创建统一的代码仓库，强制统一文件权限，避免因属主不一致导致的提权问题。
- 在混合操作系统环境中部署备份存储，利用 Windows 自带的文件历史记录或 Robocopy 进行增量备份。
- 通过高级参数（如 `force user`、`create mask`）精确控制每一个新建文件和目录的默认权限。

建议从简单的无密码只读共享开始实践，逐步引入用户认证、权限限制和加密传输，并结合企业实际需求调整性能参数。日常维护中，善用 `testparm` 命令验证配置文件语法，通过日志定位问题，可大幅提高排错效率。

Samba 官方文档和社区资源非常丰富，遇到特定场景（如域控集成、打印机共享）时，可进一步查阅资料深化应用。

---

## 8. 参考资料

- [Samba 官方文档](https://www.samba.org/samba/docs/)
- [Samba HOWTO 集锦](https://www.samba.org/samba/docs/using_samba/toc.html)
- [Ubuntu 社区 Samba 指南](https://ubuntu.com/server/docs/samba-file-server)
- [Arch Wiki: Samba](https://wiki.archlinux.org/title/Samba)