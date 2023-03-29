---
title: Hexo部署出现错误 Error Spawn failed 解决方式(code 128)
index_img: /img/article/error128/vscode1.png
excerpt: 记录一次报错处理方式，在更新了自己的必读书目之后，执行 hexo clean && hexo g -d 时出现如下错误
---

> 记录一次报错处理方式，在更新了自己的必读书目之后，执行 hexo clean && hexo g -d 时出现如下错误

## 错误信息一

```bash
Missing or invalid credentials.
Error: connect ECONNREFUSED /run/user/1000/vscode-git-2098f677fa.sock
    at PipeConnectWrap.afterConnect [as oncomplete] (node:net:1161:16) {
  errno: -111,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '/run/user/1000/vscode-git-2098f677fa.sock'
}
Missing or invalid credentials.
Error: connect ECONNREFUSED /run/user/1000/vscode-git-2098f677fa.sock
    at PipeConnectWrap.afterConnect [as oncomplete] (node:net:1161:16) {
  errno: -111,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '/run/user/1000/vscode-git-2098f677fa.sock'
}
remote: No anonymous write access.
fatal: Authentication failed for 'https://github.com/Loora1N/Loora1N.github.io/'
FATAL {
  err: Error: Spawn failed
      at ChildProcess.<anonymous> (/home/loorain/myBlog/node_modules/hexo-util/lib/spawn.js:51:21)
      at ChildProcess.emit (node:events:527:28)
      at ChildProcess._handle.onexit (node:internal/child_process:291:12) {
    code: 128
  }
} Something's wrong. Maybe you can find the solution here: %s https://hexo.io/docs/troubleshooting.html
```

### 解决方法

一、打开vscode设置界面，搜索git.terminal Authentication，将他前面的 √ 去掉

![image-20220607164521802](/img/article/error128/vscode1.png)

二、crtl + shift + p 打开搜索框，搜索reload window并运行

![image-20220607164739265](/img/article/error128/vscode2.png)

三、再次尝试hexo g，解决成功

![image-20220607164850847](/img/article/error128/vscode3.png)

## 错误信息二

```bash
fatal: unable to access 'https://github.com/Loora1N/Loora1N.github.io/': gnutls_handshake() failed: Error in the pull function.
FATAL {
  err: Error: Spawn failed
      at ChildProcess.<anonymous> (/home/loorain/myBlog/node_modules/hexo-util/lib/spawn.js:51:21)
      at ChildProcess.emit (node:events:527:28)
      at ChildProcess._handle.onexit (node:internal/child_process:291:12) {
    code: 128
  }
} Something's wrong. Maybe you can find the solution here: %s https://hexo.io/docs/troubleshooting.html
```
### 解决方法

打开_config.yml文件，更改https仓库地址为ssh地址,配置ssh密钥即可
```yaml
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: git
  repo:	https://github.com/YourName/YourName.github.io.git(不要使用这个)
  		git@github.com:YourName/YourName.github.io.git(用这个)
  branch: master
```

