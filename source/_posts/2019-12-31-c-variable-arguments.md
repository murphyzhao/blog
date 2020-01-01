---
title: 'C 语言 #、##、__VA_ARGS__、#__VA_ARGS__、##__VA_ARGS__'
tags:
  - C语言
  - __VA_ARGS__
categories: Programing
abbrlink: '40137057'
date: 2019-12-31 18:46:00
---

## 操作符 ‘#’ 和 ‘##’

‘#’ 和 ‘##’ 属于预处理标记。‘#’ 和 ‘##’ 用于类似函数的宏定义中（或者简称为宏定义函数）。

### 操作符 ‘#’

在宏定义展开的时候，标记 ‘#’ 用于将 ‘#’ 后面的宏定义函数中的参数转化为对应的字符串。

在宏定义展开的时候，宏定义函数的参数与预处理标记 ‘#’ 之间出现的每一个空格都将成为字符串文本中的一个空格字符。并删除第一个预处理标记之前和最后一个预处理标记之后的空白字符。

其中，空参数（NULL）转化为为空白字符。

### 操作符 ‘##’

‘##’ 是预处理拼接标记。在宏定义展开的时候，将 ‘##’ 左边的内容，与 ‘##’ 右边的内容拼接到一起。

注意，对于任何一种形式的宏定义，‘##’ 预处理标记都不应出现在*替换列表*的开头或结尾。

关于*替换列表*：

例如宏定义 ‘#define aa(x, y) (x+y)’ 后面的部分 ‘x+y’ 就是替换列表。

### 示例

```
#define hash_hash # ## #
#define mkstr(a) # a
#define in_between(a) mkstr(a)
#define join(c, d) in_between(c hash_hash d)
char p[] = join(x, y); // equivalent to
                       // char p[] = "x ## y";
```

按顺序展开，如下所示：

```
join(x, y)
in_between(x hash_hash y)
in_between(x ## y)
mkstr(x ## y)
"x ## y"
```

如上所示 `hash_hash` 并不是一个 `##` 连接操作。

## 标识符 `__VA_ARGS__`

`__VA_ARGS__` 是在 C99 中增加的新特性。虽然 C89 引入了一种标准机制，允许定义具有可变数量参数的函数，但是 C89 中不允许这种定义可变数量参数的方式出现在宏定义中。C99 中加入了 `__VA_ARGS__` 关键字，用于支持在宏定义中定义可变数量参数。[<sup>1</sup>](#refer-anchor-1)

`__VA_ARGS__` 只能出现在使用了省略号的像函数一样的宏定义里。例如 `#define myprintf(...) fprintf(stderr, __VA_ARGS__)`。

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
