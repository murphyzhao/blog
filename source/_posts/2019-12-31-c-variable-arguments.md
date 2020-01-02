---
title: 'C 语言 #、##、__VA_ARGS__、#__VA_ARGS__、##__VA_ARGS__'
tags:
  - C语言
  - __VA_ARGS__
categories: Programing
abbrlink: '40137057'
date: 2019-12-31
index_img: https://gitee.com/zhaojuntao/PictureBed/raw/master/blog/ArticleImage/C-2020-01-02.png
---

```
‘#’ 和 ‘##’ 属于预处理标记。‘#’ 和 ‘##’ 用于类似函数的宏定义中（或者简称为宏定义函数）。
‘__VA_ARGS__’ 是 C99 引入的用于支持宏定义函数中使用可变参数。
```

## 操作符 ‘#’

在宏定义展开的时候，标记 ‘#’ 用于将 ‘#’ 后面的宏定义函数中的参数转化为对应的字符串。宏定义函数的参数与预处理标记 ‘#’ 之间出现的每一个空格都会被删除，并删除第一个预处理标记之前和最后一个预处理标记之后的空白字符，但是宏定义函数参数中的空格会保留。

其中，空参数转化为为空，即宏定义函数入参为空，那么展开的时候也为空。

上面的这段话比较难理解，这里为了准确地传达其意义，我们来看一个示例程序。

> 在看到代码后，可以先猜猜可能的输出结果，如果你答对了，那就是真的会了！
> 注意，这里我基于 RT-Thread QEMU BSP 进行代码展示，代码真实编译通过，运行正常。

### 示例程序 A

请看以下代码：

```
#include <stdint.h>
#include <rtthread.h>

#define mkstr(var) (#var)

int main(void)
{
    rt_kprintf("hello rt-thread\n");

    rt_kprintf(mkstr(hello rt-thread));

    return 0;
}
```

请问：

- 它能编译通过吗？
- 它能输出什么内容？

答案：

- 它可以正常编译通过
- 它输出的内容

    ```
    hello rt-thread
    hello rt-threadmsh />
    ```

从上面输出的信息可以看到，`hello rt-thread` 字符串被准确地输出到了控制台，但是没有增加回车换行。其中 `msh />` 字符串是 RT-Thread 控制台回显。

如上，代码 `rt_kprintf(mkstr(hello rt-thread));` 中的 `hello rt-thread` 在没有加引号的情况下，被转化成了字符串。

### 示例程序 B

为示例程序 A 打印的字符串增加回车换行。

```
#include <stdint.h>
#include <rtthread.h>

#define mkstr(var) (#var)

int main(void)
{
    rt_kprintf("hello rt-thread\n");

    rt_kprintf(mkstr(hello rt-thread\r\n));

    return 0;
}
```

有了示例 A 的基础，示例 B 那就是 soeasy，直接在原有的基础上增加 `\r\n` 转义字符即可输出回车换行。

输出结果如下：

```
hello rt-thread
hello rt-thread
msh />
```

### 示例程序 C

我们在示例程序 B 中成功增加了回车换行的输出，但是你有没有想过一个问题，如果你又很多地方用到 `mkstr` 宏定义函数输出信息，那你是不是每一个地方都要增加 `\r\n`，这岂不是很累，有没有好的方法？

好方法当然有，下面介绍下程序中常用的方式，利用 C 语言相邻字符串自动拼接的特性（当然，这是编译器支持的）。代码如下：

```
#include <stdint.h>
#include <rtthread.h>

#define mkstr(var) (#var"\r\n")

int main(void)
{
    rt_kprintf("hello rt-thread\n");

    rt_kprintf(mkstr(hello rt-thread));

    return 0;
}
```

以上代码在 `#var` 后面增加了一个字符串 `\r\n`，我们来看宏定义展开过程：

```
-> rt_kprintf(mkstr(hello rt-thread));
-> rt_kprintf("hello rt-thread""\r\n");
-> rt_kprintf("hello rt-thread\r\n");
```

### 示例程序 D

