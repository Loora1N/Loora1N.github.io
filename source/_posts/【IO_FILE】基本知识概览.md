---
title: 【IO_FILE】源码详解
index_img: img/article/fopen/t017448d9547dc07159-1669695992482-3.png
excerpt: 由于glibc版本的不断更新，常见几种hook在glibc2.34终被舍去，我们急需一种能够在高版本的glibc下仍能有效的getshell方法。而IO_FILE作为heap题目的进阶技能逐渐引起大家注意，吸引了许多大佬投入其中。仅在2021-2022年期间就出现出现了大量以IO_FILE为基础的调用链利用方式，在比赛中也逐渐成为了当下heap题目的主流getshell方式。本文以glibc2.36为例，从相关结构体以及fopen函数源码入手，详细介绍IO_FILE相关知识点。
---
> 由于glibc版本的不断更新，常见几种hook在glibc2.34终被舍去，我们急需一种能够在高版本的glibc下仍能有效的getshell方法。而IO_FILE作为heap题目的进阶技能逐渐引起大家注意，吸引了许多大佬投入其中。仅在在2021-2022年期间就出现出现了大量以IO_FILE为基础的调用链利用方式，在比赛中也逐渐成为了当下heap题目的主流getshell方式。本文以glibc2.36为例，从相关结构体以及fopen函数源码入手，详细介绍IO_FILE相关知识点。

## _IO_FILE

IO_FILE结构体定义为`struct _IO_FILE`：

```c
/* The tag name of this struct is _IO_FILE to preserve historic
   C++ mangled names for functions taking FILE* arguments.
   That name should not be used in new code.  */
struct _IO_FILE
{
  int _flags;		/* High-order word is _IO_MAGIC; rest is flags. */

  /* The following pointers correspond to the C++ streambuf protocol. */
  char *_IO_read_ptr;	/* Current read pointer */
  char *_IO_read_end;	/* End of get area. */
  char *_IO_read_base;	/* Start of putback+get area. */
  char *_IO_write_base;	/* Start of put area. */
  char *_IO_write_ptr;	/* Current put pointer. */
  char *_IO_write_end;	/* End of put area. */
  char *_IO_buf_base;	/* Start of reserve area. */
  char *_IO_buf_end;	/* End of reserve area. */

  /* The following fields are used to support backing up and undo. */
  char *_IO_save_base; /* Pointer to start of non-current get area. */
  char *_IO_backup_base;  /* Pointer to first valid character of backup area */
  char *_IO_save_end; /* Pointer to end of non-current get area. */

  struct _IO_marker *_markers;

  struct _IO_FILE *_chain;

  int _fileno;
  int _flags2;
  __off_t _old_offset; /* This used to be _offset but it's too small.  */

  /* 1+column number of pbase(); 0 is unknown. */
  unsigned short _cur_column;
  signed char _vtable_offset;
  char _shortbuf[1];

  _IO_lock_t *_lock;
#ifdef _IO_USE_OLD_IO_FILE
};

struct _IO_FILE_complete
{
  struct _IO_FILE _file;
#endif
  __off64_t _offset;
  /* Wide character stream stuff.  */
  struct _IO_codecvt *_codecvt;
  struct _IO_wide_data *_wide_data;
  struct _IO_FILE *_freeres_list;
  void *_freeres_buf;
  size_t __pad5;
  int _mode;
  /* Make sure we don't get into trouble again.  */
  char _unused2[15 * sizeof (int) - 4 * sizeof (void *) - sizeof (size_t)];
};
```

然后让我们看这个结构中的一些重要部分：

