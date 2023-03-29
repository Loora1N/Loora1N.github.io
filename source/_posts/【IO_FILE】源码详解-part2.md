---
title: 【IO_FILE】源码详解-part2
index_img: img/article/fopen/t017448d9547dc07159-1669695992482-3.png
excerpt: 在上一篇文章中，主要对IO_FILE结构体相关以及fopen函数进行了深入的介绍。本文将继续从glibc2.36源码出发，介绍fread函数相关内容
---

> 在上一篇文章中，主要对IO_FILE结构体相关以及fopen函数进行了深入的介绍。本文将继续从glibc2.36源码出发，介绍fread函数相关内容

## fread

首先是一个宏定义，替换为`_IO_fread`函数

```c
//	htps://elixir.bootlin.com/glibc/latest/source/stdio-common/getw.c#L21
#define fread(p, m, n, s) _IO_fread (p, m, n, s)

// https://elixir.bootlin.com/glibc/latest/source/libio/iofread.c#L30
size_t
_IO_fread (void *buf, size_t size, size_t count, FILE *fp)
{
  size_t bytes_requested = size * count;
  size_t bytes_read;
  CHECK_FILE (fp, 0);
  if (bytes_requested == 0)
    return 0;
  _IO_acquire_lock (fp);
  bytes_read = _IO_sgetn (fp, (char *) buf, bytes_requested);
  _IO_release_lock (fp);
  return bytes_requested == bytes_read ? count : bytes_read / size;
}
libc_hidden_def (_IO_fread)
```

## CHECK_FILE

首先进入`CHECK_FILE`对**fp**进行检查

```c
// https://elixir.bootlin.com/glibc/latest/source/libio/libioP.h#L866
#ifdef IO_DEBUG
# define CHECK_FILE(FILE, RET) do {				\
    if ((FILE) == NULL						\
	|| ((FILE)->_flags & _IO_MAGIC_MASK) != _IO_MAGIC)	\
      {								\
	__set_errno (EINVAL);					\
	return RET;						\
      }								\
  } while (0)
#else
# define CHECK_FILE(FILE, RET) do { } while (0)
#endif
```

这里放置一下所有用于检测flag的宏定义的值,目前后16位只使用了15个，0x4000还未设定

```c
// https://elixir.bootlin.com/glibc/latest/source/libio/libio.h#L67
#define _IO_MAGIC         0xFBAD0000 /* Magic number */
#define _IO_MAGIC_MASK    0xFFFF0000
#define _IO_USER_BUF          0x0001 /* Don't deallocate buffer on close. */
#define _IO_UNBUFFERED        0x0002
#define _IO_NO_READS          0x0004 /* Reading not allowed.  */
#define _IO_NO_WRITES         0x0008 /* Writing not allowed.  */
#define _IO_EOF_SEEN          0x0010
#define _IO_ERR_SEEN          0x0020
#define _IO_DELETE_DONT_CLOSE 0x0040 /* Don't call close(_fileno) on close.  */
#define _IO_LINKED            0x0080 /* In the list of all open files.  */
#define _IO_IN_BACKUP         0x0100
#define _IO_LINE_BUF          0x0200
#define _IO_TIED_PUT_GET      0x0400 /* Put and get pointer move in unison.  */
#define _IO_CURRENTLY_PUTTING 0x0800
#define _IO_IS_APPENDING      0x1000
#define _IO_IS_FILEBUF        0x2000
                           /* 0x4000  No longer used, reserved for compat.  */
#define _IO_USER_LOCK         0x8000
```

所以这里的函数就是检测`(FILE)->_flags & _IO_MAGIC_MASK) != _IO_MAGIC`即，高位是否为**0xFBAD0000**。然后会调用` _IO_acquire_lock`，这个宏同样是对结构体进行一些处理，基本可以无视

