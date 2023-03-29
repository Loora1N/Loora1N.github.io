---
title: vscode使用pwntools代码高亮出错
index_img: img/article/vscode-pwntools/image-20221025192616883.png
excerpt: vscode在使用pwntools的send系列函数时，会出现高亮错误的情况，目前最好的解决办法时对pwntools源码进行更改
---

> vscode在使用pwntools的send系列函数时，会出现高亮错误的情况，目前最好的解决办法时对pwntools源码进行更改

右键send函数，选择`Go to Definition`，打开pwntools源码

![image-20221025192410368](img/article/vscode-pwntools/image-20221025192410368.png)

`ctrl+F`搜索send_raw函数

![image-20221025192616883](img/article/vscode-pwntools/image-20221025192616883.png)

把函数末尾的`raise EOFError('Not implemented')`更换成`raise NotImplementedError()`既可

