---
title: 'RT-Thread Pin 设备驱动框架快速对接'
tags:
  - '嵌入式操作系统'
  - 'RT-Thread'
categories: 操作系统
date: 2019-07-10
index_img: https://gitee.com/zhaojuntao/PictureBed/raw/master/blog/ArticleImage/blog.jpg
---

## 为什么需要 Pin 设备驱动框架

- 跨平台可移植应用
- 操作简单

试想下面这个场景：

你基于 STM32 MCU 编写了一个包含很多 GPIO 操作的应用程序，GPIO 控制函数使用的是 HAL 库。
后面，由于某种原因，MCU 需要更换，使用的是 NXP 的芯片，不支持 HAL 库，那你怎么办？

通常，你会查找所有 **GPIO 操作相关的接口**，然后替换成 NXP 提供的 GPIO 驱动函数，如果 GPIO 的编排方式不一样（有的有 GPIOA 这样的分组，有的没有），那就更加的麻烦。这样，你就维护了两个版本的应用程序。

那么，有了 Pin 驱动框架能解决这个问题吗？

当然，Pin 只能解决应用层代码不需要变动的问题，但需要将 GPIO 驱动对接到 Pin 驱动框架上。
另外，RT-Thread Pin 设备驱动框架采用了统一 GPIO 号编排的方式，从 0 开始向上计数，每一个 GPIO 对应一个 pin 号。这样做的好处是，应用程序无需关心该 GPIO 属于哪一个组（GPIOA，还是 GPIOB），也无需关心相关 GPIO 的时钟初始化、中断等。因为，Pin 设备驱动框架都帮你抽象好了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190710144526656.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIzNDk2Nzk=,size_16,color_FFFFFF,t_70)

## 怎么对接到 Pin 设备框架

### 首先了解下文件结构

PIN 设备框架对应文件：

- pin.h
- pin.c

源文件路径： **components/drivers/misc/pin.c**

对接到 PIN 设备框架的驱动代码需要我们自行实现（如果官方没有现成的 BSP 可以使用），为了保持驱动实现的一致性，这里有一些潜在规则：

- RT-Thread 通常将对接设备框架的驱动相关文件存放在对应 bsp 的 **drivers** 目录下
- 驱动程序文件命名采用 **drv_** 前缀加驱动名称的方式

当然，并不是所有的 BSP 对接 rt-thread 设备框架的驱动程序文件都存放在 **drivers** 目录下，如 STM32 新 BSP  中的驱动实现在 rt-thread/bsp/stm32/libraries/HAL_Drivers 目录下。

由此，本次我们需要实现的 GPIO 驱动如下：

GPIO 驱动对应文件：

- drv_gpio.h
- drv_gpio.c

### PIN 驱动框架和 GPIO 驱动对接

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190709223437627.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIzNDk2Nzk=,size_16,color_FFFFFF,t_70)

从上图可以直观地看到 Pin 设备驱动框架和 GPIO 驱动是如果对接到一起的。

如上图所示：

- 左侧为 Pin 设备驱动框架封装的 API 接口，向上提供给应用层使用；
- 右侧为具体芯片平台对接 Pin 设备的驱动程序，与 Pin 设备框架提供的 API 接口一一对应，通过 rt_pin_ops 结构体关联到一起，驱动框架层接口最终会回调驱动层接口；
- 最后通过 rt_device_pin_register 函数接口，将底层驱动与 Pin 设备框架绑定到一起，注册到 RT-Thread 的设备框架中。

在对接 RT-Thread Pin 设备框架的时候，仅需要实现上图右侧 `rt_pin_ops` 结构体中的所有 callback 回调函数即可，然后通过 `rt_device_pin_register` 注册到系统，最后使用 `rt_hw_pin_init` 调用以进行初始化。
`rt_hw_pin_init` 在系统启动过程中被 `rt_hw_board_init` 调用。

**注意：**

