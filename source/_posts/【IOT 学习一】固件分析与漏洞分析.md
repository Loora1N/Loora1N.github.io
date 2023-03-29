---
title: 【IOT 学习一】固件分析与漏洞利用 
index_img: img/article/iot-1/image-20220914135931400.png
excerpt: 本文内容是基于《物联网渗透测试》-美·亚伦古兹曼的学习总结
---

> 本文内容是基于《物联网渗透测试》-美·亚伦古兹曼的学习总结

## 固件提取

- 在硬件厂商网站内直接下载

- 固件更新时，利用中间人攻击，转发代理流量

- 通过物理连接，直接从设备转储固件。固件存储为bin文件指令:

   `$ flashrom -p ft2232_spi:type=232H -r spidump.bin`

- 利用搜索引擎搜集固件，还可以利用 [Google Hacking数据库](https://www.exploit-db.com/google-hacking-database)搜索固件或设备

![image-20220914135931400](img/article/iot-1/image-20220914135931400.png)

### MITM转发代理流量流程

> 接下来，我们简要介绍了如何使用Kali Linux、SSLstrip、 Ettercap和Wireshark搭建MITM测试环境，以在设备更新期间捕获设备 流量

1. 启用IP转发功能`echo 1 > /proc/sys/net/ipv4/ip_forward`

2. 配置iptables，将目的端口80的流量重定向至SSLstrip监听的端口10000。`iptables -t nat -p tcp -A PREROUTING --dport 80 -j REDIRECT --to-port 10000`

3. 启动SSLstrips: `sslstrip -a`

4. 启动Ettercap GUI：`ettercap -G`


![image-20220914134344716](img/article/iot-1/image-20220914134344716.png)

5. 在`primary interface`选择要监听的设备，这里选择网卡`eth0`
6. 启动Wireshark程序，对`eth0`进行抓包

![image-20220914135308518](img/article/iot-1/image-20220914135308518.png)

7. 根据实际需求过滤关注的报文

## 固件文件分析

> 一般直接使用 `binwalk -eM file.bin` 即可分解文件

### D-Link 923B固件文件系统获取

> 固件压缩包[下载链接](https://media.dlink.eu/ftp/products/dwr/dwr-932/driver_software/DWR-932_fw_revB_2_02_eu_en_20150709.zip)，版本号为DWR-932_fw_revB_2_02_eu_en_20150709。安全研究员Gianni Carabelli和Pierre Kim已经在这款固件中挖掘出了漏洞，这里我们只是用于练习

首先是从固件中提取文件系统，内有加密过的zip压缩包，可以使用`fcrackzip`进行密码爆破，这里省略爆破过程，直接输入密码 **beUT9Z**

![压缩包已加密](img/article/iot-1/image-20220914145816127.png)

接下来可以使用针对yaffs2格式 的工具解压文件系统来提取2K-mdm-image-mdm9625.yaffs2文件，也可以直接通过Binwalk来完成提取工作。

> 这里我采用的是unyaffs，下载链接 [Google Code Archive - Long-term storage for Google Code Project Hosting.](https://code.google.com/archive/p/unyaffs/downloads)

下载可执行程序后，使用`./unyaffs file.yaffs2`指令即可

![image-20220914192637485](img/article/iot-1/image-20220914192637485.png)

![解压出的文件系统](img/article/iot-1/image-20220914192719147.png)



### 手动信息挖掘

然后我们需要在这个文件目录中查找有用的信息，如下

```bash
find . -name "*.conf"
find . -name "shadow"
find . -name "passwd"
find . -name "*config*"
find . -name "*history*"
find . -name "*ssh*_config*"
find . -name "*ssh_host*"
```

![搜索示例](img/article/iot-1/image-20220914194404354.png)

在`inadyn-mt.conf `文件中存储了路由器的no-IP配置信息，包括用于访问网站 [no-ip](https://www.no-ip.com)的用户名和口令

![image-20220914195009565](img/article/iot-1/image-20220914195009565.png)

查看shadow文件

![image-20220914195346782](img/article/iot-1/image-20220914195346782.png)

查看passwd文件

![image-20220914195118433](img/article/iot-1/image-20220914195118433.png)



### 利用Firmwalker搜索敏感信息

安装firmwalker

```bash
npm i -g eslint                                                                       

git clone https://github.com/craigz28/firmwalker.git 
```

使用方式

```bash
./firmwalker.sh {floder path}
```

> Firmwalker脚本能够识别出多种敏感信息，包括二进制文件、 证书、IP地址、私钥等，同时将输出结果都保存在**firmwalker.txt**文 件之中

![image-20220914202334231](img/article/iot-1/image-20220914202334231.png)

![firmwalker.txt](img/article/iot-1/image-20220914204146429.png)



### 查看启动项

```bash
cd /etc/init.d
```

![image-20220914214159035](img/article/iot-1/image-20220914214159035.png)

这里可以查看 **start_appmgr**脚本，一般来说mgr就是主控程序的意思  

![image-20220914214951502](img/article/iot-1/image-20220914214951502.png)

可以看到该脚本在开机时会以服务的形式运行`/bin/appmgr`程序，同时该脚本还会开启**telnet**服务

![image-20220914215128332](img/article/iot-1/image-20220914215128332.png)



### appmgr分析

用IDA PRO打开`/bin/appmg`程序进行逆向分析，可以发现有一个线程会持续监听 0.0.0.0:39889（UDP）（这个IP和端口还是没有确定出来），并等待传入控制命令,如果某个用户向目标路由器发送了一个 `HELODBG` 字符串，那么路由器将会执行 `/sbin/telnetd -l /bin/sh` ，并允许这名用户在未经身份验证的情况下以 root 用户的身份登录路由器。

> 这里的-11877转换二进制后，末尾两字节为port，前面都是0，末尾字节为**\xD1\x9B**注意小段存储方式，则port号为**9BD1**即**39889**

![image-20220914223737117](img/article/iot-1/image-20220914223737117.png)

![image-20220914223809561](img/article/iot-1/image-20220914223809561.png)

搜索 **mod_sysadm_config_passwd** 函数，可以得到admin默认密码

![image-20220914231714471](img/article/iot-1/image-20220914231714471.png)

路由器的管理员账号。设备的管理员账号默认为“admin”，而密码同样也是“admin”。

搜索 **wifi_get_default_wps_pin** 函数

![image-20220914231829944](img/article/iot-1/image-20220914231829944.png)

默认配置下，该路由器 WPS 系统的 PIN 码永远都是 `28296607` 因为这个 PIN 码是硬编码在 `/bin/appmgr` 程序中

### fotad 分析

路由器与 FOTA 服务器进行通信时的凭证数据硬编码在 `/sbin/fotad` 代码中，我们用 IDA 进行分析

搜索 `sub_CAAC `函数,可以发现被 base64 过的凭证

![image-20220914232237866](img/article/iot-1/image-20220914232237866.png)

用户/密码如下

```
cWRwYzpxZHBj        qdpc:qdpc
cWRwZTpxZHBl        qdpe:qdpe
cWRwOnFkcA==        qdp:qdp
```



### UPnP安全问题

UPnP 允许用户动态添加防火墙规则。因为这种做法会带来一定的安全风险，因此设备通常都会对这种操作进行限制，以避免不受信任的客户端添加不安全的防火墙规则。

UPnP 的不安全性早在2006年就已经是众所周知的事情了。而该路由器中 UPnP 程序的安全等级仍然非常的低，处于局域网内的攻击者可以随意修改路由器的端口转发规则。

文件 `/var/miniupnpd.conf` 是由 `/bin/appmgr` 程序生成的：

搜索 sub_2AE0C 函数

![image-20220914232710172](img/article/iot-1/image-20220914232710172.png)

该程序会生成 `/var/miniupnpd.conf`：

```
ext_ifname=rmnet0
listening_ip=bridge0
port=2869
enable_natpmp=yes
enable_upnp=yes
bitrate_up=14000000
bitrate_down=14000000
secure_mode=no      # "secure" mode : when enabled, UPnP client are allowed to add mappings only to their IP.
presentation_url=http://192.168.1.1
system_uptime=yes
notify_interval=30
upnp_forward_chain=MINIUPNPD
upnp_nat_chain=MINIUPNPD
```



## 基于固件仿真的动态分析

### 准备工作

安装**FAT(Firmware-Analysis-Toolkit)**，执行以下命令即可，但需注意环境需同时具有**python2**和**python3**

```bash
git clone https://github.com/attify/firmware-analysis-toolkit
cd firmware-analysis-toolkit
./setup.sh
```

> Kali和Ubuntu安装都出现了一些问题人麻了

![Ubuntu安装错误](img/article/iot-1/image-20220914212925067.png)

Kali则是报找不到**Package 'lsb-core' has no installation candidate**