预处理标记 `#` 的基本用法已经展示完了，但怎么理解 “宏定义函数的参数与预处理标记 ‘#’ 之间出现的每一个空格都会被删除，并删除第一个预处理标记之前和最后一个预处理标记之后的空白字符”？

请看下面的代码：

```
#include <stdint.h>
#include <rtthread.h>

#define mkstr(var) ("aa"  #  var  "bb")

int main(void)
{
    rt_kprintf("hello rt-thread\n");

    rt_kprintf(mkstr(hello rt-thread));

    return 0;
}
```

宏定义 `mkstr(var) ("aa"  #  var  "bb\r\n")` 中的 `#  var` 中间有两个空格，根据定义，`#` 号与宏定义函数参数 `var` 中间的两个空格会被删除，但是 `var` 参数中的 `hello rt-thread` 中的空格不会被删除。

继续，根据定义，宏定义 `mkstr(var) ("aa"  #  var  "bb\r\n")` 中只有一个预处理标记 `#`，其作为第一个和最后一个预处理标记，它前面和后面的空格都会被删除。

因此，以上代码预计输出结果为:

```
hello rt-thread
aahello rt-threadbb
msh />
```

### 示例程序 E

在实际操作时，操作符 `#` 被常用于枚举转字符串。以下代码截取自我的 [FlexibleButton 按键库](https://github.com/murphyzhao/FlexibleButton/blob/master/flexible_button_demo.c) 的示例程序。

```
#define ENUM_TO_STR(e) (#e)

typedef enum
{
    USER_BUTTON_0 = 0,
    USER_BUTTON_1,
    USER_BUTTON_2,
    USER_BUTTON_3,
    USER_BUTTON_MAX
} user_button_t;

static char *enum_btn_id_string[] = {
    ENUM_TO_STR(USER_BUTTON_0),
    ENUM_TO_STR(USER_BUTTON_1),
    ENUM_TO_STR(USER_BUTTON_2),
    ENUM_TO_STR(USER_BUTTON_3),
    ENUM_TO_STR(USER_BUTTON_MAX),
};
```

## 操作符 ‘##’

‘##’ 是预处理拼接标记。在宏定义展开的时候，将 ‘##’ 左边的内容，与 ‘##’ 右边的内容拼接到一起。

注意，对于任何一种形式的宏定义，‘##’ 预处理标记都不应出现在*替换列表*的开头或结尾。

关于*替换列表*：

例如宏定义 ‘#define aa(x, y) (x##y)’ 后面的部分 ‘x##y’ 就是替换列表。

预处理拼接符 `##` 常用于使用宏定义批量生成函数或者变量。

### 示例程序 F

```
#include <stdint.h>
#include <rtthread.h>

#define my_math(x, y) (x##e##y)

int main(void)
{
    rt_kprintf("hello rt-thread\n");

    printf("%e\r\n", my_math(3, 4));

    return 0;
}

```

以上代码是科学计数法的格式输出到控制台，输出内容如下：

```
hello rt-thread
3.000000e+04
msh />
```

### 示例程序 G

用预处理拼接符 `##` 批量生成函数或者变量。

以下代码截取自 [RT-Thread `finsh_api.h`](https://github.com/RT-Thread/rt-thread/blob/master/components/finsh/finsh_api.h)，该段代码用于导出 Finsh 命令，代码如下所示：

```
#define FINSH_FUNCTION_EXPORT_CMD(name, cmd, desc)                      \
    const char __fsym_##cmd##_name[] SECTION(".rodata.name") = #cmd;    \
    const char __fsym_##cmd##_desc[] SECTION(".rodata.name") = #desc;   \
    RT_USED const struct finsh_syscall __fsym_##cmd SECTION("FSymTab")= \
    {                           \
        __fsym_##cmd##_name,    \
        __fsym_##cmd##_desc,    \
        (syscall_func)&name     \
    };
```

该宏定义的应用，如 `list_timer` 命令，如下所示：

```
FINSH_FUNCTION_EXPORT_CMD(list_timer, list_timer, list timer in system);
```

## 标识符 `__VA_ARGS__`

`__VA_ARGS__` 是在 C99 中增加的新特性。虽然 C89 引入了一种标准机制，允许定义具有可变数量参数的函数，但是 C89 中不允许这种定义可变数量参数的方式出现在宏定义中。C99 中加入了 `__VA_ARGS__` 关键字，用于支持在宏定义中定义可变数量参数，用于接收 `...` 传递的多个参数。[<sup>1</sup>](#refer-anchor-1)

`__VA_ARGS__` 只能出现在使用了省略号的像函数一样的宏定义里。例如 `#define myprintf(...) fprintf(stderr, __VA_ARGS__)`。

### 解析不定参

通过宏定义，将多个参数传递给函数，那么函数是如何解析不定参的呢？

这就需要使用标准库头文件 `<stdarg.h>` 中的三个宏，分别是 “va_start()”、“va_arg()”、“va_end()”，以及一个可变参类型 “va_list”，示例使用方式借用 RT-Thread 中的 `rt_sprintf` 的实现，代码如下所示：

```
rt_int32_t rt_sprintf(char *buf, const char *format, ...)
{
    rt_int32_t n;
    va_list arg_ptr;

    va_start(arg_ptr, format);
    n = rt_vsprintf(buf, format, arg_ptr);
    va_end(arg_ptr);

    return n;
}
```

- 首先使用 “va_list” 类型定义一个变量 “arg_ptr”
- 然后调用 “va_start(arg_ptr, format);” 函数，第一个入参是 “va_list” 类型，第二个参数是 “rt_sprintf” 函数参数列表中的最后一个定参 “format”
- 然后，通过调用 “rt_vsprintf” 函数，根据 “format” 来解析不定参，并将结果存放到 “buf” 中
- 最后，使用 “va_end(arg_ptr);” 来释放不定参列表占用的资源

## 带 ‘#’ 的标识符 `#__VA_ARGS__`

预处理标记 ‘#’ 用于将宏定义参数转化为字符串，因此 `#__VA_ARGS__` 会被展开为参数列表对应的字符串。

示例：

```
#define showlist(...) put(#__VA_ARGS__)

测试如下：
showlist(The first, second, and third items.);
showlist(arg1, arg2, arg3);

输出结果分别为：
The first, second, and third items.
arg1, arg2, arg3
```

## 带 ‘##’ 的标识符 `##__VA_ARGS__`

`##__VA_ARGS__` 是 GNU 特性，不是 C99 标准的一部分，C 标准不建议这样使用，但目前已经被大部分编译器支持。

标识符 `##__VA_ARGS__` 的意义来自 ‘##’，主要为了解决一下应用场景：

```
#define myprintf_a(fmt, ...) printf(fmt, __VA_ARGS__)
#define myprintf_b(fmt, ...) printf(fmt, ##__VA_ARGS__)

应用：
myprintf_a("hello");
myprintf_b("hello");

myprintf_a("hello: %s", "world");
myprintf_b("hello: %s", "world");
```

这个时候，编译器会报错，如下所示：

```
applications\main.c: In function 'main':
applications\main.c:26:57: error: expected expression before ')' token
 #define myprintf_a(fmt, ...) printf(fmt, __VA_ARGS__)
                                                         ^
applications\main.c:36:5: note: in expansion of macro 'myprintf_a'
     myprintf_a("hello");
```

为什么呢？

我们展开 `myprintf_a("hello");` 之后为 `printf("hello",)`。因为没有不定参，所以，`__VA_ARGS__` 展开为空白字符，这个时候，printf 函数中就多了一个 ‘,’（逗号），导致编译报错。而 `##__VA_ARGS__` 在展开的时候，因为 ‘##’ 找不到连接对象，会将 ‘##’ 之前的空白字符和 ‘,’（逗号）删除，这个时候 printf 函数就没有了多余的 ‘,’（逗号）。

## 参考

<div id="refer-anchor-1"></div>

- [1] [open-std 网站](http://www.open-std.org/jtc1/sc22/wg14/www/standards.html#9899)

    - [C99 标准 V5.10](http://www.open-std.org/jtc1/sc22/wg14/www/docs/C99RationaleV5.10.pdf)
    - [C99 公共版本 N1256](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf)
    - [C11 公共版本 N1570](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)