```c
// https://elixir.bootlin.com/glibc/latest/source/sysdeps/generic/stdio-lock.h#L51
# ifdef __EXCEPTIONS
# define _IO_acquire_lock(_fp) \
  do {									      \
    FILE *_IO_acquire_lock_file						      \
	__attribute__((cleanup (_IO_acquire_lock_fct)))			      \
	= (_fp);							      \
    _IO_flockfile (_IO_acquire_lock_file);
# else
#  define _IO_acquire_lock(_fp) _IO_acquire_lock_needs_exceptions_enabled
# endif
# define _IO_release_lock(_fp) ; } while (0)

#endif
```

## _IO_sgetn

核心部分是这里的`_IO_sgetn`函数

```c
// https://elixir.bootlin.com/glibc/latest/source/libio/genops.c#L408
size_t
_IO_sgetn (FILE *fp, void *data, size_t n)
{
  /* FIXME handle putback buffer here! */
  return _IO_XSGETN (fp, data, n);
}
libc_hidden_def (_IO_sgetn)

// https://elixir.bootlin.com/glibc/latest/source/libio/libioP.h#L182
typedef size_t (*_IO_xsgetn_t) (FILE *FP, void *DATA, size_t N);
#define _IO_XSGETN(FP, DATA, N) JUMP2 (__xsgetn, FP, DATA, N)
#define _IO_WXSGETN(FP, DATA, N) WJUMP2 (__xsgetn, FP, DATA, N)
```

在源码中跟踪这个宏到这里就断了，为找实际函数调用，所以我写了一个实例代码用于gdb跟踪。

```c
#include <stdio.h>
#include <string.h>
 
int main()
{
   FILE *fp;
   char c[] = "This is runoob";
   char buffer[20];
   fp = fopen("file.txt", "w+");   /* 打开文件用于读写 */
   fwrite(c, strlen(c) + 1, 1, fp);   /* 写入数据到文件 */
   fseek(fp, 0, SEEK_SET);   /* 查找文件的开头 */
   fread(buffer, strlen(c)+1, 1, fp);   /* 读取并显示数据 */
   printf("%s\n", buffer);
   fclose(fp);
   return(0);
}
```

我们看一下汇编的`_IO_sgetn`函数

```assembly
0x7ffff7e0d050 <__GI__IO_sgetn>:     endbr64 
0x7ffff7e0d054 <__GI__IO_sgetn+4>:   push   rbx
0x7ffff7e0d055 <__GI__IO_sgetn+5>:   lea    rcx,[rip+0x1879a4]        # 0x7ffff7f94a00 <_IO_helper_jumps>
0x7ffff7e0d05c <__GI__IO_sgetn+12>:  lea    rax,[rip+0x188705]        # 0x7ffff7f95768
0x7ffff7e0d063 <__GI__IO_sgetn+19>:  sub    rax,rcx
0x7ffff7e0d066 <__GI__IO_sgetn+22>:  sub    rsp,0x20
0x7ffff7e0d06a <__GI__IO_sgetn+26>:  mov    rbx,QWORD PTR [rdi+0xd8]
0x7ffff7e0d071 <__GI__IO_sgetn+33>:  mov    r8,rbx
0x7ffff7e0d074 <__GI__IO_sgetn+36>:  sub    r8,rcx
0x7ffff7e0d077 <__GI__IO_sgetn+39>:  cmp    rax,r8
0x7ffff7e0d07a <__GI__IO_sgetn+42>:  jbe    0x7ffff7e0d090 <__GI__IO_sgetn+64>
0x7ffff7e0d07c <__GI__IO_sgetn+44>:  mov    rax,QWORD PTR [rbx+0x40]
0x7ffff7e0d080 <__GI__IO_sgetn+48>:  add    rsp,0x20
0x7ffff7e0d084 <__GI__IO_sgetn+52>:  pop    rbx
0x7ffff7e0d085 <__GI__IO_sgetn+53>:  jmp    rax
0x7ffff7e0d087 <__GI__IO_sgetn+55>:  nop    WORD PTR [rax+rax*1+0x0]
0x7ffff7e0d090 <__GI__IO_sgetn+64>:  mov    QWORD PTR [rsp+0x18],rdx
0x7ffff7e0d095 <__GI__IO_sgetn+69>:  mov    QWORD PTR [rsp+0x10],rsi
0x7ffff7e0d09a <__GI__IO_sgetn+74>:  mov    QWORD PTR [rsp+0x8],rdi
0x7ffff7e0d09f <__GI__IO_sgetn+79>:  call   0x7ffff7e08f70 <_IO_vtable_check>
0x7ffff7e0d0a4 <__GI__IO_sgetn+84>:  mov    rax,QWORD PTR [rbx+0x40]
0x7ffff7e0d0a8 <__GI__IO_sgetn+88>:  mov    rdx,QWORD PTR [rsp+0x18]
0x7ffff7e0d0ad <__GI__IO_sgetn+93>:  mov    rsi,QWORD PTR [rsp+0x10]
0x7ffff7e0d0b2 <__GI__IO_sgetn+98>:  mov    rdi,QWORD PTR [rsp+0x8]
0x7ffff7e0d0b7 <__GI__IO_sgetn+103>: add    rsp,0x20   
0x7ffff7e0d0bb <__GI__IO_sgetn+107>: pop    rbx
0x7ffff7e0d0bc <__GI__IO_sgetn+108>: jmp    rax
```