- `_flags`:高位字为`_IO_MAGIC`，剩余的部分是flag
- `_IO_read_ptr`正在使用的input缓冲区的input地址
- `_IO_read_end` input缓冲区的结束地址
- `_IO_read_base` input缓冲区的基址
- `_IO_write_base` output缓冲区的基址
- `_IO_write_ptr` 指向还没输出的那个字节
- `_IO_write_end` output缓冲区的结束地址
- `_IO_buf_base` input和output缓冲区的基址
- `_IO_buf_end`  input和output缓冲区的结束地址
- `_chain` 存放着一个单链表，用于串联所有的file stream
- `_fileno` 与文件相关的文件描述符
- `_vtable_offset` 存放虚表(virtual table)的偏移
- `_offset` 存放当前文件的偏移

## _IO_FILE_plus

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/libioP.h#L324

struct _IO_FILE_plus
{
  FILE file;
  const struct _IO_jump_t *vtable;
};
```

这里FILE其实就是_IO_FILE，换了个名字罢了，如下所示。不得不说，glibc源码里宏定义是真的多😅，有点恶心人。

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/bits/types/FILE.h#L7

#ifndef __FILE_defined
#define __FILE_defined 1

struct _IO_FILE;

/* The opaque type of streams.  This is the definition used elsewhere.  */
typedef struct _IO_FILE FILE;

#endif
```

## _IO_jump_t

可以注意到，在`_IO_FILE_plus`中，前一部分是`_IO_FILE`结构体，后一部分是指向`_IO_jump_t`结构体的**vtable**指针，这里的vtable就是**virtual table**的缩写，即我们常说的虚表。所以`_IO_jump_t`其实就是虚表,源码定义如下：

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/libioP.h#L293

struct _IO_jump_t
{
    JUMP_FIELD(size_t, __dummy);
    JUMP_FIELD(size_t, __dummy2);
    JUMP_FIELD(_IO_finish_t, __finish);
    JUMP_FIELD(_IO_overflow_t, __overflow);
    JUMP_FIELD(_IO_underflow_t, __underflow);
    JUMP_FIELD(_IO_underflow_t, __uflow);
    JUMP_FIELD(_IO_pbackfail_t, __pbackfail);
    /* showmany */
    JUMP_FIELD(_IO_xsputn_t, __xsputn);
    JUMP_FIELD(_IO_xsgetn_t, __xsgetn);
    JUMP_FIELD(_IO_seekoff_t, __seekoff);
    JUMP_FIELD(_IO_seekpos_t, __seekpos);
    JUMP_FIELD(_IO_setbuf_t, __setbuf);
    JUMP_FIELD(_IO_sync_t, __sync);
    JUMP_FIELD(_IO_doallocate_t, __doallocate);
    JUMP_FIELD(_IO_read_t, __read);
    JUMP_FIELD(_IO_write_t, __write);
    JUMP_FIELD(_IO_seek_t, __seek);
    JUMP_FIELD(_IO_close_t, __close);
    JUMP_FIELD(_IO_stat_t, __stat);
    JUMP_FIELD(_IO_showmanyc_t, __showmanyc);
    JUMP_FIELD(_IO_imbue_t, __imbue);
};
```

所以我们可以看到，`_IO_FILE_plus`的总体结构如下图

![img](img/article/fopen/t017448d9547dc07159-1669695992482-3.png)

## fopen

让我们复现一下打开一个文件的过程，以及`_IO_FILE`结构体如何初始化，fopen的源码如下：

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/include/stdio.h#L191

extern FILE *_IO_new_fopen (const char*, const char*);
#   define fopen(fname, mode) _IO_new_fopen (fname, mode)

// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/iofopen.c#L83

FILE *
_IO_new_fopen (const char *filename, const char *mode)
{
  return __fopen_internal (filename, mode, 1);
}
```

可以看到又是一个宏然后，源码实际定义为`_IO_new_fopen`。

> 真爱生命，远离宏定义

然后它会调用`__fopen_internal`：

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/iofopen.c#L56

