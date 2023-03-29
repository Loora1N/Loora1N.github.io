---
title: Kernel-pwnbase详解
index_img: /img/article/kernelbase/kernel.png
excerpt: 参加starnewctf week4后，想要复现一道kernel的题。网上基本找不到Kernel-pwn的base，因此做了这个pwnbase用于在ctfd上提供动态kernel题目。
---

> 参加starnewctf week4后，想要复现一道kernel的题。网上基本找不到Kernel-pwn的base，因此做了这个pwnbase用于在ctfd上提供动态kernel题目。
> base下载地址[kernelbase](https://github.com/Loora1N/pwnbase/tree/main/kernelbase)



## Dockfile

```dockerfile
FROM ubuntu:20.04

RUN sed -i "s/http:\/\/archive.ubuntu.com/http:\/\/mirrors.tuna.tsinghua.edu.cn/g" /etc/apt/sources.list && \
    apt-get update && apt-get -y dist-upgrade && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y lib32z1 xinetd git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev qemu qemu-system-x86

RUN useradd -m ctf

WORKDIR /home/ctf

COPY ./ctf.xinetd /etc/xinetd.d/ctf
COPY ./start.sh /start.sh
RUN echo "Blocked by ctf_xinetd" > /etc/banner_fail

RUN chmod +x /start.sh

COPY ./files/ /home/ctf/
RUN chown -R root:ctf /home/ctf && \
    chmod -R 750 /home/ctf

CMD ["/start.sh"]

EXPOSE 9999
```

docker基础是ubuntu20.04，整个dockerfile主要是下载一些关键库，以及对目录权限的管理，最后会调用`start.sh`, 并且开放9999端口



## start.sh

```sh
#!/bin/sh
# Add your startup script

# DO NOT DELETE
/etc/init.d/xinetd start;
sleep infinity;
```

东西很短，其实就是调用`ctf.xinetd`



## ctf.xinetd

```
service ctf
{
    disable = no
    socket_type = stream
    protocol    = tcp
    wait        = no
    user        = root
    type        = UNLISTED
    port        = 9999
    bind        = 0.0.0.0
    server      = /usr/sbin/chroot
    # replace helloworld to your program
    server_args = --userspec=1000:1000 / ./home/ctf/helloworld
    banner_fail = /etc/banner_fail
    # safety options
    per_source	= 10 # the maximum instances of this service per source IP address
    rlimit_cpu	= 20 # the maximum number of CPU seconds that the service may use
    #rlimit_as  = 1024M # the Address Space resource limit for the service
    #access_times = 2:00-9:00 12:00-24:00
}
```

核心在13行的后半，nc连接后会调用helloworld程序



## helloworld

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>

void init()
{
    setvbuf(stdin, 0LL, 2, 0LL);
    setvbuf(stdout, 0LL, 2, 0LL);
    setvbuf(stderr, 0LL, 2, 0LL);
}

int main()
{
    init();   
    signal(SIGCHLD, SIG_DFL);
    system("/home/ctf/boot.sh");
}
```

这个程序与普通的pwn题目类似，先初始化输入输出，随后调用boot.sh启动qemu即可。signal函数是为了防止system函数可能出现的错误。



## boot.sh

```sh
#!/bin/sh

qemu-system-x86_64  \
-m 128M \
-kernel /home/ctf/bzImage \
-initrd /home/ctf/rootfs.cpio \
-monitor /dev/null \
-cpu kvm64,+smep,+smap \
-append "root=/dev/ram console=ttyS0 oops=panic quiet panic=1 kaslr noapic" \
-nographic \
-no-reboot 

```

关键在于 kernel和initrd一定要使用绝对路径，即便它们与`helloworld`在一个目录下，system调用时依然可能会出现找不到文件的情况出现，建议在 `-append`里加入 `noapic` 可以防止出现AIPC错误。