需要注意的是 `rt_pin_attach_irq` 接口的使用，其函数原型如下：

```C
rt_err_t rt_pin_attach_irq(rt_int32_t pin, rt_uint32_t mode,
                             void (*hdr)(void *args), void  *args)
```
该函数的功能是，为指定的 `pin` 脚绑定中断回调函数 `hdr`，并设置对应的中断模式 `mode` 。该函数是唯一的从应用层，通过 Pin 设备框架向下注册到 GPIO 驱动层的 API 接口。

用户在实现 `rt_pin_attach_irq` 对应的 `pin_attach_irq` 回调函数时，记得将 `hdr` 回调函数应用到对应 pin 脚的外部中断 Handler 中。

## 如何使用

RT-Thread 官方的文档中心中 Pin 设备给出了一个基本的测试例程，这里做引用，如下所示：

> 以下内容引用自 RT-Thread 文档中心，地址：https://www.rt-thread.org/document/site/programming-manual/device/pin/pin/

PIN 设备的具体使用方式可以参考如下示例代码，示例代码的主要步骤如下：

- 设置蜂鸣器对应引脚为输出模式，并给一个默认的低电平状态
- 设置按键 0 和 按键1 对应引脚为输入模式，然后绑定中断回调函数并使能中断
- 按下按键 0 蜂鸣器开始响，按下按键 1 蜂鸣器停止响

代码如下：

```C
/*
 * 程序清单：这是一个 PIN 设备使用例程
 * 例程导出了 pin_beep_sample 命令到控制终端
 * 命令调用格式：pin_beep_sample
 * 程序功能：通过按键控制蜂鸣器对应引脚的电平状态控制蜂鸣器
*/

#include <rtthread.h>
#include <rtdevice.h>

/* 引脚编号，通过查看设备驱动文件drv_gpio.c确定 */
#ifndef BEEP_PIN_NUM
    #define BEEP_PIN_NUM            35  /* PB0 */
#endif
#ifndef KEY0_PIN_NUM
    #define KEY0_PIN_NUM            55  /* PD8 */
#endif
#ifndef KEY1_PIN_NUM
    #define KEY1_PIN_NUM            56  /* PD9 */
#endif

void beep_on(void *args)
{
    rt_kprintf("turn on beep!\n");

    rt_pin_write(BEEP_PIN_NUM, PIN_HIGH);
}

void beep_off(void *args)
{
    rt_kprintf("turn off beep!\n");

    rt_pin_write(BEEP_PIN_NUM, PIN_LOW);
}

static void pin_beep_sample(void)
{
    /* 蜂鸣器引脚为输出模式 */
    rt_pin_mode(BEEP_PIN_NUM, PIN_MODE_OUTPUT);
    /* 默认低电平 */
    rt_pin_write(BEEP_PIN_NUM, PIN_LOW);

    /* 按键0引脚为输入模式 */
    rt_pin_mode(KEY0_PIN_NUM, PIN_MODE_INPUT_PULLUP);
    /* 绑定中断，下降沿模式，回调函数名为beep_on */
    rt_pin_attach_irq(KEY0_PIN_NUM, PIN_IRQ_MODE_FALLING, beep_on, RT_NULL);
    /* 使能中断 */
    rt_pin_irq_enable(KEY0_PIN_NUM, PIN_IRQ_ENABLE);

    /* 按键1引脚为输入模式 */
    rt_pin_mode(KEY1_PIN_NUM, PIN_MODE_INPUT_PULLUP);
    /* 绑定中断，下降沿模式，回调函数名为beep_off */
    rt_pin_attach_irq(KEY1_PIN_NUM, PIN_IRQ_MODE_FALLING, beep_off, RT_NULL);
    /* 使能中断 */
    rt_pin_irq_enable(KEY1_PIN_NUM, PIN_IRQ_ENABLE);
}
/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(pin_beep_sample, pin beep sample);
```
