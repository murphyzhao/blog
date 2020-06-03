
---
title: 'RT-Thread DHT11 温湿度传感器驱动'
tags:
  - 'RT-Thread'
  - 'DHT11'
categories: 物联网
date: 2019-08-17
index_img: https://gitee.com/zhaojuntao/PictureBed/raw/master/blog/ArticleImage/personal-computer-motherboard-4316.jpg
---

> 分享一个我整理的 [DHT11 温湿度传感器驱动 dht11_rtt 软件包](https://github.com/murphyzhao/dht11_rtt)

[dht11_rtt](https://github.com/murphyzhao/dht11_rtt) 是基于 RT-Thread 物联网操作系统实现的 dht11 驱动软件包，该软件包托管在 GitHub，使用 `Apache-2.0` 协议许可。

[dht11_rtt](https://github.com/murphyzhao/dht11_rtt) 驱动使用了 RT-Thread Sensor 传感器框架和 Pin 驱动框架，因此在使用的过程中需要开启这两个功能。不过，在使用 RT-Thread 的 menuconfig 进行配置的时候会默认选中 Sensor 和 Pin 设备配置。

## 简介

DHT11 数字温湿度传感器是一款含有已校准数字信号输出的温湿度复合传感器。它应用专用的数字模块采集技术和温湿度传感技术，确保产品具有枀高的可靠性与卓越的长期稳定性。传感器包括一个电阻式感湿元件和一个 NTC 测温元件，幵与一个高性能 8 位单片机相连接，对外提供单总线通讯接口。更多 DHT11 的使用说明请参考 DHT11 厂家数据手册。

## 示例

[dht11_rtt](https://github.com/murphyzhao/dht11_rtt) 驱动软件包提供了示例程序，menuconfig 配置如下：

选择一个 BSP 工程，使用 menuconfig 命令打开如下界面：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019081722242710.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIzNDk2Nzk=,size_16,color_FFFFFF,t_70)

由 menuconfig 配置保存后，使用 `pkgs --update` 命令将该软件包从 github 下载到本地 bsp 工程中。

该示例程序使用了 RT-Thread 的组件初始化特性，编译下载后，板卡上电的时候自动完成初始化工作，不需要用户修改任何代码（可能需要修改 DHT11 使用的 GPIO，示例中使用的是 GPIOB_12）。

修改 DHT11 连接的 GPIO 管脚：

在 `dht11_sample.c` 中修改以下代码：

```
#define DHT11_DATA_PIN    GET_PIN(B, 12)
```

程序编译下载到板卡后，会在串口中每 1s 打印一次温湿度数据。

## 注意事项

DHT11 是采用单总线通讯的传感器，本软件包采用 GPIO 模拟单总线时序。DHT11 的一次完整读时序需要 20ms，时间过长，故无法使用关中断或者关调度的方式实现独占 CPU 以保证时序完整正确。因此可能出现读取数据失败的情况，请用户注意。

DHT11 通讯时序：

```
主机发送起始信号 —> DHT11 检测到并发送响应信号
DHT11 发送 40 位数据（HSB） —> DHT11 发送结束信号
```
