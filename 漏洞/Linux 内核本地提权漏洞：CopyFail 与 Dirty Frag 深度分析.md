# Linux 内核本地提权漏洞：CopyFail 与 Dirty Frag 深度分析

> **文档版本**：v1.0 · 更新日期：2026-05-09  
> **适用对象**：安全工程师、系统管理员、DevSecOps 团队

---

## 漏洞速览对比

| | **CopyFail** | **Dirty Frag** |
|---|---|---|
| CVE 编号 | CVE-2026-31431 | CVE-2026-43284（ESP）<br>CVE-2026-43500（RxRPC） |
| 漏洞代号 | Copy Fail | Dirty Frag / CopyFail 2 |
| **发现者** | **Xint Code（Theori 团队）** | **Hyunwoo Kim（@v4bel）** |
| 披露日期 | 2026-04-29 | 2026-05-07 |
| CVSS 3.1 | 7.8（HIGH） | 7.8（HIGH） |
| 漏洞类型 | 本地权限提升（LPE） | 本地权限提升（LPE） |
| 受影响模块 | `algif_aead`（AF_ALG 加密子系统） | `esp4` / `esp6`（IPsec）、`rxrpc`（AFS） |
| 漏洞存在年限 | 自 2017 年（~9 年） | ESP 自 2017 年，RxRPC 自 2023 年 |
| 公开 PoC | ✅ 已有（Python 脚本，732 字节） | ✅ 已有（含"CopyFail 2: Electric Boogaloo"） |
| 竞争条件依赖 | ❌ 确定性触发 | ❌ 确定性触发 |
| CISA KEV 收录 | ✅ 已收录（联邦机构须于 2026-05-15 前修复） | 暂未收录 |
| ESP 补丁状态 | — | ✅ 主线已合并（commit f4c50a4034e6） |
| RxRPC 补丁 | — | ⚠️ 暂无官方补丁 |

---

## CopyFail（CVE-2026-31431）

### 漏洞简介

CopyFail 于 2026 年 4 月 29 日由安全研究团队 **Theori**（研究员署名 Xint Code）公开披露。漏洞位于 Linux 内核加密子系统的 `algif_aead` 模块，即 `AF_ALG`（内核用户空间加密 API）的 AEAD socket 接口。

该漏洞属于**确定性逻辑缺陷**（非竞争条件），极易利用：无需额外软件依赖，一个 732 字节的 Python 脚本即可将普通用户权限提升至 root，每次运行均可成功，且**不在磁盘留下痕迹**。

### 技术原理

漏洞根源在于 2017 年的 [commit `72548b093ee3`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git)，该 commit 将 AEAD 操作切换为**就地（in-place）处理**模式，引入了一处内存处理逻辑缺陷：

```
AF_ALG AEAD socket
    │
    ├── splice() 将 pipe page 注入 socket 接收路径
    │
    ├── 就地解密时，内核未独占 paged buffer 所有权
    │
    └── 非特权进程保留明文引用
            │
            └── 对 page cache 执行受控 4 字节写入
                    │
                    └── 修改 SUID 二进制文件内存映射 → 获取 root
```

**关键特性**：

- 攻击面：本地（`AV:L`），低权限，无需用户交互
- 不修改磁盘文件，传统文件完整性监控（FIM）无法发现
- 利用路径：`AF_ALG + splice()` → 页缓存污染 → SUID 二进制劫持

### 受影响版本

- **Linux kernel 4.14 ~ 7.0-rc**（2017 年 7 月以来全线受影响）
- 已验证受影响发行版：Ubuntu 24.04、Amazon Linux 2023、RHEL 10.1、SUSE 16
- **不受影响**：Ubuntu 26.04（Resolute）及更新内核

### 漏洞影响场景

| 场景 | 影响 |
|---|---|
| 普通 Linux 主机 | 本地用户提权至 root |
| Kubernetes 集群 | 容器逃逸 + 宿主机 root |
| CI/CD 运行器 | 恶意 CI Job 获取宿主机权限 |
| 云 VM | 低权限账号接管实例 |

### 修复措施

#### 方案一：升级内核（推荐）

