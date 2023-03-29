---
title:  PWN环境搭建（萌新入门）
index_img: img/article/pwn-envir/image-20220905153059189-1662367638683-2.png
excerpt: 为大一新生提供的pwn入门环境搭建指南
---


## IDA PRO-反汇编利器

> 推荐版本7.6及以上，高版本增添了对于go语言程序的反汇编，寻找资源需要谨慎哦
>
> 写文章的时候顺便看了眼价格，，，，，算了还是自己找资源吧

![image-20220905153059189](img/article/pwn-envir/image-20220905153059189-1662367638683-2.png)

IDA可以对可执行程序反汇编操作，能够看到整个程序的框架、汇编代码、伪代码、函数调用、以及一些字符串的全局遍变量。无论对于Reverse还是PWN都是最核心的工具之一，这也是我放在最前面提及的原因，大概界面如下图。

![image-20220905153323145](img/article/pwn-envir/image-20220905153323145.png)



## Linux发行版选择

这里我主要推荐Ubuntu，相比于其他发行版，Ubuntu的图形化界面比较方便新人入手学习Linux. 另外一方面，CTF中大多数PWN题环境都是在ubuntu下的，一般为近几年的版本。

> 以2022年PWN题目环境为例，Ubuntu版本基本集中在Ubuntu18，Ubuntu20，Ubuntu22. 同理在2024年，不出意外的话题目环境将会变化为Ubuntu20，Ubuntu22，Ubuntu24. 主要是因为Ubuntu发行为LTS制(Long Term Support)即版本支持年份为6年，且每两年发布一次新版本

那么在选择时就推荐选择距离时间最近的偶数年份版本，比如Ubuntu20或22. Ubuntu系统的iso镜像文件可以自行在网上寻找，也可以在大工镜像网站下载：[大连理工大学开源软件镜像站 (dlut.edu.cn)](http://mirror.dlut.edu.cn/ubuntu-releases/22.04/)

![image-20220905164622966](img/article/pwn-envir/image-20220905164622966.png)

虚拟机安装Windows环境下建议使用VMware。同样建议版本尽量新一些，如VM16以上，这样可以避免一部分的蓝屏BUG发生。

> Linux安装教学不在这里赘述了，毕竟本文的核心是PWN环境的搭建，如需安装的相关资源，可自行网络搜索



## Pwntools安装

Ubuntu安装完成后默认应该是有python3的，可以用python -V或python3 -V来查看版本

![image-20220905161539363](img/article/pwn-envir/image-20220905161539363.png)

> 这里推荐使用python3, 虽然有人说Pwntools对于python2的适配性更好，但由于python2过于古老，新版本反倒在其他方面更加稳定一些。在拥有其他更多插件和包的情况下，个人感觉还是python3更优一些。

```bash	
$ sudo apt-get update
$ sudo apt-get install python3 python3-pip python3-dev git libssl-dev libffi-dev build-essential
$ sudo python3 -m pip install --upgrade pip
$ sudo python3 -m pip install --upgrade pwntools
```

之后我们可以使用`import pwn`来测试一些是否成功安装,出现下方情况基本就可以了

![image-20220905162034730](img/article/pwn-envir/image-20220905162034730.png)



## pwndbg & pwngdb安装

这两个均是GDB的插件之一，用于在Linux对程序进行动态调试，而动态调试也是PWN最核心的部分之一。拥有这些工具，我们能更加方便的查看程序运行过程中，内存变化。如栈、堆、寄存器等等地方的内容。

> pwndbg安装

```bash
git clone https://github.com/pwndbg/pwndbg
cd pwndbg
./setup.sh
```

> pwngdb安装

```bash
cd ~/
git clone https://github.com/scwuaptx/Pwngdb.git 
cp ~/Pwngdb/.gdbinit ~/
```

> 注意：上述两个插件并不兼容，但可以通过更改.gdbinit文件切换使用
>
> .gdbinit文件配置如下

```js
source /home/pwnki/pwndbg/gdbinit.py 
#source ~/peda/peda.py //使用 pwndbg 就要把 peda 注释掉，反过来也一样
source ~/Pwngdb/pwngdb.py
source ~/Pwngdb/angelheap/gdbinit.py

define hook-run
python
import angelheap
angelheap.init_angelheap()
end
end
```

![image-20220905165859512](img/article/pwn-envir/image-20220905165859512.png)

## checksec安装

用于检测程序开启了哪些保护机制，往往是PWN解题的第一步，安装方式：

```bash
$ sudo apt install checksec
```

使用方式:

```bash
$ checksec 程序文件
```

![image-20220905164504590](img/article/pwn-envir/image-20220905164504590.png)





## 更多插件

> 虽然还有很多插件并没有提及，但对于刚刚接触pwn的同学来说，以上插件已经够完成一部分简单的pwn题目，更多的插件需要在将来自己学习的过程中慢慢补充。下面只做一列举，有兴趣的同学可以自行搜索查看

### glibc-all-in-one

> 可以用于下载各种版本的glibc文件，在本地模拟其他libc环境时非常实用

### seccomp-tools

> 用于检测程序沙盒防护开启情况，往往在一些会用到ORW等方法解题时会用来检测

### one_gadget

> 这个工具可以用查找libc中可以用来getshell的片段地址，在各类题目中都应用广泛

### LibcSeacher

> 该工具往往用于确定libc版本并确定其libc版本下的函数偏移，可以被网页端的libc库替代