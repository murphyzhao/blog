
---
title: 'Ubuntu 上安装 iperf'
tags:
  - 'iperf'
  - 'Ubuntu'
categories: 物联网
date: 2019-11-05
index_img: https://gitee.com/zhaojuntao/PictureBed/raw/master/blog/ArticleImage/2020-06-03-network-test.png
---

# Ubuntu 上安装 iperf

> 最近 NB-IOT 项目需要进行外网测试，那么就先用 iperf 测个速度吧。
>
> 网速不一定能够反映真实情况，跟您使用的外网服务器带宽也有关系。

## 下载

```
git clone https://github.com/esnet/iperf.git
```

## 安装

依次执行以下命令：

```
$ cd iperf
$ ./configure
$ sudo ldconfig /usr/local/lib
$ make
$ make install
```

## 运行

- 运行 server

    `iperf3 -s`

- 运行 client

    `iperf3 -c server_addr -p server_port`

### 后台运行

```
$ sudo nohup iperf3 -s >> iperf_s.log 2>&1 &
```

注意 iperf3 后台运行需要使用 root 权限，否则终端退出后，服务也就结束了。

## 常见问题

### 无法找到 libiperf.soxx

在运行 `iperf3 -s` 的时候，提示以下错误：

```
iperf3: error while loading shared libraries: libiperf.so.0: cannot open shared object file: No such file or directory
```

简单的解决方法，在 iperf Git 仓库根目录执行以下命令：

```
sudo ldconfig /usr/local/lib
```

参考 [github issue](https://github.com/esnet/iperf/issues/153)。

### 不能解析客户端参数

```
iperf3: error - unable to receive parameters from client:
```

该问题是因为使用了不同版本的客户端服务器软件导致的。因为 iperf3 与 iperf2 不兼容。
解决方法是使用相同版本的客户端和服务器。

参考 [github issue](https://github.com/esnet/iperf/issues/153)。
