# 网关程序介绍
网关主控程序与转码推流程序

网关主控程序路径：`/opt/WebConfig/mastercontrolserver/mastercontrolserver-0.0.1.jar`

转码推流程序路径：`/home/QMCY/AGWS/QMCY`

一键运行程序命令：`cd /opt && ./start_env.sh`

查看运行情况：

tmux复用终端运行情况：`tmux ls`

```bash
root@ubuntu:~# tmux ls
MASTER: 1 windows (created Mon Sep 22 10:22:57 2025)
QMCY: 1 windows (created Mon Sep 22 10:22:57 2025)
```

主控程序运行查看：`ps -ef | grep -Ei "guard|mastercontrolserver" | grep -v grep`

```bash
root@ubuntu:~# ps -ef | grep -Ei "guard|mastercontrolserver" | grep -v grep
root      321339  321156  0 10:23 pts/1    00:00:01 /bin/sh ./guard_master.sh
root      321361  321339  1 10:23 pts/1    00:04:23 java -jar mastercontrolserver-0.0.1.jar --spring.config.location=./application.properties
```

转码推流程序运行查看：`ps -ef | grep "./QMCY" | grep -v grep`

```bash
root@ubuntu:~# ps -ef | grep "./QMCY" | grep -v grep
root      321327  321161 99 10:22 pts/2    23:21:12 ./QMCY
```



## 网关配置中心平台
登录页面
输入用户名、密码和验证码登录

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758522059226-f89a59f2-2ce2-484a-8004-956d0ece2f9c.png)

网关配置中心首页
网关配置中心左侧为地区分类栏，中间为录入网关配置中心的网关单元

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758523729670-6022f385-14c2-4848-97a6-8b0550944278.png)

创建网关单元
点击新增网关

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758522488912-c70e6e7d-a029-453a-a3f7-11f059609f85.png)

填入设备名称、IP地址和运行主控程序的端口号

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758522420214-3528a632-4829-404b-9e5d-7a3c5cb5651d.png)

## 网关单元
网关单元首页面板信息有设备名称、CPU占用率、内存占用率、网关所属地区、访问地址、注册摄像机数、有效日期、当前的转码推流程序和主控程序版本号

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758523094617-877d194a-4a69-4586-a3ea-208d6536d569.png)

点击进入网关单元，可查看摄像机列表，此界面控制网关数据库摄像机信息的新增、修改和删除，上云平台的摄像机注册上报与取消注册上报（删除已注册的视频点位）

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758523033305-ea9a26a2-3eab-4503-9e7c-91dd818486b3.png)

## 平台管理
视频上云平台与上云网关的互相访问
上云平台与上云网关采用OAuth 2.0互相认证

上云平台访问上云网关：如下发切换高清4M码流推流指令、下发128K长推流指令

上云网关访问上云平台：新增/修改/删除上报摄像机信息、主动获取推流地址并推流

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758532966947-22d8a31f-ba8d-4d26-84f0-16345f6bb962.png)

### 平台管理页面
设置网关要上报的视频上云平台，展示信息：平台名称、云平台地址、平台编号、平台User、平台Token、平台密钥、网关编号、网关User、网关Token、网关密钥

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758523428062-91e2b4de-506a-49e6-ae59-13a9d6aa90d1.png)

#### 新增上云平台信息
点击新增平台，编辑云平台必填信息：平台名称、云平台地址、平台编号、平台User、平台Token、平台私钥。

> 点击确定后，网关会生成自己的网关。

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758523551737-33701c42-707d-4191-b9f4-0a385dc0bea5.png)

可在网关配置中心首页的网关单元导出网关授权信息为excel表格

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758524836005-0ce0629e-c525-4afa-894d-35d2a7a5930a.png)



## 网关摄像机信息管理
### 新增/修改摄像机信息
分为单条新增与批量新增，推荐使用批量新增功能，此功能可代替批量修改摄像机的功能

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758522998443-25d8cf46-9ee8-4051-8195-e617cfe9323b.png)

批量新增需上传xlsx或xls格式的文件

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758524984921-61a543ab-b074-439e-ad12-07efaa716ce4.png)

上云网关所需的摄像机信息信息如下