FILE *
__fopen_internal (const char *filename, const char *mode, int is32)
{
  struct locked_FILE
  {
    struct _IO_FILE_plus fp;
#ifdef _IO_MTSAFE_IO
    _IO_lock_t lock;
#endif
    struct _IO_wide_data wd;
  } *new_f = (struct locked_FILE *) malloc (sizeof (struct locked_FILE));

  if (new_f == NULL)
    return NULL;
#ifdef _IO_MTSAFE_IO
  new_f->fp.file._lock = &new_f->lock;
#endif
  _IO_no_init (&new_f->fp.file, 0, 0, &new_f->wd, &_IO_wfile_jumps);
  _IO_JUMPS (&new_f->fp) = &_IO_file_jumps;
  _IO_new_file_init_internal (&new_f->fp);
  if (_IO_file_fopen ((FILE *) new_f, filename, mode, is32) != NULL)
    return __fopen_maybe_mmap (&new_f->fp.file);

  _IO_un_link (&new_f->fp);
  free (new_f);
  return NULL;
}
```

在这个函数中首先通过malloc了一个`locked_FILE`结构体，其中就包含着`_IO_FILE_plus`，并将整个空间由指针**new_f**指向。`_IO_no_init`和`_IO_new_file_init_internal `对虚表进行了初始化。

先来看看`_IO_no_init`函数，开头就调用了`_IO_old_init`

 ```c
 // https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/genops.c#L561
 void
 _IO_no_init (FILE *fp, int flags, int orientation,
 	     struct _IO_wide_data *wd, const struct _IO_jump_t *jmp)
 {
   _IO_old_init (fp, flags);
   fp->_mode = orientation;
   if (orientation >= 0)
     {
       fp->_wide_data = wd;
       fp->_wide_data->_IO_buf_base = NULL;
       fp->_wide_data->_IO_buf_end = NULL;
       fp->_wide_data->_IO_read_base = NULL;
       fp->_wide_data->_IO_read_ptr = NULL;
       fp->_wide_data->_IO_read_end = NULL;
       fp->_wide_data->_IO_write_base = NULL;
       fp->_wide_data->_IO_write_ptr = NULL;
       fp->_wide_data->_IO_write_end = NULL;
       fp->_wide_data->_IO_save_base = NULL;
       fp->_wide_data->_IO_backup_base = NULL;
       fp->_wide_data->_IO_save_end = NULL;
 
       fp->_wide_data->_wide_vtable = jmp;
     }
   else
     /* Cause predictable crash when a wide function is called on a byte
        stream.  */
     fp->_wide_data = (struct _IO_wide_data *) -1L;
   fp->_freeres_list = NULL;
 }
 
 
 
 // https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/genops.c#L530
 void
 _IO_old_init (FILE *fp, int flags)
 {
   fp->_flags = _IO_MAGIC|flags;
   fp->_flags2 = 0;
   if (stdio_needs_locking)
     fp->_flags2 |= _IO_FLAGS2_NEED_LOCK;
   fp->_IO_buf_base = NULL;
   fp->_IO_buf_end = NULL;
   fp->_IO_read_base = NULL;
   fp->_IO_read_ptr = NULL;
   fp->_IO_read_end = NULL;
   fp->_IO_write_base = NULL;
   fp->_IO_write_ptr = NULL;
   fp->_IO_write_end = NULL;
   fp->_chain = NULL; /* Not necessary. */
 v
   fp->_IO_save_base = NULL;
   fp->_IO_backup_base = NULL;
   fp->_IO_save_end = NULL;
   fp->_markers = NULL;
   fp->_cur_column = 0;
 #if _IO_JUMPS_OFFSET
   fp->_vtable_offset = 0;
 #endif
 #ifdef _IO_MTSAFE_IO
   if (fp->_lock != NULL)
     _IO_lock_init (*fp->_lock);
 #endif
 }
 ```

> `_IO_old_init`函数有两个重载，上面调用的是2参数，即`_IO_old_init (FILE *fp, int flags)`；另外还有一个5参数`_IO_no_init (FILE *fp, int flags, int orientation,struct _IO_wide_data *wd, const struct _IO_jump_t *jmp)`，源码就定义在2参数的后面

可以看到，首先`_IO_old_init`对`_IO_FILE`的主要几个成员变量进行了初始化，然后由`_IO_no_init`会根据传入参数**orientation**的值决定是否对`_wide_data`相关值做初始化(同时需注意到`fopen`是直接常量0传入，所以是必定初始化该区域)。

然后接下来执行了 **_IO_JUMPS (&new_f->fp) =  & _IO_file_jumps**，这里的`_IO_JUMPS`其实就是**vtable**指针，而`_IO_FILE_jumps`就是我们之前谈到的虚表。相当于让**vtable**指针指向了**虚表**，`_IO_JUMPS`和`_IO_FILE_jumps`源码如下：

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/libioP.h#L98
#define _IO_JUMPS(THIS) (THIS)->vtable

// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/fileops.c#L1432
const struct _IO_jump_t _IO_file_jumps libio_vtable =
{
  JUMP_INIT_DUMMY,
  JUMP_INIT(finish, _IO_file_finish),
  JUMP_INIT(overflow, _IO_file_overflow),
  JUMP_INIT(underflow, _IO_file_underflow),
  JUMP_INIT(uflow, _IO_default_uflow),
  JUMP_INIT(pbackfail, _IO_default_pbackfail),
  JUMP_INIT(xsputn, _IO_file_xsputn),
  JUMP_INIT(xsgetn, _IO_file_xsgetn),
  JUMP_INIT(seekoff, _IO_new_file_seekoff),
  JUMP_INIT(seekpos, _IO_default_seekpos),
  JUMP_INIT(setbuf, _IO_new_file_setbuf),
  JUMP_INIT(sync, _IO_new_file_sync),
  JUMP_INIT(doallocate, _IO_file_doallocate),
  JUMP_INIT(read, _IO_file_read),
  JUMP_INIT(write, _IO_new_file_write),
  JUMP_INIT(seek, _IO_file_seek),
  JUMP_INIT(close, _IO_file_close),
  JUMP_INIT(stat, _IO_file_stat),
  JUMP_INIT(showmanyc, _IO_default_showmanyc),
  JUMP_INIT(imbue, _IO_default_imbue)
};
```

