---
title: 【IOT】固件解包及打包
index_img: img/article/iot-bin/image-20220916182204458-1663326116419-2.png
excerpt: 固件解包魔改及重新打包教学
---
## 解包

```bash
binwalk -eM file.bin
```



## 打包

首先查看原.bin文件文件结构

![image-20220916182204458](img/article/iot-bin/image-20220916182204458-1663326116419-2.png)

然后切割出文件目录之前的头部

```bash
sudo dd if=file.bin of=head.bin bs=1 skip=0 count=1376400
#if是原固件，of是输出文件，bs是单位长度，skip是偏移量，count是分区大小取到文件目录之前
```

正常解包，然后更改文件内容，随后制作文件目录的bin文件

```bash
sudo mksquashfs squashfs-root rootfs.bin -comp xz
```

头部和文件部分联合

```bash
sudo cat header.bin rootfs.bin > pwn.bin
```