| 字段名称 | 值域 |
| --- | --- |
| 通道号 | 内容与子码流播放地址相同 |
| 管辖单位 | 如：广东省高速公路有限公司京珠北分公司 |
| 摄像机编号 | 采用UUID4规则，如：aef89ecc-4cc6-4a6d-80fd-283196105b0b |
| 摄像机名称 | G4(京港澳高速)京珠北路段广东省韶关市广东省高速公路有限公司<br/>京珠北分公司K1844+158(粤北收费站) |
| 经纬度 | 格式：经度/纬度，如：130.34567/29.346587 |
| 摄像机所在路线(编号) | 如G4 |
| 摄像机位置类型 | 0：默认<br/>1：道路沿线<br/>2：桥梁<br/>3：隧道<br/>4：收费广场<br/>5：收费站<br/>6：服务区<br/>7：ETC门架<br/>8：移动视频源 |
| 摄像机方向 | 0：上行<br/>1：下行<br/>2：双向 |
| 摄像机桩号 | 请严格按照如下格式：K100+100 |
| 行政区划代码 | 采用最新的民政部行政区划代码标准(到区县级)，如411528 |
| 摄像机所属路段 | 例如：京珠北段 |
| 摄像机类型 | 1:监控型枪机<br/>2:监控型球机<br/>3:全景型<br/>4:抓拍型 |
| 视频上云网关编号 | 平台线下下发，如ZM1200202005290008 |
| 兴趣点名称 | 道路摄像机上传道路名称<br/>如“京沪高速道路”<br/>如“苏通大桥”<br/>如“九华山隧道”<br/>如：“马群收费站”等 |
| 其他参数 | 扩展预留字段，如果该点位通过IPCamera方式取流，该字段填写IP |
| 子码流通道 | 子码流RTSP协议取流地址，如：rtsp://... |
| 主码流通道 | 主码流RTSP协议取流地址，如：rtsp://... |
| 主码流通道转码 | 通常填1 |
| 子码流通道转码 | 通常填1 |
| 云控类型：SDK、国标、onvif | onvif/国标:gb/sdk |
| 接入网关/NVR国标ID | 接入网关/NVR国标ID |
| 相机国标ID | 相机国标ID |
| 相机品牌 | 海康:hk/宇视:ys |
| 云台控制ip | 云台控制ip |
| 云台控制用户名 | 云台控制用户名 |
| 云台控制密码 | 云台控制密码 |
| 云台控制端口 | 云台控制端口 |


:::warning
注意事项：管辖单位（department）值与摄像机所属路段（roadSec）值需要和云平台的组织名称一致。

:::

删除摄像机信息
可删除单个摄像机、也可勾选多路摄像机批量删除

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758525511049-fe1ad4a2-5bc7-4cf2-8eb4-9f184ef57616.png)

网关上报摄像机信息至云平台
支持勾选多路摄像机上报也可一键全部上报。接收数据上报方提供的摄像机列表信息，如果摄像机存在则对信息进行修改，否则新增。

平台接收的数据信息

|  | 数据元名称 | 参数名 | 值域 | 必填 |
| :--- | :--- | :--- | :--- | :---: |
| 1 | 摄像机所属单位名称 | department |  | √ |
| 2 | 摄像机编号 | cameraNum | 采用 UUID4 规则 | √ |
| 3 | 摄像机名称 | cameraName |  | √ |
| 4 | 摄像机经度、纬度 | longAndLati | 格式：经度/纬度；如： 130.34567/29.346587 | √ |
| 5 | 摄像机所在路线 | road | G42&S38 | √ |
| 6 | 摄像机位置类型 | classify | 0：默认;1：道路沿线;2：桥梁;3：隧道; 4：收费广场5：收费站 6： 服务区; 7 ：ETC 门架;8：移动视频源; | √ |
| 7 | 摄像机方向 | cameraOrientation | 0：上行（桩号数字由小到大方向） 1 ：下行（桩号数字由大到小方 向）2：上下行（双向） | √ |
| 8 | 摄像机桩号 | pileNum | 格式： K100+100 | √ |
| 9 | 行政区划代码 | area | 采用最新的民政部行政区划代码标准 | √ |
| 10 | 摄像机所在路段 | roadSec | 例如：宁镇管理处 | √ |
| 11 | 摄像机类型 | cameraType | 1:监控型枪机2:监控型球机3:全景型4:抓拍型 | √ |
| 12 | 视频上云网关编号 | gatewayNum | “路段-部机云平台”对接方式时使用，该值由平台提供 |  |
| 13 | 其他参数 | others | 扩展预留字段 |  |


:::warning
注意事项：管辖单位（department）值与摄像机所属路段（roadSec）值需要和云平台的组织名称一致。

:::

### 网关删除上报摄像机信息至云平台
支持勾选多路摄像机取消上报也可一键全部取消上报，也可直接删除摄像机信息自动触发向云平台取消上报摄像机信息

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758527128193-bdf4f9a6-2f8c-4661-8b8f-bf9c55d10935.png)

# 视频点位离线原因排查
## 全部离线
1. 查看服务器IP，默认网关，公网IP是否正常
2. 路由和推流域名检查
3. 转码推流程序内部运行情况查看

### 查看服务器IP、默认网关、公网连接是否正常
使用`ifconfig`或`ip addr show`查看服务器的网口情况与IP情况，排除IP异常或网口状态异常。