接着调用了`_IO_new_file_init_internal`,源码如下:

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/fileops.c#L105
void
_IO_new_file_init_internal (struct _IO_FILE_plus *fp)
{
  /* POSIX.1 allows another file handle to be used to change the position
     of our file descriptor.  Hence we actually don't know the actual
     position before we do the first fseek (and until a following fflush). */
  fp->file._offset = _IO_pos_BAD;
  fp->file._flags |= CLOSED_FILEBUF_FLAGS;

  _IO_link_in (fp);
  fp->file._fileno = -1;
}
```

其中，`_IO_FILE`的**flag**被设置**CLOSED_FILEBUF_FLAGS**，具体含义可看如下定义：

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/fileops.c#L100
#define CLOSED_FILEBUF_FLAGS \
  (_IO_IS_FILEBUF+_IO_NO_READS+_IO_NO_WRITES+_IO_TIED_PUT_GET)

// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/libio.h#L81
#define _IO_IS_FILEBUF        0x2000
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/libio.h#L70
#define _IO_NO_READS          0x0004 /* Reading not allowed.  */
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/libio.h#L71
#define _IO_NO_WRITES         0x0008 /* Writing not allowed.  */
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/libio.h#L78
#define _IO_TIED_PUT_GET      0x0400 /* Put and get pointer move in unison.  */
```

随后调用了`_IO_link_in`，首先判断 **_flags**是否已经被link过。如果没有，就将 **_chain**赋值为 **_IO_list_all**。其实就是一个一个往单链表上增添节点，串联了所有的`_IO_FILE_plus`

