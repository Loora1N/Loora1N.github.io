---
title: 【IOT PWN】 环境搭建
index_img: /img/article/iot-envir/image-20220913113915723.png
excerpt: IOT PWN所需的硬件和软件环境：Binwalk，Firmadyne...
---
## 硬件资源

太多辣，以后看需求慢慢购入吧

- 万用表，主要的功能是用来测试UART(TLL)中的几个接口，比如`RX`、`TX`、`GND`。
- CH340G设备(USB转TTL)，这个设备主要的将电脑与IoT设备用TTL线连接，方便进入IoT设备的终端(Shell)。
- 电烙铁，这里买电烙铁的主要用途是焊接UART接口的针脚。
- 焊锡丝
- 针脚，针脚主要用来焊接UART接口
- 编辑器，编程器主要是用来帮助我们dump IoT设备上例如`闪存芯片`里面的数据，一般都为固件包。编程器的话一般买CH341A够用了，基本上8脚的芯片都支持。如果富裕的话编程器可以买个爱修的RT80H编程器，或者RT809F也不错。
- 热风枪，这个其实可有可无，相对于技术比较高的人用，因为热风枪主要是吹出芯片然后放到编程器上面用来dump估计，但是一般来说用夹子就可以把固件dump出来。
- 测试夹、探针，这个的主要用途是省的我们去焊接UART接口的针脚了，直接用这夹子加上去用CH340G设备连接即可，不过UART接口要规则才行，要一排的那种。
- 免拆芯片通用测试夹，这设备主要也是帮助用来dump芯片固件的，我们买的编程器夹子一般只能用来夹8脚，当超过8脚就不好用了，所以可以用这种通用测试夹来夹住芯片然后dump估计。



## 环境搭建

### Binwalk

binwalk是一个固件解包的工具，当我们用编程器dump出一个固件用，需要用binwalk来解压。一般做misc也会经常用到。

```bash
sudo apt install binwalk
```

![image-20220913113915723](/img/article/iot-envir/image-20220913113915723.png)

固件解包命令

```bash
binwalk -Me file.bin
```

家里旧路由器解包出来的文件，太多了没截全

![家里旧路由器的固件](/img/article/iot-envir/image-20220913114057712.png)

### Firmadyne（固件仿真）

这工具主要是用来仿真，将固件用qemu模拟启动起来，不过不是百分百模拟成功的，经常会仿失败，常见就是环境等问题。(建议还是买真机好)

> 安装可参考github上的文章：https://github.com/firmadyne/firmadyne#introduction

#### 安装

```bash
sudo apt-get install busybox-static fakeroot git dmsetup kpartx netcat-openbsd nmap python-psycopg2 python3-psycopg2 snmp uml-utilities util-linux vlan
git clone --recursive https://github.com/firmadyne/firmadyne.git
```

#### 重新构建binwalk

```bash
git clone https://github.com/ReFirmLabs/binwalk.git
cd binwalk
sudo ./deps.sh
sudo python3 ./setup.py install
```

#### 数据库安装

```bash
sudo apt-get install postgresql
sudo -u postgres createuser -P firmadyne，带密码firmadyne
sudo -u postgres createdb -O firmadyne firmware
sudo -u postgres psql -d firmware < ./firmadyne/database/schema
```

#### 设置Firmadyne

替换config文件中的绝对文件路径

![image-20220913122554653](/img/article/iot-envir/image-20220913122554653.png)

构建二进制文件

```bash
cd ./firmadyne; ./download.sh
```

安装所需的其他`qemu`相关依赖

```bash
sudo apt-get install qemu-system-arm qemu-system-mips qemu-system-x86 qemu-utils
```



## 仿真Netgear(网络路由器) WNAP320测试

#### 固件包下载

```bash
wget http://www.downloads.netgear.com/files/GDC/WNAP320/WNAP320%20Firmware%20Version%202.0.3.zip

mv WNAP320\ Firmware\ Version\ 2.0.3.zip WNAP320.zip  
```

![image-20220913133114466](/img/article/iot-envir/image-20220913133114466.png)

#### 解压固件包

```bash
sudo python3 ./sources/extractor/extractor.py -b Netgear -sql 127.0.0.1 -np -nk "WNAP320.zip" images

#参数解释
-b   "brand 品牌"
-sql "连接本地数据库"
-np  "代表没有并行操作"
-nk  "代表不提取内核"
```

> 如出现No module named 'magic'错误，可以尝试使用pip3 install python-magic解决

![image-20220913133817467](/img/article/iot-envir/image-20220913133817467.png)

#### 识别CPU框架

接着是执行`./script/getArch.sh`脚本来获取路由器固件的CPU架构。

```bash
sudo ./scripts/getArch.sh ./images/1.tar.gz
```

焯，这里卡住了. 但是找到了一个issue: [./images/1.tar.gz: Cannot open: No such file or directory · Issue #27 · attify/firmware-analysis-toolkit (github.com)](https://github.com/attify/firmware-analysis-toolkit/issues/27)

暂时不知道怎么解决

![image-20220913143102694](/img/article/iot-envir/image-20220913143102694.png)