使用`route -n`查看默认网关，必要时编辑配置文件`/etc/netplan/00-installer-config.yaml`修复异常

公网连接检测，使用ping、traceroute、nmap检测

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758529126337-133aeb49-f805-4d43-9623-99f1e1f30e3b.png)

### 路由和推流域名检查
下级摄像机或流媒体的服务器IP的路由丢失，编写`/etc/netplan/00-installer-config.yaml`并执行`netplan apply`生成永久路由不丢失

利用`traceroute`和`mtr`工具检测CDN推流节点的丢包位置

```bash
root@ubuntu:~# mtr -rn -c 30 jsgs-push.jchc.cn
Start: 2025-09-22T16:27:54+0800
HOST: JS-ninghu-nanbutongdao-spsy Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- ???                       100.0    30    0.0   0.0   0.0   0.0   0.0
  2.|-- 10.137.255.33              0.0%    30    0.2   0.2   0.1   0.4   0.0
  3.|-- 223.112.147.209            0.0%    30    1.5   2.0   1.5   3.3   0.5
  4.|-- 218.206.108.205            0.0%    30    2.6   2.7   2.3   3.9   0.4
  5.|-- ???                       100.0    30    0.0   0.0   0.0   0.0   0.0
  6.|-- ???                       100.0    30    0.0   0.0   0.0   0.0   0.0
  7.|-- 221.183.58.70             40.0%    30   14.8  15.0  14.8  15.8   0.3
  8.|-- 111.6.222.254              0.0%    30   19.7  20.0  19.6  21.2   0.4
  9.|-- 111.7.184.18              53.3%    30   20.0  19.9  19.8  20.4   0.2
 10.|-- ???                       100.0    30    0.0   0.0   0.0   0.0   0.0
 11.|-- ???                       100.0    30    0.0   0.0   0.0   0.0   0.0
 12.|-- 111.6.234.248              0.0%    30   21.4  21.0  20.7  21.9   0.3
```



### 主控程序日志查看与转码推流程序内部运行情况查看
主控程序日志路径：`/opt/WebConfig/mastercontrolserver/logs/mastercontrolserver/mastercontrolserver_info.log`

转码推流程序日志路径：`/home/QMCY/AGWS/log/status.log`

转码程序状态详情接口：`. /home/QMCY/AGWS/status.sh`