> **_flags**相关判断和附加主要通过位运算实现，这里不做这种程度的具体解释(都开始研究IO_FILE了，总不能位运算还搞不明白吧)

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/genops.c#L86
void
_IO_link_in (struct _IO_FILE_plus *fp)
{
  if ((fp->file._flags & _IO_LINKED) == 0)
    {
      fp->file._flags |= _IO_LINKED;
#ifdef _IO_MTSAFE_IO
      _IO_cleanup_region_start_noarg (flush_cleanup);
      _IO_lock_lock (list_all_lock);
      run_fp = (FILE *) fp;
      _IO_flockfile ((FILE *) fp);
#endif
      fp->file._chain = (FILE *) _IO_list_all;
      _IO_list_all = fp;
#ifdef _IO_MTSAFE_IO
      _IO_funlockfile ((FILE *) fp);
      run_fp = NULL;
      _IO_lock_unlock (list_all_lock);
      _IO_cleanup_region_end (0);
#endif
    }
}
libc_hidden_def (_IO_link_in)
```

在link完成后，就会接着调用`_IO_file_fopen`，实际函数定义的为` _IO_new_file_fopen`

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/fileops.c#L211

FILE *
_IO_new_file_fopen (FILE *fp, const char *filename, const char *mode,
		    int is32not64)
{
  int oflags = 0, omode;
  int read_write;
  int oprot = 0666;
  int i;
  FILE *result;
  const char *cs;
  const char *last_recognized;

  if (_IO_file_is_open (fp))
    return 0;
  switch (*mode)
    {
    case 'r':
      omode = O_RDONLY;
      read_write = _IO_NO_WRITES;
      break;
    case 'w':
      omode = O_WRONLY;
      oflags = O_CREAT|O_TRUNC;
      read_write = _IO_NO_READS;
      break;
    case 'a':
      omode = O_WRONLY;
      oflags = O_CREAT|O_APPEND;
      read_write = _IO_NO_READS|_IO_IS_APPENDING;
      break;
    default:
      __set_errno (EINVAL);
      return NULL;
    }
  last_recognized = mode;
  for (i = 1; i < 7; ++i)
    {
      switch (*++mode)
	{
	case '\0':
	  break;
	case '+':
	  omode = O_RDWR;
	  read_write &= _IO_IS_APPENDING;
	  last_recognized = mode;
	  continue;
	case 'x':
	  oflags |= O_EXCL;
	  last_recognized = mode;
	  continue;
	case 'b':
	  last_recognized = mode;
	  continue;
	case 'm':
	  fp->_flags2 |= _IO_FLAGS2_MMAP;
	  continue;
	case 'c':
	  fp->_flags2 |= _IO_FLAGS2_NOTCANCEL;
	  continue;
	case 'e':
	  oflags |= O_CLOEXEC;
	  fp->_flags2 |= _IO_FLAGS2_CLOEXEC;
	  continue;
	default:
	  /* Ignore.  */
	  continue;
	}
      break;
    }

  result = _IO_file_open (fp, filename, omode|oflags, oprot, read_write,
			  is32not64);

  if (result != NULL)
    {
      /* Test whether the mode string specifies the conversion.  */
      cs = strstr (last_recognized + 1, ",ccs=");
      if (cs != NULL)
	{
	  /* Yep.  Load the appropriate conversions and set the orientation
	     to wide.  */
	  struct gconv_fcts fcts;
	  struct _IO_codecvt *cc;
	  char *endp = __strchrnul (cs + 5, ',');
	  char *ccs = malloc (endp - (cs + 5) + 3);

	  if (ccs == NULL)
	    {
	      int malloc_err = errno;  /* Whatever malloc failed with.  */
	      (void) _IO_file_close_it (fp);
	      __set_errno (malloc_err);
	      return NULL;
	    }

	  *((char *) __mempcpy (ccs, cs + 5, endp - (cs + 5))) = '\0';
	  strip (ccs, ccs);

	  if (__wcsmbs_named_conv (&fcts, ccs[2] == '\0'
				   ? upstr (ccs, cs + 5) : ccs) != 0)
	    {
	      /* Something went wrong, we cannot load the conversion modules.
		 This means we cannot proceed since the user explicitly asked
		 for these.  */
	      (void) _IO_file_close_it (fp);
	      free (ccs);
	      __set_errno (EINVAL);
	      return NULL;
	    }

	  free (ccs);

	  assert (fcts.towc_nsteps == 1);
	  assert (fcts.tomb_nsteps == 1);

	  fp->_wide_data->_IO_read_ptr = fp->_wide_data->_IO_read_end;
	  fp->_wide_data->_IO_write_ptr = fp->_wide_data->_IO_write_base;

	  /* Clear the state.  We start all over again.  */
	  memset (&fp->_wide_data->_IO_state, '\0', sizeof (__mbstate_t));
	  memset (&fp->_wide_data->_IO_last_state, '\0', sizeof (__mbstate_t));

	  cc = fp->_codecvt = &fp->_wide_data->_codecvt;

	  cc->__cd_in.step = fcts.towc;

	  cc->__cd_in.step_data.__invocation_counter = 0;
	  cc->__cd_in.step_data.__internal_use = 1;
	  cc->__cd_in.step_data.__flags = __GCONV_IS_LAST;
	  cc->__cd_in.step_data.__statep = &result->_wide_data->_IO_state;

	  cc->__cd_out.step = fcts.tomb;

	  cc->__cd_out.step_data.__invocation_counter = 0;
	  cc->__cd_out.step_data.__internal_use = 1;
	  cc->__cd_out.step_data.__flags = __GCONV_IS_LAST | __GCONV_TRANSLIT;
	  cc->__cd_out.step_data.__statep = &result->_wide_data->_IO_state;

	  /* From now on use the wide character callback functions.  */
	  _IO_JUMPS_FILE_plus (fp) = fp->_wide_data->_wide_vtable;

	  /* Set the mode now.  */
	  result->_mode = 1;
	}
    }

  return result;
}
libc_hidden_ver (_IO_new_file_fopen, _IO_file_fopen)
```