我们先分析前半部分，即**jbe**之前的部分

```assembly
0x7ffff7e0d055 <__GI__IO_sgetn+5>:   lea    rcx,[rip+0x1879a4]        # 0x7ffff7f94a00 <_IO_helper_jumps>
0x7ffff7e0d05c <__GI__IO_sgetn+12>:  lea    rax,[rip+0x188705]        # 0x7ffff7f95768
0x7ffff7e0d063 <__GI__IO_sgetn+19>:  sub    rax,rcx
0x7ffff7e0d066 <__GI__IO_sgetn+22>:  sub    rsp,0x20
0x7ffff7e0d06a <__GI__IO_sgetn+26>:  mov    rbx,QWORD PTR [rdi+0xd8]
0x7ffff7e0d071 <__GI__IO_sgetn+33>:  mov    r8,rbx
0x7ffff7e0d074 <__GI__IO_sgetn+36>:  sub    r8,rcx
0x7ffff7e0d077 <__GI__IO_sgetn+39>:  cmp    rax,r8
0x7ffff7e0d07a <__GI__IO_sgetn+42>:  jbe    0x7ffff7e0d090 <__GI__IO_sgetn+64>
```

**rdi**指向的就是`IO_FILE`结构体开头，所以**rdi+0xd8**就是**vtable**指向的值，其实就是判断**rax**是否小于等于**vtable**，然后决定跳转。

- 如果此时不跳转，即**vtable > rax**，会跳转到将**rbx+0x40**的地方，而由前半部分，我们知道**rbx**就是**vtable**

```assembly
0x7ffff7e0d07c <__GI__IO_sgetn+44>:  mov    rax,QWORD PTR [rbx+0x40]
0x7ffff7e0d080 <__GI__IO_sgetn+48>:  add    rsp,0x20
0x7ffff7e0d084 <__GI__IO_sgetn+52>:  pop    rbx
0x7ffff7e0d085 <__GI__IO_sgetn+53>:  jmp    rax
```

- 如果此时跳转，即**vtable <= rax**，同样会跳转到**rbx+0x40**的地方，不过区别是先进行了一次vtable的check

