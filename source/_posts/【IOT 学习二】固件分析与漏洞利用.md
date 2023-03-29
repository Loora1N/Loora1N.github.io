---
title: 【IOT 学习二】固件分析与漏洞利用 
index_img: img/article/iot-2/image-20220915191611191-1663245953755-2.png
excerpt: 本文内容是基于《物联网渗透测试》-美·亚伦古兹曼的学习总结
---

> 本文内容是基于《物联网渗透测试》-美·亚伦古兹曼的学习总结

## 基于固件仿真的动态分析

### 前期准备

> 书接上一篇文章的安装错误，在我几乎连续不断的尝试了10余个小时后，环境终于搭建完毕，这里采用Ubuntu20.04，Kali有各种各样奇葩问题是真有点麻烦。
>
> 本篇文章写于2020/9/15，由于时间有效性的存在，本篇文章仅供参考

![运行成功截图](img/article/iot-2/image-20220915191611191-1663245953755-2.png)

#### 项目下载

首先下载下来项目FAT

```bash
git clone https://github.com/attify/firmware-analysis-toolkit
cd firmware-analysis-toolkit
```

**这里注意注意：先打开编辑setup.sh，不是运行不是运行！！！**

先注释掉安装binwalk的部分，我们采用手动编译的方式

![image-20220915191827246](img/article/iot-2/image-20220915191827246.png)

#### 手动编译binwalk

```bash
git clone --depth=1 https://github.com/ReFirmLabs/binwalk.git
cd binwalk
sed -i '/REQUIRED_UTILS="wget tar python"/c\REQUIRED_UTILS="wget tar python3"' deps.sh
```

这时候要注意一个点，其中的一部分链接404找不到了。我突然发现**binwalk**多了一个4小时前的**issue**更新了链接，是真服了。[fix stuffit archive's download url by dev2ero · Pull Request #613 · ReFirmLabs/binwalk (github.com)](https://github.com/ReFirmLabs/binwalk/pull/613)

![image-20220915154244272](img/article/iot-2/image-20220915154244272.png)

需要参照这个issue确定自己的下载地址没有问题后，运行

```bash
sudo ./deps.sh --yes
sudo python3 ./setup.py install
sudo apt-get install libmagic1
```

>  如果在运行./deps.sh中出现问题，可同样根据其脚本手动输入命名安装相关依赖

#### 安装jefferson

>  [sviehb/jefferson: JFFS2 filesystem extraction tool (github.com)](https://github.com/sviehb/jefferson)

```bash
git clone https://github.com/sviehb/jefferson.git
cd jefferson
sudo apt update
sudo apt install python3-pip liblzo2-dev
sudo python3 -m pip install -r requirements.txt
sudo python3 setup.py install
```

回到FAT目录下，运行`setup.sh`

```bash
./setup.sh
```

安装成功后，更改`fat.config`文件内的`root`密码为当前系统的`root`账户密码

![image-20220915193111383](img/article/iot-2/image-20220915193111383.png)

然后正常运行即可，我存放了一些固件在GitHub上可以用于练习：[Loora1N/IOT-Frameware (github.com)](https://github.com/Loora1N/IOT-Frameware)

```bash
./fat.py <firmware file>
```