这个函数，首先检查了是否正常打开文件，紧接着对**omode**和**oflags**进行了设置，即文件权限相关设置。直接进入`_IO_file_open`查看

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/fileops.c#L180
FILE *
_IO_file_open (FILE *fp, const char *filename, int posix_mode, int prot,
	       int read_write, int is32not64)
{
  int fdesc;
  if (__glibc_unlikely (fp->_flags2 & _IO_FLAGS2_NOTCANCEL))
    fdesc = __open_nocancel (filename,
			     posix_mode | (is32not64 ? 0 : O_LARGEFILE), prot);
  else
    fdesc = __open (filename, posix_mode | (is32not64 ? 0 : O_LARGEFILE), prot);
  if (fdesc < 0)
    return NULL;
  fp->_fileno = fdesc;
  _IO_mask_flags (fp, read_write,_IO_NO_READS+_IO_NO_WRITES+_IO_IS_APPENDING);
  /* For append mode, send the file offset to the end of the file.  Don't
     update the offset cache though, since the file handle is not active.  */
  if ((read_write & (_IO_IS_APPENDING | _IO_NO_READS))
      == (_IO_IS_APPENDING | _IO_NO_READS))
    {
      off64_t new_pos = _IO_SYSSEEK (fp, 0, _IO_seek_end);
      if (new_pos == _IO_pos_BAD && errno != ESPIPE)
	{
	  __close_nocancel (fdesc);
	  return NULL;
	}
    }
  _IO_link_in ((struct _IO_FILE_plus *) fp);
  return fp;
}
libc_hidden_def (_IO_file_open)
```

首先会根据**flag2**决定调用`__open_nocancel`或`open`。打开文件后利用`_IO_mask_flags`重新初始化**fp->flag**,随后调用`_IO_link_in`初始化chain域，下面是`_IO_mask_flags`和`_IO_link_in`的源码

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/libioP.h#L518
#define _IO_mask_flags(fp, f, mask) \
       ((fp)->_flags = ((fp)->_flags & ~(mask)) | ((f) & (mask)))

// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/genops.c#L86
void
_IO_link_in (struct _IO_FILE_plus *fp)
{
  if ((fp->file._flags & _IO_LINKED) == 0)
    {
      fp->file._flags |= _IO_LINKED;
#ifdef _IO_MTSAFE_IO
      _IO_cleanup_region_start_noarg (flush_cleanup);
      _IO_lock_lock (list_all_lock);
      run_fp = (FILE *) fp;
      _IO_flockfile ((FILE *) fp);
#endif
      fp->file._chain = (FILE *) _IO_list_all;
      _IO_list_all = fp;
#ifdef _IO_MTSAFE_IO
      _IO_funlockfile ((FILE *) fp);
      run_fp = NULL;
      _IO_lock_unlock (list_all_lock);
      _IO_cleanup_region_end (0);
#endif
    }
}
```