```assembly
0x7ffff7e0d090 <__GI__IO_sgetn+64>:  mov    QWORD PTR [rsp+0x18],rdx
0x7ffff7e0d095 <__GI__IO_sgetn+69>:  mov    QWORD PTR [rsp+0x10],rsi
0x7ffff7e0d09a <__GI__IO_sgetn+74>:  mov    QWORD PTR [rsp+0x8],rdi
0x7ffff7e0d09f <__GI__IO_sgetn+79>:  call   0x7ffff7e08f70 <_IO_vtable_check>
0x7ffff7e0d0a4 <__GI__IO_sgetn+84>:  mov    rax,QWORD PTR [rbx+0x40]
0x7ffff7e0d0a8 <__GI__IO_sgetn+88>:  mov    rdx,QWORD PTR [rsp+0x18]
0x7ffff7e0d0ad <__GI__IO_sgetn+93>:  mov    rsi,QWORD PTR [rsp+0x10]
0x7ffff7e0d0b2 <__GI__IO_sgetn+98>:  mov    rdi,QWORD PTR [rsp+0x8]
0x7ffff7e0d0b7 <__GI__IO_sgetn+103>: add    rsp,0x20
0x7ffff7e0d0bb <__GI__IO_sgetn+107>: pop    rbx
0x7ffff7e0d0bc <__GI__IO_sgetn+108>: jmp    rax
```

### _IO_file_xsgetn

![image-20221207172245186](img/article/fread/image-20221207172245186.png)

根据gdb动调的结果，**rbx+0x40**即**vtable**偏移为0x40处的函数是`_IO_file_xsgetn`，在源码中可以查到其定义如下：

```c
// https://elixir.bootlin.com/glibc/latest/source/libio/fileops.c#L1271
size_t
_IO_file_xsgetn (FILE *fp, void *data, size_t n)
{
  size_t want, have;
  ssize_t count;
  char *s = data;

  want = n;

  if (fp->_IO_buf_base == NULL)
    {
      /* Maybe we already have a push back pointer.  */
      if (fp->_IO_save_base != NULL)
	{
	  free (fp->_IO_save_base);
	  fp->_flags &= ~_IO_IN_BACKUP;
	}
      _IO_doallocbuf (fp);
    }

  while (want > 0)
    {
      have = fp->_IO_read_end - fp->_IO_read_ptr;
      if (want <= have)
	{
	  memcpy (s, fp->_IO_read_ptr, want);
	  fp->_IO_read_ptr += want;
	  want = 0;
	}
      else
	{
	  if (have > 0)
	    {
	      s = __mempcpy (s, fp->_IO_read_ptr, have);
	      want -= have;
	      fp->_IO_read_ptr += have;
	    }

	  /* Check for backup and repeat */
	  if (_IO_in_backup (fp))
	    {
	      _IO_switch_to_main_get_area (fp);
	      continue;
	    }

	  /* If we now want less than a buffer, underflow and repeat
	     the copy.  Otherwise, _IO_SYSREAD directly to
	     the user buffer. */
	  if (fp->_IO_buf_base
	      && want < (size_t) (fp->_IO_buf_end - fp->_IO_buf_base))
	    {
	      if (__underflow (fp) == EOF)
		break;

	      continue;
	    }

	  /* These must be set before the sysread as we might longjmp out
	     waiting for input. */
	  _IO_setg (fp, fp->_IO_buf_base, fp->_IO_buf_base, fp->_IO_buf_base);
	  _IO_setp (fp, fp->_IO_buf_base, fp->_IO_buf_base);

	  /* Try to maintain alignment: read a whole number of blocks.  */
	  count = want;
	  if (fp->_IO_buf_base)
	    {
	      size_t block_size = fp->_IO_buf_end - fp->_IO_buf_base;
	      if (block_size >= 128)
		count -= want % block_size;
	    }

	  count = _IO_SYSREAD (fp, s, count);
	  if (count <= 0)
	    {
	      if (count == 0)
		fp->_flags |= _IO_EOF_SEEN;
	      else
		fp->_flags |= _IO_ERR_SEEN;

	      break;
	    }

	  s += count;
	  want -= count;
	  if (fp->_offset != _IO_pos_BAD)
	    _IO_pos_adjust (fp->_offset, count);
	}
    }

  return n - want;
}
libc_hidden_def (_IO_file_xsgetn)
```

