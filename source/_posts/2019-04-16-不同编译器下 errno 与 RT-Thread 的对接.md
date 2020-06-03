
---
title: '不同编译器下 errno 与 RT-Thread 的对接'
tags:
  - 'RT-Thread'
  - 'errno'
categories: 操作系统
date: 2019-04-16
index_img: https://gitee.com/zhaojuntao/PictureBed/raw/master/blog/ArticleImage/blog.jpg
---

## 支持的编译器

RT-Thread 支持的编译器有 newlib/minilibc/dlib/armlibc 的支持。

- 在开启了 RT_USING_LIBC 后，GCC 编译使用 newlib
- 未开启 RT_USING_LIBC 时，GCC 编译使用 minilibc
- dlib 是 RT-Thread 针对 IAR 编译器的移植适配（使用标准库接口时注意开启 RT_USING_LIBC）
- armlibc 是 RT-Thread 针对 MDK 编译器的移植适配

## errno 的重定向

通常 errno 的声明在 `errno.h` 文件中。`errno.h` 在 RT-Thread 中的引入顺序如下所示：

```
rtthread.h -> rtdef.h -> rtlibc.h -> libc_errno.h
```

因此，如果需要对 errno 进行重定向，那么需要在引入 `errno.h` 之后，重新在 rtthread.h 中实现 errno 设置函数。

在 rtthread.h 中对 errno 的重定向如下所示：

```
#if !defined(RT_USING_NEWLIB) && !defined(_WIN32)
#ifndef errno
#define errno    *_rt_errno()
#endif
#endif
```

如上代码所示，当使用非 NEWLIB 编译器，非 WIN32 的时候，并且没有定义过 errno，才会对 errno 进行重定向。

在 libc_errno.h 中，errno.h 的引入如下所示：

```
#if defined(RT_USING_NEWLIB) || defined(_WIN32)
/* use errno.h file in toolchains */
#include <errno.h>
#endif
```

总结：

- 当使用 NWELIB 编译器时

    在 rtthread.h 中没有重定向 errno，在 libc_errno.h 中引入了 errno.h，在 rt-thread/components/libc/compilers/newlib/syscalls.c 中对 errno 函数进行了再次实现，如下所示。因此，使用 NEWLIB 可以正确重定向 errno。

    ```
    #ifndef _REENT_ONLY
    int *
    __errno ()
    {
    return _rt_errno();
    }
    #endif
    ```

- 当使用 MINILIB 编译器时

    未开启 RT_USING_LIBC，使用 GCC 编译，这个时候使用 MINILIB 编译器编译。这时，libc_errno.h 中没有引入 errno.h，也就未定义 errno，那么 rtthread.h 中就将 errno 重定向到 _rt_errno() 函数。

- 当使用 IAR 编译器时

    同 MINILIB。

- 当使用 ARMCC 编译器时

    同 MINILIB。

**注意：**

以上所有的原则是，没有在 rtthread.h 之前的其他任何地方引入 errno.h。如果在 rtthread.h 之前引入了 errno.h，那么就会定义 errno，那么在 rtthread.h 中就不会将 errno 重定向到 _rt_errno() 函数。

> 在 lwip 的其他地方有引入 errno.h，需要注意。
