yum源配置

<font style="color:rgb(221, 74, 104);">sudo cp</font><font style="color:rgb(0, 0, 0);background-color:rgb(250, 250, 250);"> /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak</font>

<font style="color:rgb(0, 0, 0);background-color:rgb(250, 250, 250);">sudo wget -O /etc/yum.repos.d/CentOS-Base.repo </font>[http://mirrors.aliyun.com/repo/Centos-7.repo](http://mirrors.aliyun.com/repo/Centos-7.repo)

<font style="color:rgb(0, 0, 0);background-color:rgb(250, 250, 250);"></font>

<h4 id="1fa376aa"><font style="color:rgb(79, 79, 79);">清理 YUM 缓存</font></h4>
<font style="color:rgb(221, 74, 104);">sudo</font><font style="color:rgb(0, 0, 0);background-color:rgb(250, 250, 250);"> yum clean all</font>

<font style="color:rgb(221, 74, 104);">sudo</font><font style="color:rgb(0, 0, 0);background-color:rgb(250, 250, 250);"> yum makecache</font>

<font style="color:rgb(0, 0, 0);background-color:rgb(250, 250, 250);"></font>

<h4 id="b791dae0"><font style="color:rgb(79, 79, 79);">验证新源是否可用</font></h4>
sudo yum repolist



ntp安装

yum install ntp安装后，后台会启动一个非systemctl启动的ntp，

```bash
[root@localhost sysconfig]# ps -ef | grep ntp
root       5421      1  0 10:31 ?        00:00:00 ntpd
root      62267   3326  0 13:32 pts/1    00:00:00 grep --color=auto ntp
```

需要先kill掉进程号后，再启动`systemctl start ntpd`

```bash
[root@localhost sysconfig]# systemctl start ntpd
[root@localhost sysconfig]# systemctl status ntpd
● ntpd.service - Network Time Service
   Loaded: loaded (/usr/lib/systemd/system/ntpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2025-10-30 13:33:14 CST; 1s ago
  Process: 62280 ExecStart=/usr/sbin/ntpd -u ntp:ntp $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 62282 (ntpd)
    Tasks: 1
   CGroup: /system.slice/ntpd.service
           └─62282 /usr/sbin/ntpd -u ntp:ntp -g

Oct 30 13:33:14 localhost.localdomain ntpd[62282]: 0.0.0.0 c01d 0d kern kernel time sync enabled
Oct 30 13:33:14 localhost.localdomain ntpd[62282]: ntp_io: estimated max descriptors: 1024, initial socket boundary: 16
Oct 30 13:33:14 localhost.localdomain ntpd[62282]: Listen and drop on 0 v4wildcard 0.0.0.0 UDP 123
Oct 30 13:33:14 localhost.localdomain ntpd[62282]: Listen and drop on 1 v6wildcard :: UDP 123
Oct 30 13:33:14 localhost.localdomain ntpd[62282]: Listen normally on 2 lo 127.0.0.1 UDP 123
Oct 30 13:33:14 localhost.localdomain ntpd[62282]: Listen normally on 3 ens33 192.168.60.5 UDP 123
Oct 30 13:33:14 localhost.localdomain ntpd[62282]: Listen normally on 4 virbr0 192.168.122.1 UDP 123
Oct 30 13:33:14 localhost.localdomain ntpd[62282]: Listen normally on 5 lo ::1 UDP 123
Oct 30 13:33:14 localhost.localdomain ntpd[62282]: Listen normally on 6 ens33 fe80::2aa9:b9ba:3e66:9202 UDP 123
Oct 30 13:33:14 localhost.localdomain ntpd[62282]: Listening on routing socket on fd #23 for interface updates

```

ntpstat查看与上游服务器是否同步

```bash
[root@localhost sysconfig]# ntpstat
synchronised to NTP server (203.107.6.88) at stratum 3
   time correct to within 210 ms
   polling server every 64 s

```



查看时间同步的状态 ntpd -p

```bash
[root@localhost sysconfig]# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*203.107.6.88    100.107.25.114   2 u   23   64    1   19.645    4.105   0.087
+111.230.189.174 100.122.36.196   2 u   22   64    1   31.608   -1.412   0.068
+139.199.215.251 100.122.36.196   2 u   21   64    1   33.723    0.289   0.123
-stratum2-1.ntp. 130.173.91.58    2 u   20   64    1  196.848  -37.950   0.978
 a.chl.la        131.188.3.222    2 u   19   64    1  175.292   11.037   0.860

```