我们把这段代码拆分成几个部分分别来进行分析

#### Part-1

```c
if (fp->_IO_buf_base == NULL)
    {
      /* Maybe we already have a push back pointer.  */
      if (fp->_IO_save_base != NULL)
	{
	  free (fp->_IO_save_base);
	  fp->_flags &= ~_IO_IN_BACKUP;
	}
      _IO_doallocbuf (fp);
    }
```

开头是一个判断，如果 **_IO_buf_base** 为空即代表也许已经存在一个 **push back pointer**，然后会检查 **_IO_save_base**是否为空。如果不空，就会free掉这个指针，并且将 **_IO_IN_BACKUP**这一位置零。然后调用`_IO_doallocbuf`

```c
// https://elixir.bootlin.com/glibc/latest/source/libio/genops.c#L342
void
_IO_doallocbuf (FILE *fp)
{
  if (fp->_IO_buf_base)
    return;
  if (!(fp->_flags & _IO_UNBUFFERED) || fp->_mode > 0)
    if (_IO_DOALLOCATE (fp) != EOF)
      return;
  _IO_setb (fp, fp->_shortbuf, fp->_shortbuf+1, 0);
}
libc_hidden_def (_IO_doallocbuf)
```

这里如果**fp->_IO_buf_base** 非空，代表已经初始化完成，直接返回。反之在检测**flag**之后调用`_IO_DOALLOCATE`，其实就是`_IO_file_doallocate `函数：

```c
// https://elixir.bootlin.com/glibc/latest/source/libio/filedoalloc.c#L77
int
_IO_file_doallocate (FILE *fp)
{
  size_t size;
  char *p;
  struct __stat64_t64 st;

  size = BUFSIZ;
  if (fp->_fileno >= 0 && __builtin_expect (_IO_SYSSTAT (fp, &st), 0) >= 0)
    {
      if (S_ISCHR (st.st_mode))
	{
	  /* Possibly a tty.  */
	  if (
#ifdef DEV_TTY_P
	      DEV_TTY_P (&st) ||
#endif
	      local_isatty (fp->_fileno))
	    fp->_flags |= _IO_LINE_BUF;
	}
#if defined _STATBUF_ST_BLKSIZE
      if (st.st_blksize > 0 && st.st_blksize < BUFSIZ)
	size = st.st_blksize;
#endif
    }
  p = malloc (size);
  if (__glibc_unlikely (p == NULL))
    return EOF;
  _IO_setb (fp, p, p + size, 1);
  return 1;
}
libc_hidden_def (_IO_file_doallocate)
```

这个函数主要对`_IO_FILE`进行初始化，

- 首先，对**fileno**做最基本的检测，然后调用`__stat`，用于获取文件信息
- 然后调用malloc申请一个空间
- 调用`_IO_setb`,对 **_IO_buf_base**和 **_IO_buf_end**进行了赋值为刚刚申请 **chunk**的 **p**和 **p+1**

```c
// https://elixir.bootlin.com/glibc/latest/source/libio/genops.c#L328
void
_IO_setb (FILE *f, char *b, char *eb, int a)
{
  if (f->_IO_buf_base && !(f->_flags & _IO_USER_BUF))
    free (f->_IO_buf_base);
  f->_IO_buf_base = b;
  f->_IO_buf_end = eb;
  if (a)
    f->_flags &= ~_IO_USER_BUF;
  else
    f->_flags |= _IO_USER_BUF;
}
libc_hidden_def (_IO_setb)
```

#### Part-2

这里开始便是读入的主体部分

