---
title: HEAP 攻击方式记录以及总结
index_img: /img/article/index.png
excerpt: 文章为台湾国立大学的PWN例会heap方面学习记录
---

> 文章很多图源来自NTUSTISC例会视频，油管链接：[NTUSTISC - Pwn 3 - YouTube](https://www.youtube.com/watch?v=ItpJY9Lpw-o&list=PLHP-B3mcSY_5r5AJsZyTZLwjX9z8v_b5x)
> 目前只支持到libc2.34版本

## Heap-Based Buffer Overflow

最最基本的攻击方式，由于未对输入长度限制，且堆块是一片连续存储的内存空间特性。导致可以对其后高地址的chunk的各种信息进行覆写。

![](/img/article/1.png)

![](/img/article/2.png)

## UAF(Use-after-free)

在free()掉chunk之后，原本指向chunk的指针由于程序猿自身的懒惰，没有将pointer = NULL; 导致依然可以使用该指针进行额外的操作

UAF漏洞一般常用于泄露，或者更改已经free掉的chunk中的信息，最常见便是libc的泄露,以及对Tcache/Fastbin链表的控制

> 当第一块chunk进入unsorted bin后，他的fd和bk指针会共同指向main_arena+88(具体偏移似乎会随着版本变化，做题时手动调试即可)，意味着当存在UAF漏洞时，我们可以输出这个chunk的内容即能得到libc

## Fastbin Dup

fastbin dup 实际是在fastbin链上实现的狭义的double free。由于fastbin 自身源码的缺陷，在对double free 的检测机制仅仅验证了当前free掉的chunk与fast bin 第一个chunk是否相同，而对于其他chunk则没有做检测。另外fastbin在free时会检测当前当前的chunk于头部chunk的size大小是否相同。

那么构造double free 时只需要穿插一块其他的chunk即可实现

```c
free(1);
free(2);
free(1);
```



![](/img/article/3.png)

在完成double free 之后我们只需要malloc 这里的chunk1 就能得到一块 既是allocated 又是 free 状态的chunk。接下来我们就能对其fd指针进行修改，从而控制fastbin链表的方向是我们想要的地方，便能实现对任意地址的控制(读写)

![](/img/article/4.png)

> 额外需要注意的点：
>
> - fastbin会对chunk头的大小进行检测，那么图中malice处chunk的size需要控制好
> - libc2.26之后，free的小于0x410的chunk会先进入Tcache中，最多承纳7个chunk
> - 且在tcache填满之后，malloc优先会取出tcache中的chunk，可以利用calloc不会申请tcache的特点

## Tcache Dup

> Tcache 在最近的几个libc中加入了多个机制，用于保证其安全性，这里我将分不同版本讨论

### libc2.26-libc2.28

在这几个版本中，几乎没有检测机制，对同一块chunk直接进行double free即可，甚至比fastbin dup还要简易

### libc2.29-libc2.31

在这几个版本中，tcache 的 bk指针处存放了一个8byte的随机数作为key，即代表这块chunk在tcache链中。每次free时，检测bk的位置的值是否为key，若是，则为doublefree。

> 绕过方式：存在UAF漏洞的情况下，我们可以直接更改，tcache中chunk的key值即可

### libc2.32

#### Safe-linking(ptr xor)

libc2.32中引入了Safe-linking机制，应用于fastbin和tcache中

```c
static __always_inline void
tcache_put (mchunkptr chunk, size_t tc_idx)
{
tcache_entry *e = (tcache_entry *) chunk2mem (chunk);

/* Mark this chunk as “in the tcache” so the test in _int_free will
detect a double free.  */
e->key = tcache;

e->next = PROTECT_PTR (&e->next, tcache->entries[tc_idx]);
tcache->entries[tc_idx] = e;
++(tcache->counts[tc_idx]);
}
```

可以看到经glibc-2.32更新后，e->next不再是直接指向原来的tcache头指针，而是指向了经PROTECT_PTR处理过的指针，查看PROTECT_PTR定义：

```c
#define PROTECT_PTR(pos, ptr) \
((__typeof (ptr)) ((((size_t) pos) >> 12) ^ ((size_t) ptr)));
```

结合tcache_put()函数可以得出e->next最终指向了e->next地址右移12位后的值与当前tcache头指针值异或的结果，这里引用Safe-Linking设计师文章[1]中的一张图描述此过程：

![img](/img/article/5.png)

其实就是一个简单的异或操作，这里所有的指针都进行了处理，具体细节可以自行查询，不过多赘述

#### 内存对齐检测

此外，在glibc-2.32版本中还引入了对tcache和fastbins中申请及释放内存地址的对齐检测，以tcache_get()为例：

```c
static __always_inline void *

tcache_get (size_t tc_idx)

{

tcache_entry *e = tcache->entries[tc_idx];

if (__glibc_unlikely (!aligned_OK (e)))

malloc_printerr (“malloc(): unaligned tcache chunk detected”);

tcache->entries[tc_idx] = REVEAL_PTR (e->next);

–(tcache->counts[tc_idx]);

e->key = NULL;

return (void *) e;

}
```

aligned_OK()定义为：

```c
#define aligned_OK(m)  (((unsigned long)(m) & MALLOC_ALIGN_MASK) == 0)
#define MALLOC_ALIGN_MASK (MALLOC_ALIGNMENT – 1)
#define MALLOC_ALIGNMENT (2 * SIZE_SZ < __alignof__ (long double) \
? __alignof__ (long double) : 2 * SIZE_SZ)
```

可以看到内存地址需要以0x10字节对齐。

### libc-2.34

glibc-2.34 的到来给予了hook劫持沉重的一击，移除了以下几个hook符号：

> __free_hook
>
> __malloc_hook
>
> __realloc_hook
>
> __memalign_hook
>
> __after_morecore_hook

#### 绕过思路

1. 利用IO_vtable
2. 利用exit_hook
3. 劫持返回地址进行ROP

## unsafe unlink

unsafe-unlink利用chunkfree时的合并机制，实现了地址的任意读写

> 使用条件：
>
> - 能够利用溢出或其他漏洞，将某chunk的size域的最低16进制位修改为0，例: 0x91 --> 0x90

```c
/*这是一段unsafe-unlink的示例代码，我们在最后成功利用这个漏洞，
实现了对非堆上地址victimstring的内容更改*/
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

uint64_t * chunk0_ptr;
int main()
{
    chunk0_ptr = (uint64_t*)malloc(0x80);       //chunk0
    uint64_t *chunk1_ptr = (uint64_t*) malloc(0x80);        //chunk1
    fprintf(stderr, "chunk0_ptr: %p -> %p\n",&chunk0_ptr, chunk0_ptr);
    fprintf(stderr, "victim chunk: %p\n\n",chunk1_ptr);

    /*pass the check:   (chunksize(p) != prev_size(next_chunk(P)) == False)
                        chunk0_ptr[1] = 0x0;// or 0x8, 0x80*/

    //pass the check:   (p->fd->bk != p || p->bk->fd != p) == False

    chunk0_ptr [2] = (uint64_t) &chunk0_ptr - 0x18;
    chunk0_ptr [3] = (uint64_t) &chunk0_ptr - 0x10;
    fprintf(stderr, "fake fd: %p = &chunk0_ptr - 0x18", (void*)chunk0_ptr[2]);
    fprintf(stderr, "fake bk: %p = &chunk0_ptr - 0x10", (void*)chunk0_ptr[3]);
    
    uint64_t *chunk1_hdr = (void*) chunk1_ptr - 0x10;
    chunk1_hdr[0] = 0x80;
    chunk1_hdr[1] &= ~1;

    /*  int *t[10], i;                  //tcache
        for(i=0; i<7; i++){
            t[i] = malloc(0x80);
        }
        for(i=0;i<7;i++){
            free(t[i]);
        };  */

    free(chunk1_ptr);               //unlink

    char victim_string[8] = "AAAAAAA";
    chunk0_ptr[3] = (uint64_t) victim_string;   //overwrite itself
    fprintf(stderr, "old value: %s\n", victim_string);

    chunk0_ptr[0] = 0x42424242424242LL;     //over wirting victim_string
    fprintf(stderr, "new Value: %s\n", victim_string);

}
```

#### 示例图

![](/img/article/6.png)