![](https://cdn.nlark.com/yuque/0/2025/png/34489707/1758530357365-62d25777-2f77-43f3-be06-78ff2af1a614.png)

## 部分离线
1. 推流域名检查
2. 都是同一网段检查路由是否丢失，添加路由
3. 精细化排查——如何用脚本检查摄像机IP和摄像机拉流地址，对应的返回结果情况枚举说明

### 推流域名检查
同上

### 都是同一网段检查路由是否丢失，添加路由
同上

### 精细化排查——如何用脚本检查摄像机IP和摄像机拉流地址，对应的返回结果情况枚举说明
批量检测IP脚本，检测ping大包丢包率

```bash
#!/bin/bash
#多线程同时ping主机IP地址：
> ping_result.log

pingduoduo(){
    local ip=$1
    # ping 15次，间隔1秒，超时1秒，包大小16000字节
    local loss=$(ping -c15 -i1 -W1 -s16000 "$ip" | grep "packet loss" | awk -F 'received, ' '{print $2}' | awk -F ' packet loss' '{print $1}')
    echo "$ip packet loss:$loss" >> ping_result.log
}
main(){
    # 读取ip_hosts文件的每一行，作为IP地址
    for i in $(cat ip_hosts); do
        pingduoduo "$i" &
    done
    # 等待所有后台任务完成
    wait
    cat ping_result.log
}
# 调用主函数执行
main
```

```bash
root@JS-yanjiang-spsygw-10-105-159-4:/opt/ping_ips# cat ip_hosts 
10.131.9.27
10.131.8.27
10.131.7.5
10.105.14.70
10.105.14.71
10.105.14.75
10.105.13.182
root@JS-yanjiang-spsygw-10-105-159-4:/opt/ping_ips# ./ping_ips.sh 
10.131.8.27 packet loss:0%
10.131.9.27 packet loss:0%
10.131.7.5 packet loss:0%
10.105.14.70 packet loss:93.3333%
10.105.14.75 packet loss:100%
10.105.14.71 packet loss:100%
10.105.13.182 packet loss:100%

```

批量检测摄像机RTSP取流地址

```bash
#!/bin/bash

# 定义文件名称
RTSP_FILE="rtspAddress.txt"
OUTPUT_FILE="output.txt"
# 清空output.txt文件
: > "$OUTPUT_FILE"

check_ffmpeg() {
    # 检查ffmpeg是否已安装
    if ! which ffmpeg >/dev/null 2>&1; then
        echo "未检测到ffmpeg，请安装ffmpeg静态构建版"
        echo "ffmpeg静态构建版下载: wget https://johnvansickle.com/ffmpeg/builds/ffmpeg-git-amd64-static.tar.xz"
    fi
}


run_ffmpeg_test() {
    rtsp="$1"
    # 运行ffmpeg
    video_info=`ffmpeg -timeout 5000000 -i "$rtsp" 2>&1`

    # 用case判断video_info中的多种情况
    case $video_info in
        # 如果包含“failed:”，则拉流失败，获取失败原因
        *failed:*)
            result=`echo "$video_info" | grep "failed: " | awk -F': ' '{print$NF}'`
	          echo "$rtsp failed, $result"
            echo "$rtsp failed, $result" >> "$OUTPUT_FILE"
            ;;

        # 如果包含"Video:"，说明有该rtsp流地址可用，打印分辨率和编码格式信息
        *Video:*)
        # 提取编码格式
            encoding=`echo "$video_info" | grep "Video: " | awk '{print $4}' | awk '{print $1}'`
            # 提取分辨率
            width=`echo "$video_info" | grep "Video:" | awk -F 'x' '{print$1}' | awk -F ', ' '{print $NF}'`
            height=`echo "$video_info" | grep "Video:" | awk -F 'x' '{print$2}' | awk -F ', ' '{print $1}'`
            # 输出结果到output.txt文件
            echo "$rtsp succeed, encoding: ${encoding}, resolution: ${width}x${height}"
            echo "$rtsp succeed, encoding: ${encoding}, resolution: ${width}x${height}" >> "$OUTPUT_FILE"
            ;;

        # 如果以上情况都不包含，则输出最后一个": "后面的内容
        *)
            result=`echo "$video_info" | grep -n ': ' | tail -n1 | awk -F ": " '{print $NF}'`
            echo "$rtsp failed, $result"
            echo "$rtsp failed, $result" >> "$OUTPUT_FILE"
            ;;
    esac
}

main() {
    check_ffmpeg
    for i in $(cat $RTSP_FILE); do
        run_ffmpeg_test "$i"
    done
    wait
    printf "脚本运行结束，请打开%s查看运行结果\n" "$OUTPUT_FILE"
}

# 运行主程序
main
```

```bash
root@JS-yanjiang-spsygw-10-105-159-4:/opt/test-RTSP-stream2.0# cat rtspAddress.txt 
rtsp://admin:Dnzn_sz_2025@10.131.8.7:554/Streaming/Channels/102
rtsp://admin:YJgs@ETC168@10.105.13.176:554/Streaming/Channels/102
rtsp://admin:YJgs@ETC168@10.105.13.178:554/Streaming/Channels/102
root@JS-yanjiang-spsygw-10-105-159-4:/opt/test-RTSP-stream2.0# ./testRTSPstream.sh 
rtsp://admin:Dnzn_sz_2025@10.131.8.7:554/Streaming/Channels/102 succeed, encoding: h264, resolution: 704x576
rtsp://admin:YJgs@ETC168@10.105.13.176:554/Streaming/Channels/102 failed, Connection timed out
rtsp://admin:YJgs@ETC168@10.105.13.178:554/Streaming/Channels/102 failed, 401 Unauthorized
脚本运行结束，请打开output.txt查看运行结果
```

ffmpeg播放RTSP协议流媒体地址返回信息枚举表好的，这是根据您提供的信息总结的RTSP连接常见错误及原因表格：

| 错误信息 | 可能原因 | 解决建议 |
| :--- | :--- | :--- |
| **Connection timed out** | 网络IP不通、端口不通或目标主机的RTSP服务完全无响应。 | 检查IP地址、端口号、网络连通性、防火墙以及目标服务是否启动。 |
| **Connection refused** | 目标主机已收到请求，但明确拒绝连接该RTSP服务端口。 | 确认端口号是否正确、目标主机防火墙规则是否放行、服务是否仅监听本地地址。 |
| **401 Unauthenticated** | 鉴权失败。用户名或密码不正确；或多次尝试失败导致源IP被临时锁定。 | 核对登录凭证，等待锁定解除，或检查服务器鉴权配置。 |
| **5XX Server Error** | 服务器内部错误。常见原因是RTSP服务的最大连接数被占满（如多个设备同时在取流）。 | 减少并发取流数，或增加服务器/设备允许的最大连接数限制。 |
| **404 Not Found** | 连接已建立，但请求的媒体流路径（URL）不存在或该通道无视频流。 | 检查RTSP URL路径（如`/Channels/102`）对应序号码流和通道号是否开启。 |
| **Invalid data found...** | 与404类似，表示能连接到服务但请求不到有效的视频流数据，导致FFmpeg超时中断。 | 确认通道是否存在、是否有视频信号、以及URL格式是否正确。 |


