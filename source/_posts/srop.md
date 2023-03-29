---
title: setcontext & SROP
index_img: /img/article/srop/srop1.png
excerpt: SROP(Sigreturn Oriented Programming) 于 2014 年被 Vrije Universiteit Amsterdam 的 Erik Bosman 提出，其相关研究为Framing Signals — A Return to Portable Shellcode
---
## 前言

>SROP(Sigreturn Oriented Programming) 于 2014 年被 Vrije Universiteit Amsterdam 的 Erik Bosman 提出，其相关研究**Framing Signals — A Return to Portable Shellcode** 其中相关的 paper 以及如下：
>[paper](https://www.ieee-security.org/TC/SP2014/papers/FramingSignals-AReturntoPortableShellcode.pdf)

## 机制详解

signal 机制是类 unix 系统中进程之间相互传递信息的一种方法。一般，我们也称其为软中断信号，或者软中断。比如说，进程之间可以通过系统调用 kill 来发送软中断信号。一般来说，信号机制常见的步骤如下图所示：

![Untitled](/img/article/srop/srop1.png)

1. 内核向某个进程发送 signal 机制，该进程会被暂时挂起，进入内核态。
2. 内核会为该进程保存相应的上下文，**主要是将所有寄存器压入栈中，以及压入 signal 信息，以及指向 `sigreturn` 的系统调用地址**。此时栈的结构如下图所示，我们称 `ucontext` 以及 `siginfo` 这一段为 **Signal Frame**。**需要注意的是，这一部分是在用户进程的地址空间的。**之后会跳转到注册过的 signal handler 中处理相应的 signal。因此，当 signal handler 执行完之后，就会执行 `sigreturn` 代码。
3. `signal handler` 返回后，内核为执行 `sigreturn` 系统调用，为该进程恢复之前保存的上下文，其中包括将所有压入的寄存器，重新 pop 回对应的寄存器，最后恢复进程的执行。其中，32 位的 `sigreturn` 的调用号为 77，64 位的系统调用号为 15。

![Untitled](/img/article/srop/srop2.png)

上述谈到的从栈中POP出寄存器内容的过程由 `setcontext` 函数实现，如下图所示：

![Untitled](/img/article/srop/srop3.png)

我们可以看到，从指令`mov rsp, [rdi+0A0h]`开始，便是一连串对于寄存器的恢复操作。而且结构由Signal Frame结构体实现，以下是结构源码

**X86**

```c
struct sigcontext
{
  unsigned short gs, __gsh;
  unsigned short fs, __fsh;
  unsigned short es, __esh;
  unsigned short ds, __dsh;
  unsigned long edi;
  unsigned long esi;
  unsigned long ebp;
  unsigned long esp;
  unsigned long ebx;
  unsigned long edx;
  unsigned long ecx;
  unsigned long eax;
  unsigned long trapno;
  unsigned long err;
  unsigned long eip;
  unsigned short cs, __csh;
  unsigned long eflags;
  unsigned long esp_at_signal;
  unsigned short ss, __ssh;
  struct _fpstate * fpstate;
  unsigned long oldmask;
  unsigned long cr2;
};
```

**X64**

```c
struct _fpstate
{
  /* FPU environment matching the 64-bit FXSAVE layout.  */
  __uint16_t        cwd;
  __uint16_t        swd;
  __uint16_t        ftw;
  __uint16_t        fop;
  __uint64_t        rip;
  __uint64_t        rdp;
  __uint32_t        mxcsr;
  __uint32_t        mxcr_mask;
  struct _fpxreg    _st[8];
  struct _xmmreg    _xmm[16];
  __uint32_t        padding[24];
};

struct sigcontext
{
  __uint64_t r8;
  __uint64_t r9;
  __uint64_t r10;
  __uint64_t r11;
  __uint64_t r12;
  __uint64_t r13;
  __uint64_t r14;
  __uint64_t r15;
  __uint64_t rdi;
  __uint64_t rsi;
  __uint64_t rbp;
  __uint64_t rbx;
  __uint64_t rdx;
  __uint64_t rax;
  __uint64_t rcx;
  __uint64_t rsp;
  __uint64_t rip;
  __uint64_t eflags;
  unsigned short cs;
  unsigned short gs;
  unsigned short fs;
  unsigned short __pad0;
  __uint64_t err;
  __uint64_t trapno;
  __uint64_t oldmask;
  __uint64_t cr2;
  __extension__ union
    {
      struct _fpstate * fpstate;
      __uint64_t __fpstate_word;
    };
  __uint64_t __reserved1 [8];
};
```

## 攻击方式

### 直接getshell

**示意图**

![Untitled](/img/article/srop/srop4.png)

我们可以直接在栈上伪造Signal Frame，利用syscall(0xF)来触发对于伪造寄存器的恢复，在`setcontext` 执行完毕后，便会getshell（或利用栈转移到其他地址也可以，但无论怎样都要保证在syscall时，rsp需指向Fake Frame 的开头)

### SROP

在上面直接getshell的基础上，我们可以自然的想到，通过不断伪造sigframe可以实现类似ROP的效果，只需要将其中rsp控制到下一个fakeframe的开头,rip加上ret即可。

**示意图**

![Untitled](/img/article/srop/srop5.png)

### 栈转移

由于其可以轻易的控制几乎所有寄存器的值，栈转移这样的操作也可以轻松实现

### 堆中直接利用setcontext

在面对堆题目时，我们很难直接在栈上构造内容并且控制rsp指向frame开头。此时可以直接通过劫持hook为`setcontext+53` 处(**mov rsp, [rdi+0A0h]**)，直接触发对于寄存器的一系列pop操作。且由于free时，rip指向chunk的mem域，那么我们只需要在将要free的chunk里写上Fake Frame，并合理控制与rip的距离即可。

## 例题

平台题目SROP、 ciscn 2021 silverwolf