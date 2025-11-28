## 下载、解压、编译、安装
```bash
cd /mnt
wget https://github.com/openssl/openssl/releases/download/openssl-3.5.4/openssl-3.5.4.tar.gz
tar -zxvf openssl-3.5.4.tar.gz
cd openssl-3.5.4
./config --prefix=/usr/local --openssldir=/usr/ssl shared zlib
make -j$(nproc)
checkinstall --install=no --pkgname=openssl-local
```



```plain
软件包将用下面的值来创建：

0 -  Maintainer: [ root@shiyaofei-virtual-machine ]
1 -  Summary: [ Package created with checkinstall 1.6.3 ]
2 -  Name:    [ openssl-local ]
3 -  Version: [ 3.5.4 ]
4 -  Release: [ 1 ]
5 -  License: [ GPL ]
6 -  Group:   [ checkinstall ]
7 -  Architecture: [ amd64 ]
8 -  Source location: [ openssl-3.5.4 ]
9 -  Alternate source location: [  ]
10 - Requires: [  ]
11 - Recommends: [  ]
12 - Suggests: [  ]
13 - Provides: [ openssl ]
14 - Conflicts: [  ]
15 - Replaces: [  ]
```

## 解压deb包添加脚本
```bash
dpkg-deb -R openssh-local_10.2p1-1_amd64.deb tmp/
cd tmp
```

### 1️⃣ `DEBIAN/postinst`（安装后执行）
```bash
#!/bin/bash
set -e
echo 'export LD_LIBRARY_PATH=/usr/local/lib64${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}' > /etc/profile.d/openssl-local.sh
source /etc/profile
echo "==> 已将/usr/local/lib64加入到LD_LIBRARY_PATH动态链接库"
echo "==> openssl-local_3.5.4-1已安装完成"
exit 0
```

### 2️⃣ `DEBIAN/postrm`（卸载后执行）
```bash
#!/bin/bash
set -e
rm -rf /etc/profile.d/openssl-local.sh
source /etc/profile
exit 0
```

##  🔐 设置脚本权限
```bash
chmod 755 DEBIAN/{postinst,postrm}
```



## 📦 重新打包deb文件
```bash
dpkg-deb -b tmp/ openssl-local_3.5.4-1_amd64.deb
```



