---
title: 适用多版本libc的pwn-base
index_img: img/article/pwnbase/image-20221028202807333.png
excerpt: 由于经常要在校内的CTFd平台上题，不使用pwn-base的构建方式会使得镜像体积过大，浪费许多服务器的存储空间。但使用pwn-base的情况下，不同题目libc要求不同，又需要构建多个pwn-base环境，过于麻烦。因而创建了这个可根据需求自行更换libc版本的pwn-base
---


## 前言

由于经常要在校内的CTFd平台上题，不使用pwn-base的构建方式会使得镜像体积过大，浪费许多服务器的存储空间。但使用pwn-base的情况下，不同题目libc要求不同，又需要构建多个pwn-base环境，过于麻烦。因而创建了这个可根据需求自行更换libc版本的pwn-base，有需要可以自己下载源码更改，链接在下方：

> docker：loora1n/Ubuntu2.35
>
> 链接：[pwnbase/Ubuntu2.35 at main · Loora1N/pwnbase (github.com)](https://github.com/Loora1N/pwnbase/tree/main/Ubuntu2.35)

这个库里同时还包含之前制作的Kernel pwn base

![pwnbase](img/article/pwnbase/image-20221028202807333.png)

## 默认libc版本

> GNU C Library (Ubuntu GLIBC 2.35-0ubuntu3.1) stable release version 2.35. Copyright (C) 2022 Free Software Foundation, Inc. This is free software; see the source for copying conditions. There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Compiled by GNU CC version 11.2.0. libc ABIs: UNIQUE IFUNC ABSOLUTE For bug reporting instructions, please see: https://bugs.launchpad.net/ubuntu/+source/glibc/+bugs.

## 使用方式

首先在创建一个文件夹用于构建你的题目镜像，如我这里的pwn文件夹

![题目文件夹](img/article/pwnbase/image-20221028203034274.png)

将你的题目和所需的libc, ld文件也放入这里，新建一个Dockerfile，粘贴如下内容即可

```dockerfile
FROM loora1n/ubuntu2.35
COPY helloworld /pwn/pwn					
COPY ld-2.32.so /pwn/ld-2.32.so		
COPY libc-2.32.so /pwn/libc-2.32.so
RUN chmod +x /pwn/pwn && \
    patchelf --set-interpreter /pwn/ld-2.32.so /pwn/pwn && \
    patchelf --replace-needed libc.so.6 /pwn/libc-2.32.so /pwn/pwn   
```

只需替换`COPY`后紧挨着的文件名即可，我这里以libc版本需求为2.32的helloworld为例。

保存后，开始构建题目镜像即可

```shell
docker build -t usrname/image .
docker push usrname/image
```

