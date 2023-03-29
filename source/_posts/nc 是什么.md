---
title: nc/ncat 使用教学
index_img: /img/article/nc/explorer-1024x510-1663035408603-1.png
excerpt: 本文用于介绍信息安全大赛中不少题目需要使用的 nc/ncat 工具。
---

>本文用于介绍信息安全大赛中不少题目需要使用的 nc/ncat 工具。
>如果有什么建议，可以在页面底部登录Github使用评论功能，我看不到账号密码的，不用担心🤩

## nc 是什么

`nc/ncat` 是 netcat 的缩写，它可以读写 TCP 与 UDP 端口——或许你看不懂这句话，这没有关系。简单地说，它可以在字符界面下，让你和服务器沟通交流。

一般来说，有很多题目都会要求你使用 `nc` 连接到他们的服务器，并且进行交互，获得 flag。

## 如何安装 nc

`nc` 命令在 macOS 中是自带的，在 Linux 中一般自带，或是可以使用相应的软件包管理器安装（如在 Debian/Ubuntu 中这个包名叫 netcat）。

当然，如果你在看这篇手册，你的操作系统很有可能是 Windows。它不自带 `nc`，尽管可以用 WSL 或者虚拟机的方式解决，但是如果你觉得这样太麻烦了，也有另外一些方法。

### 使用静态链接的 ncat 程序

前往 https://github.com/andrew-d/static-binaries/blob/master/binaries/windows/x86/ncat.exe 下载此程序。我们也在这里提供了一份。下载下来即可。

[ncat.zip 下载](https://planet.ustclug.org/wp-content/uploads/2019/09/ncat.zip)

*注：`nc`/`ncat` 事实上是两个不同的程序，但在我们接下来的使用上，基本没有区别。`ncat` 是由 Nmap 项目开发的“重置版的 Netcat”。*

#### 我的杀毒软件报毒！

这是 virustotal 的检测结果：https://www.virustotal.com/gui/file/d0baada87420dd7c2e63d0dd3248749c06b53806d3021863c4fa659608053a8a/detection

如果你不相信此来源，也可以下载 `nmap`（一个网络探测、扫描工具），它会附带一个 `ncat`。

## 如何使用 nc?

### Windows

直接双击解压出来的`ncat.exe`，你会看到一个窗口一扫而过——这是正常现象。你需要使用「命令提示符」或者「PowerShell」来访问这个程序。

Windows 10 中，你可以使用资源管理器 -> 文件来在你存放 `nc` 的目录中打开命令提示符或 PowerShell。

![在 Windows 资源管理器中打开命令提示符或 PowerShell](/img/article/nc/explorer-1024x510-1663035408603-1.png)

或者，在Windows 11中，你也可以在开始菜单中搜索PowerShell，然后使用 `cd` 命令转移到`ncat.exe`对应的目录：输入 `cd 文件夹名` 可以让你转移到对应的文件夹，输入 `cd ..` 可以让你转移到上面一层目录。使用 `dir` 命令，可以显示当前目录下所有文件。同时，使用 Tab 键可以帮助你补全名称。操作如下

- 首先在开始菜单中搜索Windows PowerShell并打开，在Windows 11和Windows 10中PowerShell的背景颜色不同，会存在蓝色和黑色两种背景，不用特别在意

![在开始菜单中搜索PowerShell](/img/article/nc/image-20220913095851978.png)

- 复制解压出来的`ncat.exe`文件的地址，我这里是`E:\迅雷下载\ncat`

![复制文件地址](/img/article/nc/image-20220913100512883.png)

- 然后在PowerShell中输入`cd 你的文件地址`，例如`cd E:\迅雷下载\ncat`，便可进入对应文件夹地址，cmd操作相同

![进入文件地址](/img/article/nc/image-20220913100455215.png)

当跳转到你存放 `nc` 的文件夹后：

- 如果你使用的是 PowerShell，输入 `./ncat`（根据 `nc` 对应的名称不同而不同）。
- 如果你使用的是命令提示符cmd，输入 `ncat`（根据 `nc` 对应的名称不同而不同）。

当显示以下内容时，说明你成功运行了它。![成功运行 ncat](/img/article/nc/image-20220913100732337.png)

![成功运行 ncat](/img/article/nc/cmdncat-1024x596-1663035419107-3.png)

成功运行之后我们便可以通过下面这样的指令来运行程序服务并打开题目了

```
#PowerShell中使用
./ncat <ip> <port> 
#例如本题：./ncat 121.4.113.135 20001

#cmd命令提示符中使用
ncat <ip> <port> 
#例如本题：ncat 121.4.113.135 20001
```

![开始做题吧！](/img/article/nc/image-20220913101045325.png)

### Linux & macOS

在Linux和macOS中，由于系统自带`nc`功能，我们的操作就很简便了。首先打开「终端」，输入 `nc`。

```
$ nc
usage: nc [-46AacCDdEFhklMnOortUuvz] [-K tc] [-b boundif] [-i interval] [-p source_port] [--apple-delegate-pid pid] [--apple-delegate-uuid uuid]
	  [-s source_ip_address] [-w timeout] [-X proxy_version]
	  [-x proxy_address[:port]] [hostname] [port[s]]
```

出现了类似上面的输出，说明我们的系统确实存在`nc`指令。那么我们只需要如下输入即可，因为我没有mac，所以就没有mac成功运行的截图了😭😭😭

```bash
nc <ip> <port> 
//例如这次招新赛：nc 121.4.113.135 20001
```

![开始做题吧](/img/article/nc/image-20220913101349105.png)

## 此外

在我们使用浏览器上网的时候，我们和服务器使用了 HTTP 协议进行连接。关于 HTTP 协议的更多细节，你需要自己上网搜索。一般来说，默认是 80 端口。

我们可以使用 `nc`，尝试一次与网页服务器的连接，以百度为例。

输入 `nc www.baidu.com 80`（或者 `ncat www.baidu.com 80`，或者 `./ncat www.baidu.com 80`，请根据以上的介绍自行修改），程序会等待你的输入。

输入 `GET / HTTP/1.0`。这表示，我们使用 `HTTP/1.0` 这个协议版本，用 `GET` 的方式请求根 `/`。输入两下回车，代表我们的 HTTP 请求完成。如果你的网络畅通，百度的网页服务器会立刻返回大量信息，可以自行搜索，了解它们的含义。现在，你成功（在不使用浏览器的情况下）完成了一次与百度网站的连接！

![Success!](/img/article/nc/nc-1024x596.png)

如果你成功了，那么你可以开始愉快地完成我们的题目了！

除了使用 `nc` 直接与服务器交互之外，也可以编程与服务器交互。例如：

- 在信息安全类的竞赛中一个常用的工具是 Python 的 [`pwntools`](https://docs.pwntools.com/en/stable/intro.html)，具体使用方法可以自行搜索了解。



  