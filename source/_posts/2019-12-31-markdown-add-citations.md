---
title: Markdown 添加文献引用
tags:
  - Markdown
  - 文献引用
categories: Markdown
abbrlink: 49cc9dcc
date: 2019-12-31 18:50:00
---

## 增加引用的文献列表

示例如下：

```
## 参考

- [1] [百度学术](http://xueshu.baidu.com/)
- [2] [Wikipedia](https://en.wikipedia.org/wiki/Main_Page)
```

## 增加锚点

```
## 参考

<div id="refer-anchor-1"></div>
- [1] [百度学术](http://xueshu.baidu.com/)
<div id="refer-anchor-2"></div>
- [2] [Wikipedia](https://en.wikipedia.org/wiki/Main_Page)
```

## 增加引用

使用 `<sup>` 标注引用，如下所示：

```
## Markdown 增加文献引用

这边文章是介绍如何在 Markdown 中增加文献引用。[<sup>1</sup>](#refer-anchor-1)

## 参考

<div id="refer-anchor-1"></div>

- [1] [百度学术](http://xueshu.baidu.com/)

<div id="refer-anchor-2"></div>

- [2] [Wikipedia](https://en.wikipedia.org/wiki/Main_Page)
```

## 演示

### Markdown 增加文献引用

这边文章是介绍如何在 Markdown 中增加文献引用。[<sup>1</sup>](#refer-anchor-1)

### 参考

<div id="refer-anchor-1"></div>

- [1] [百度学术](http://xueshu.baidu.com/)

<div id="refer-anchor-2"></div>

- [2] [Wikipedia](https://en.wikipedia.org/wiki/Main_Page)