```c
while (want > 0)
    {
      have = fp->_IO_read_end - fp->_IO_read_ptr;
      if (want <= have)
	{
	  memcpy (s, fp->_IO_read_ptr, want);
	  fp->_IO_read_ptr += want;
	  want = 0;
	}
      else
	{
	  if (have > 0)
	    {
	      s = __mempcpy (s, fp->_IO_read_ptr, have);
	      want -= have;
	      fp->_IO_read_ptr += have;
	    }

	  /* Check for backup and repeat */
	  if (_IO_in_backup (fp))
	    {
	      _IO_switch_to_main_get_area (fp);
	      continue;
	    }

	  /* If we now want less than a buffer, underflow and repeat
	     the copy.  Otherwise, _IO_SYSREAD directly to
	     the user buffer. */
	  if (fp->_IO_buf_base
	      && want < (size_t) (fp->_IO_buf_end - fp->_IO_buf_base))
	    {
	      if (__underflow (fp) == EOF)
		break;

	      continue;
	    }

	  /* These must be set before the sysread as we might longjmp out
	     waiting for input. */
	  _IO_setg (fp, fp->_IO_buf_base, fp->_IO_buf_base, fp->_IO_buf_base);
	  _IO_setp (fp, fp->_IO_buf_base, fp->_IO_buf_base);

	  /* Try to maintain alignment: read a whole number of blocks.  */
	  count = want;
	  if (fp->_IO_buf_base)
	    {
	      size_t block_size = fp->_IO_buf_end - fp->_IO_buf_base;
	      if (block_size >= 128)
		count -= want % block_size;
	    }

	  count = _IO_SYSREAD (fp, s, count);
	  if (count <= 0)
	    {
	      if (count == 0)
		fp->_flags |= _IO_EOF_SEEN;
	      else
		fp->_flags |= _IO_ERR_SEEN;

	      break;
	    }

	  s += count;
	  want -= count;
	  if (fp->_offset != _IO_pos_BAD)
	    _IO_pos_adjust (fp->_offset, count);
	}
    }
```

可以看到如果，需要读入的长度小于当前`_IO_read_end`到`_IO_read_ptr`的区间，就会直接用`memcpy`进行拷贝赋值。如果不足的话，就会不断调整指针同样用`memcpy` 进行拷贝。直到读完成，然后返回**n-want** 即总读入字符数。

#### part-3

```c
......
/* If we now want less than a buffer, underflow and repeat
	     the copy.  Otherwise, _IO_SYSREAD directly to
	     the user buffer. */
	  if (fp->_IO_buf_base
	      && want < (size_t) (fp->_IO_buf_end - fp->_IO_buf_base))
	    {
	      if (__underflow (fp) == EOF)
		break;

	      continue;
	    }
......	  
```

在读入的过程中，如果遇到空间不足的情况，会调用`__underflow`去执行系统调用`read`读取数据，并放入到输入缓冲区里。

```c
// https://elixir.bootlin.com/glibc/latest/source/libio/genops.c#L268
int
__underflow (FILE *fp)
{
  if (_IO_vtable_offset (fp) == 0 && _IO_fwide (fp, -1) != -1)
    return EOF;

  if (fp->_mode == 0)
    _IO_fwide (fp, -1);
  if (_IO_in_put_mode (fp))
    if (_IO_switch_to_get_mode (fp) == EOF)
      return EOF;
  if (fp->_IO_read_ptr < fp->_IO_read_end)
    return *(unsigned char *) fp->_IO_read_ptr;
  if (_IO_in_backup (fp))
    {
      _IO_switch_to_main_get_area (fp);
      if (fp->_IO_read_ptr < fp->_IO_read_end)
	return *(unsigned char *) fp->_IO_read_ptr;
    }
  if (_IO_have_markers (fp))
    {
      if (save_for_backup (fp, fp->_IO_read_end))
	return EOF;
    }
  else if (_IO_have_backup (fp))
    _IO_free_backup_area (fp);
  return _IO_UNDERFLOW (fp);
}
libc_hidden_def (__underflow)
```

在对**flag**进行一系列检测后，会调用`_IO_UNDERFLOW`函数，同样利用动调，我们找到这个函数就是`_IO_file_underflow`，实际在源码中为`_IO_new_file_underflow`

