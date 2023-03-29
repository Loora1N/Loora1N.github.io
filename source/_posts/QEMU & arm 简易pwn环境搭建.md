---
title: QEMU & arm 简易pwn环境搭建
index_img: /img/article/qemu-arm/qemu.png
excerpt: 填坑了，填坑了！学了堆之后很久没有学新东西了，来入门下arm架构下的PWN，为国赛打一点基础。
---

>填坑了，填坑了！学了堆之后很久没有学新东西了，来入门下arm架构下的PWN，为国赛打一点基础。

## 环境搭建(Ubuntu 20.04)

### 安装QEMU

> QEMU是一种通用的开源计算机仿真器和虚拟器。QEMU共有两种操作模式
> **全系统仿真**：能够在任意支持的架构上为任何机器运行一个完整的操作系统
>**用户模式仿真**：能够在任意支持的架构上为另一个Linux/BSD运行程序 因此我们可以在linux操作系统中安装它，然后用它来调试其它架构平台的程序。

```bash
##首先安装qemu
sudo apt update
sudo apt install qemu
```
安装好后可以用tab补全看看是否正确安装。在安装qemu后，对于静态链接的arm程序就已经可以直接运行了，使用命令`qemu-arm prog`运行32位的arm程序，其中prog指代程序名,对于64位程序使用`qemu-aarch64`但对于动态链接的程序还是无法正常运行，此时需要安装对应架构的动态链接库才可以正常运行。

![Untitled](/img/article/qemu-arm/qemu.png)

### 安装动态链接库


我们可以使用如下命令来搜索可安装的动态链接库。这里我们以64位为例，安装debug版本的libc
```bash
##搜索libc
sudo apt search "libc6" | grep arm
##安装libc
sudo apt install libc6-dbg-arm64-cross
```

![Untitled](/img/article/qemu-arm/qemu2.png)


安装完后，在`/usr`目录下会出现`aarch64-linux-gnu`这个文件夹，该文件夹即对应刚安装好的arm64位libc库，之后我们使用下面的命令指定arm程序的动态链接器，即可运行程序。
```bash
qemu-aarch64 -g 12345 -L /usr/aarch64-linux-gnu/ ./prog
## -g 代表为程序运行分配的进程端口号，一般用于gdb调试，不填时系统自动分配
## -L 代表动态链接库的文件夹地址，静态链接程序可以不需要
```
`qemu-arm`指的是**arm32架构**，用这个命令来运行32位的arm程序，而`qemu-aarch64`对应的才是**arm64位架构的程序**，上面两者默认都是小端程序；`qemu-armeb`用来运行**大端的arm程序**。当然在**上面安装包截图中显示的是arm64，其实和aarch64指的是同一种体系结构，只不过命名略有不同**。

## 使用gdb-multiarch进行调试

首先使用如下命令安装gdb-multiarch
```bash
sudo apt update
sudo apt install gdb-multiarch
```

以平台上的ez-arch为例，我们先使用qemu启动程序，并为其分配一个端口

```bash
qemu-aarch64 -g 12345 -L /usr/aarch64-linux-gnu/ ./stack
```

使用另外一个终端，打开qemu-multiarch

```bash
┌─[loorain@ubuntu] - [~/dest0g3-520/ez-aarch] - [1568]
└─[$] sudo gdb-multiarch                                                                             [22:22:12]
......
......
pwndbg> set architecture aarch64
The target architecture is assumed to be aarch64
pwndbg> target remote localhost:12345
Remote debugging using localhost:12345
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
```

![Untitled](/img/article/qemu-arm/qemu3.png)

另外在pwntools中，我们可以使用如下代码来动态调试程序。

```python
p = process(["qemu-aarch64", "-g", "12345","-L","/usr/aarch64-linux-gnu/", "./stack"])
##32arm只需要替换指令为qemu-arm，并更换对应的动态链接库
```

**注意：不进行调试时不建议使用**`-p`**不然可能会导致程序停滞，只能等待gdb链接**