回到`__fopen_internal`函数中，可以看到接下来调用了`__fopen_maybe_mmap`，由注释我们可知，这里是根据文件的权限，做一些mmap的处理，然后返回即可。

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/iofopen.c#L34
FILE *
__fopen_maybe_mmap (FILE *fp)
{
#if _G_HAVE_MMAP
  if ((fp->_flags2 & _IO_FLAGS2_MMAP) && (fp->_flags & _IO_NO_WRITES))
    {
      /* Since this is read-only, we might be able to mmap the contents
	 directly.  We delay the decision until the first read attempt by
	 giving it a jump table containing functions that choose mmap or
	 vanilla file operations and reset the jump table accordingly.  */

      if (fp->_mode <= 0)
	_IO_JUMPS_FILE_plus (fp) = &_IO_file_jumps_maybe_mmap;
      else
	_IO_JUMPS_FILE_plus (fp) = &_IO_wfile_jumps_maybe_mmap;
      fp->_wide_data->_wide_vtable = &_IO_wfile_jumps_maybe_mmap;
    }
#endif
  return fp;
}
```

然后调用了`_IO_un_link`进行将该文件进行解链操作，随后free掉。

```c
// https://elixir.bootlin.com/glibc/glibc-2.36/source/libio/genops.c#L52
void
_IO_un_link (struct _IO_FILE_plus *fp)
{
  if (fp->file._flags & _IO_LINKED)
    {
      FILE **f;
#ifdef _IO_MTSAFE_IO
      _IO_cleanup_region_start_noarg (flush_cleanup);
      _IO_lock_lock (list_all_lock);
      run_fp = (FILE *) fp;
      _IO_flockfile ((FILE *) fp);
#endif
      if (_IO_list_all == NULL)
	;
      else if (fp == _IO_list_all)
	_IO_list_all = (struct _IO_FILE_plus *) _IO_list_all->file._chain;
      else
	for (f = &_IO_list_all->file._chain; *f; f = &(*f)->_chain)
	  if (*f == (FILE *) fp)
	    {
	      *f = fp->file._chain;
	      break;
	    }
      fp->file._flags &= ~_IO_LINKED;
#ifdef _IO_MTSAFE_IO
      _IO_funlockfile ((FILE *) fp);
      run_fp = NULL;
      _IO_lock_unlock (list_all_lock);
      _IO_cleanup_region_end (0);
#endif
    }
}
libc_hidden_def (_IO_un_link)
```

这就是fopen的整个过程了，这里我们小小总结一下流程：

- 首先利用`malloc`创建包含`_IO_FILE_plus`和`_IO_wide_data`的`locked_FILE`结构体
- 接着调用`_IO_no_init`函数对`_IO_FILE`的主要几个成员变量进行了初始化，以及`_wide_data`相关值做初始化
- 赋值虚表给vtable
- 调用`_IO_new_file_init_internal`进一步初始化，主要是对**_chain**进行操作
- 调用`_IO_file_fopen`打开文件
- 根据文件的权限，用`__fopen_maybe_mmap`做一些可能的mmap操作
- 脱离单链表链
- free