| 发行版 | 修复版本 |
|---|---|
| Ubuntu | `linux-image-6.8.0-xx` 及以上（已发布） |
| Debian | 通过 DSA 发布，跟踪 [security-tracker](https://security-tracker.debian.org/tracker/CVE-2026-31431) |
| AlmaLinux | 已发布补丁包 |
| SUSE | 已发布补丁包 |
| Fedora | 已发布补丁包 |
| RHEL | 补丁陆续发布中 |

```bash
# Ubuntu / Debian
sudo apt update && sudo apt upgrade -y linux-image-generic
sudo reboot

# RHEL / AlmaLinux / CentOS Stream
sudo dnf update kernel -y
sudo reboot

# SUSE
sudo zypper update kernel-default
sudo reboot
```

#### 方案二：临时禁用 algif_aead 模块

> ⚠️ 此操作不影响 dm-crypt/LUKS、kTLS、IPsec/XFRM、OpenSSL、GnuTLS、SSH。

```bash
# 立即卸载模块
sudo rmmod algif_aead 2>/dev/null || true

# 持久化禁用（重启后生效）
echo "install algif_aead /bin/false" | sudo tee /etc/modprobe.d/disable-algif-aead.conf

# Ubuntu：更新 initramfs
sudo update-initramfs -u
```

---

## Dirty Frag（CVE-2026-43284 / CVE-2026-43500）

### 漏洞简介

Dirty Frag 于 2026 年 5 月 7 日由研究员 **Hyunwoo Kim（@v4bel）** 公开披露，是 CopyFail **同一漏洞类**（页缓存写入原语）的**独立发现**，但影响的是完全不同的内核子系统。

披露发生在负责任披露协调期结束前——有人对内核补丁 commit 进行逆向工程，提前还原出利用方式并公开，导致 Hyunwoo Kim 不得不提前公开技术细节。公开可用的 PoC 包括两个：原始版本以及 "Copy Fail 2: Electric Boogaloo"（均指向同一漏洞）。

### 技术原理

Dirty Frag 将两个独立的页缓存写入原语组合成完整的提权链：

#### CVE-2026-43284：xfrm-ESP（IPsec）

```
IPsec ESP 接收路径（esp4 / esp6）
    │
    ├── splice()/sendfile() 将 pipe page 注入 socket
    │
    ├── 就地解密时未独占 paged buffer
    │
    └── 非特权进程保留解密后明文引用
            │
            └── 页缓存写入原语 → 提权至 root
```

- 漏洞存在于**2017 年以来**的内核
- 主线补丁：commit `f4c50a4034e6`（2026-05-08 合并）

#### CVE-2026-43500：RxRPC（AFS 文件系统协议）

```
RxRPC 接收路径（rxrpc 模块）
    │
    ├── 相同的 splice() 注入机制
    │
    └── 类似的页缓存写入原语
            │
            └── 与 ESP 路径组合 → 完整提权链
```

- 漏洞存在于**2023 年以来**的内核
- **补丁截至 2026-05-09 仍未合并**

#### 利用条件

- 通常需要 `CAP_NET_ADMIN` 权限（限制了容器场景中的直接利用）
- 在默认配置的 Kubernetes 节点（default seccomp profile）中利用难度较高
- 在普通 VM 或非加固 Linux 主机上，普通用户可一条命令拿到 root

### 受影响范围

| 发行版 | 受影响 CVE |
|---|---|
| Ubuntu（全版本） | CVE-2026-43284 + CVE-2026-43500 |
| RHEL 8 / 9 / 10 | CVE-2026-43284 + CVE-2026-43500 |
| AlmaLinux 8 | 仅 CVE-2026-43284（无 rxrpc 模块） |
| AlmaLinux 9 / 10 | CVE-2026-43500（需安装 kernel-modules-partner） |
| CentOS Stream 10 | CVE-2026-43284 + CVE-2026-43500 |
| Fedora（近期版本） | CVE-2026-43284 + CVE-2026-43500 |
| openSUSE Tumbleweed | CVE-2026-43284 + CVE-2026-43500 |
| OpenShift 4 | 潜在受影响 |

### 修复措施

#### 方案一：升级内核（CVE-2026-43284 已有补丁）

```bash
# Ubuntu / Debian
sudo apt update && sudo apt upgrade -y linux-image-generic
sudo reboot

# RHEL / AlmaLinux / CentOS（AlmaLinux 已发布 kernel-6.12.0-124.55.3.el10_1）
sudo dnf update kernel -y
sudo reboot
```

#### 方案二：临时禁用受影响模块

> ⚠️ 禁用 esp4/esp6 会中断 IPsec 功能；禁用 rxrpc 会影响 AFS 分布式文件系统访问。

```bash
# 立即卸载模块（当前会话）
sudo modprobe -r esp4 esp6 rxrpc 2>/dev/null || true

# 持久化禁用
cat << 'EOF' | sudo tee /etc/modprobe.d/dirty-frag-mitigation.conf
install esp4 /bin/false
install esp6 /bin/false
install rxrpc /bin/false
EOF

# Ubuntu：更新 initramfs
sudo update-initramfs -u

# RHEL/AlmaLinux：重建 initramfs
sudo dracut -f
```

#### 方案三：容器与云环境加固

```yaml
# Kubernetes - SecurityContext 示例（移除 NET_ADMIN）
securityContext:
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

- 确认 Pod 未设置 `CAP_NET_ADMIN`
- 使用 KernelCare 等 Livepatching 方案实现**无需重启**的内核热修复
- 云托管节点（AWS、Azure、GCP）跟踪托管内核更新通知

---

## 两者关系辨析

> **简答：同一漏洞类，不同作者，不同模块，不同时间独立发现。**

| 维度 | 说明 |
|---|---|
| **发现者** | CopyFail 由 Theori 团队（Xint Code）发现；Dirty Frag 由 Hyunwoo Kim（@v4bel）独立发现 |
| **漏洞类（bug class）** | 两者均属于"页缓存写入原语（page-cache write primitive）"同一漏洞类，类似于 Dirty Pipe 历史漏洞 |
| **受影响模块** | CopyFail：`algif_aead`（AF_ALG 加密 API）；Dirty Frag：`esp4/esp6`（IPsec）+ `rxrpc`（AFS） |
| **"CopyFail 2"别名** | Dirty Frag 也被称为 "CopyFail 2" / "Copy Fail 2: Electric Boogaloo"，因其漏洞机制相同，但这仅是社区命名，两者是独立 CVE |
| **披露顺序** | CopyFail（2026-04-29）→ Dirty Frag（2026-05-07），相隔约 8 天 |
| **关联性** | Hyunwoo Kim 参考了 CopyFail 的补丁 commit 进行逆向，发现了同类漏洞在其他内核路径中的存在 |

---

## 修复建议汇总

### 优先级排序

```
立即处理（P0）
├── 升级内核到含补丁版本（重启生效）
├── 若无法立即重启：禁用受影响内核模块
└── Kubernetes 节点：确认 seccomp profile + 移除 CAP_NET_ADMIN

次优先（P1）
├── 部署 Livepatching（KernelCare 等）实现无重启修复
├── 更新 EDR/IDS 检测规则，监控 AF_ALG / splice() 异常
└── 追踪 CVE-2026-43500（RxRPC）官方补丁发布
```

### 一键检查脚本

```bash
#!/bin/bash
echo "=== 内核版本 ==="
uname -r

echo ""
echo "=== algif_aead 模块状态（CopyFail）==="
lsmod | grep algif_aead && echo "⚠️  模块已加载，存在风险" || echo "✅ 模块未加载"

echo ""
echo "=== esp4/esp6/rxrpc 模块状态（Dirty Frag）==="
for mod in esp4 esp6 rxrpc; do
  lsmod | grep -q "^$mod" && echo "⚠️  $mod 已加载" || echo "✅ $mod 未加载"
done

echo ""
echo "=== modprobe 黑名单检查 ==="
for f in /etc/modprobe.d/*.conf; do
  grep -l "algif_aead\|esp4\|esp6\|rxrpc" "$f" 2>/dev/null && echo "  ↑ 已配置黑名单"
done
```

---

## 检测与监控

### Falco / Sysdig 规则（CopyFail）

```yaml
- rule: Unexpected AF_ALG AEAD Socket Usage
  desc: 检测非加密工具进程使用 AF_ALG SEQPACKET socket（CVE-2026-31431 利用必要步骤）
  condition: >
    evt.type = socket and evt.rawres >= 0
    and (evt.arg.domain contains AF_ALG or evt.rawarg.domain = 38)
    and evt.arg.type in (5, 2053, 524293, 526341)
    and not proc.name in (cryptsetup, systemd-cryptsetup, veritysetup, kcapi-enc)
  output: "疑似 CopyFail 利用：%proc.name (pid=%proc.pid uid=%user.uid)"
  priority: CRITICAL
  tags: [CVE-2026-31431, lpe, kernel]
```

### 日志关键字

监控以下内核日志关键字（`dmesg` / `journalctl -k`）：

```
algif_aead
xfrm_input
rxrpc_recvmsg
splice
```

---

## 参考资源

### CopyFail（CVE-2026-31431）

- 官方披露站点：<https://copy.fail>
- Ubuntu 公告：<https://ubuntu.com/blog/copy-fail-vulnerability-fixes-available>
- Theori PoC：<https://github.com/theori-io/copy-fail-CVE-2026-31431>
- NVD 条目：<https://nvd.nist.gov/vuln/detail/CVE-2026-31431>
- CISA KEV：<https://www.cisa.gov/known-exploited-vulnerabilities-catalog>
- Microsoft 分析：<https://www.microsoft.com/en-us/security/blog/2026/05/01/cve-2026-31431-copy-fail-vulnerability-enables-linux-root-privilege-escalation/>
- CERT-EU 公告：<https://cert.europa.eu/publications/security-advisories/2026-005/>

### Dirty Frag（CVE-2026-43284 / CVE-2026-43500）

- oss-security 披露帖：<https://seclists.org/oss-sec/2026/q2/441>
- Ubuntu 公告：<https://ubuntu.com/blog/dirty-frag-linux-vulnerability-fixes-available>
- Red Hat RHSB-2026-003：<https://access.redhat.com/security/vulnerabilities/RHSB-2026-003>
- AlmaLinux 公告：<https://almalinux.org/blog/2026-05-07-dirty-frag/>
- Wiz 技术分析：<https://www.wiz.io/blog/dirty-frag-linux-kernel-local-privilege-escalation-via-esp-and-rxrpc>
- Tenable FAQ：<https://www.tenable.com/blog/dirty-frag-cve-2026-43284-cve-2026-43500-frequently-asked-questions-linux-kernel-lpe>
- ESP 补丁 commit：<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f4c50a4034e6>

---

*本文档基于截至 2026-05-09 的公开信息整理。CVE-2026-43500（RxRPC）补丁状态仍在更新中，建议持续跟踪各发行版安全公告。*