![image-20221207200035078](img/article/fread/image-20221207200035078.png)

```c
// https://elixir.bootlin.com/glibc/latest/source/libio/fileops.c#L460
int
_IO_new_file_underflow (FILE *fp)
{
  ssize_t count;

  /* C99 requires EOF to be "sticky".  */
  if (fp->_flags & _IO_EOF_SEEN)
    return EOF;

  if (fp->_flags & _IO_NO_READS)
    {
      fp->_flags |= _IO_ERR_SEEN;
      __set_errno (EBADF);
      return EOF;
    }
  if (fp->_IO_read_ptr < fp->_IO_read_end)
    return *(unsigned char *) fp->_IO_read_ptr;

  if (fp->_IO_buf_base == NULL)
    {
      /* Maybe we already have a push back pointer.  */
      if (fp->_IO_save_base != NULL)
	{
	  free (fp->_IO_save_base);
	  fp->_flags &= ~_IO_IN_BACKUP;
	}
      _IO_doallocbuf (fp);
    }

  /* FIXME This can/should be moved to genops ?? */
  if (fp->_flags & (_IO_LINE_BUF|_IO_UNBUFFERED))
    {
      /* We used to flush all line-buffered stream.  This really isn't
	 required by any standard.  My recollection is that
	 traditional Unix systems did this for stdout.  stderr better
	 not be line buffered.  So we do just that here
	 explicitly.  --drepper */
      _IO_acquire_lock (stdout);

      if ((stdout->_flags & (_IO_LINKED | _IO_NO_WRITES | _IO_LINE_BUF))
	  == (_IO_LINKED | _IO_LINE_BUF))
	_IO_OVERFLOW (stdout, EOF);

      _IO_release_lock (stdout);
    }

  _IO_switch_to_get_mode (fp);

  /* This is very tricky. We have to adjust those
     pointers before we call _IO_SYSREAD () since
     we may longjump () out while waiting for
     input. Those pointers may be screwed up. H.J. */
  fp->_IO_read_base = fp->_IO_read_ptr = fp->_IO_buf_base;
  fp->_IO_read_end = fp->_IO_buf_base;
  fp->_IO_write_base = fp->_IO_write_ptr = fp->_IO_write_end
    = fp->_IO_buf_base;

  count = _IO_SYSREAD (fp, fp->_IO_buf_base,
		       fp->_IO_buf_end - fp->_IO_buf_base);
  if (count <= 0)
    {
      if (count == 0)
	fp->_flags |= _IO_EOF_SEEN;
      else
	fp->_flags |= _IO_ERR_SEEN, count = 0;
  }
  fp->_IO_read_end += count;
  if (count == 0)
    {
      /* If a stream is read to EOF, the calling application may switch active
	 handles.  As a result, our offset cache would no longer be valid, so
	 unset it.  */
      fp->_offset = _IO_pos_BAD;
      return EOF;
    }
  if (fp->_offset != _IO_pos_BAD)
    _IO_pos_adjust (fp->_offset, count);
  return *(unsigned char *) fp->_IO_read_ptr;
}
libc_hidden_ver (_IO_new_file_underflow, _IO_file_underflow)
```

这个函数主要做了这几件事：

- 首先进行一些检查，包括**flag**以及一些指针的合法性
- 然后对 **_IO_read_base， _IO_read_ptr， _IO_read_end， _IO_write_base**指针均设置为 **fp->_IO_buf_base**
-  调用`_IO_SYSREAD ` ,该函数最终执行系统调用read，读取文件数据，数据读入到`fp->_IO_buf_base`中，读入大小为输入缓冲区的大小`fp->_IO_buf_end - fp->_IO_buf_base`。
- 设置输入缓冲区已有数据的size，即设置`fp->_IO_read_end`为`fp->_IO_read_end += count`。

随后不断退出函数，直至结束







