---
title: 如何安全快速部署多道静态端口PWN题目 
index_img: img/article/static_port/20220912141715.png
excerpt: 本文部分引用合天网安实验室微信文章，提供静态PWN题部署方式
---

## 前言

> 本文部分引用合天网安实验室微信文章，原文链接：[如何安全快速地部署多道 ctf pwn 比赛题目 (qq.com)](https://mp.weixin.qq.com/s?__biz=MjM5MTYxNjQxOA==&mid=2652848854&idx=1&sn=ff537cc73e76e1ab058bd36cb76749a0&chksm=bd593e1b8a2eb70d41627a1d04c1abec2c071f28c2649ddd9e313c4eda854ca4a26db20a1985&mpshare=1&scene=1&srcid=1011dGXhepYahcla33btEWte#rd)

我们平时见到的多数PWN题，是采用动态端口，动态flag，这种形式在一定程度上减少了比赛中py的数量。即便有，也是思路py，而不是直接传递flag，提升了py的门槛。但有时受限于环境及服务器负担原因，被迫采用静态端口的PWN也不在少数比如[pwnable.tw](https://pwnable.tw/challenge/)以及星盟的[challenge (eonew.cn)](http://pwn.eonew.cn/challenge.php)

2022年的招新便是如此，由于疫情被迫线上上课，线上安排比赛，无法使用校内的服务器资源。受限于自身服务器的性能，只好采取如此下策（但还好只是招新，11月的DUTCTF希望能有好点的资源/(ㄒoㄒ)/~~）

🆘🆘🆘🆘🆘🆘🆘🆘🆘🆘🆘🆘🆘

![可怜的学姐](img/article/static_port/image-20220912135756313.png)

🆘🆘🆘🆘🆘🆘🆘🆘🆘🆘🆘🆘🆘

幸运的是找到了合天在2018年创建的项目`pwn_depoly_chroot`，可以非常方便的搭建多个静态题目，且比较安全，用于这种小型比赛以及普通的训练题库都非常合适。



## pwn_depoly_chroot项目介绍

> 原项目链接：[Github: pwn_depoly_chroot](https://github.com/giantbranch/pwn_deploy_chroot)

- 一次可以部署多个题目到一个 `docker` 容器中

- 自动生成 flag,并备份到当前目录(不过需注意是静态的flag)

- 基于 `xinted + docker + chroot`

- 利用 python 脚本根据 PWN 的文件名自动化地生成 3 个文件：`pwn.xinetd`，`Dockerfile` 和 `docker-compose.yml`

- 在 /bin 目录，利用自己编写的静态编译的 `catflag` 程序作为 `/bin/sh`，这样的话，`system("/bin/sh")` 实际执行的只是读取 flag 文件的内容，完全不给搅屎棍任何操作的余地

- 默认从 10000 端口监听，多一个程序就 +1，起始的监听端口可以在 `config.py` 配置，或者生成 `pwn.xinetd 和 docker-compose.yml` 后自己修改这两个文件

  

## 环境配置

```bash
#安装docker
$ curl -s https://get.docker.com/ | sh

#安装 docker compose 和 git
$ apt install docker-compose git

#下载PWN题搭建项目
$ git clone https://github.com/giantbranch/pwn_deploy_chroot.git
```



## 使用教学

只需要 3 步：

1. 将所有 pwn 题目放入 bin 目录（注意名字不带特殊字符，因为会将文件名作为 linux 用户名）
2. python initialize.py  
3. docker-compose up --build -d

### 详细操作：

- **将你要部署的 pwn 题目放到 bin 目录**

这个项目已经将一个程序 copy 了 3 份作为示例，注意文件名不要含有特殊字符，文件名建议使用字母，下划线，横杆和数字，当然全字母的当然最好了

```bash
root@instance-1:~/pwn_deploy_chroot# ls bin/
pwn1 pwn1_copy1 pwn1_copy2
```

- **运行 initialize.py**

运行脚本后会输出每个 pwn 的监听端口，

```bash
root@instance-1:~/pwn_deploy_chroot# python initialize.py

pwn1's port: 10000
pwn1_copy1's port: 10001
pwn1_copy2's port: 10002
```

文件与端口信息，还有随机生成的 flag 默认备份到 flags.txt 

```bash
root@instance-1:~/pwn_deploy_chroot# cat flags.txt 
pwn1: flag{93aa6da5-db45-46fa-a2e1-af2be6698692}
pwn1_copy1: flag{f9966c51-52e4-4212-ac44-97bf16620b41}
pwn1_copy2: flag{b17949ce-e3fa-4ca7-9fcc-44b8dc997cb3}

pwn1's port: 10000
pwn1_copy1's port: 10001
pwn1_copy2's port: 10002
```

- **启动环境**

请使用 root 用户执行命令

```bash
docker-compose up --build-d
```

不出意外，题目就启动起来了

```bash
root@instance-1:~/pwn_deploy_chroot# netstat -antp | grep docker
tcp6       0     0:::10002               :::*                   LISTEN      19828/docker-proxy
tcp6       0     0:::10000               :::*                   LISTEN      19887/docker-proxy
tcp6       0     0:::10001               :::*                   LISTEN      19873/docker-proxy
```

我们测试一下 pwn1,看看效果

![测试示例](img/article/static_port/20220912141715.png)

可以看到，虽然执行的是 `system("/bin/sh")`，但是实际功能只是输出 flag，这样就非常安全了



### 另外

> 如有二次开发意愿或想了解更多详细内容，可自行查看开头的文章链接或github项目链接，内有部分关键代码讲解

