
---
title: '灵活的按键处理程序 FlexibleButton'
tags:
  - 'RT-Thread'
  - 'FlexibleButton'
  - '按键处理'
categories: 物联网
date: 2018-12-14
index_img: https://gitee.com/zhaojuntao/PictureBed/raw/master/blog/ArticleImage/blog.jpg
---

> 灵活的按键处理程序 FlexibleButton，C程序编写，无缝兼容任意的处理器，支持任意 OS 和裸机编程。

## 前言

正好工作中用到按键处理，需要处理单击、长按等按键事件，然后就造了这么一个轮子，为了以后更方便地加入其它的项目中使用，遂将其开源到 GitHub 中。

后面发现 RT-Thread 软件包里也有一个开源的按键库 [MultiButton](https://github.com/liu2guang/MultiButton)，看到这个按键库的时候，心想，完了，又重复造轮子了，好伤心 :joy:。想想，既然按键处理方式不同，而且时间已经花出去了，那就把轮子圆一圆，放到 GitHub 中，给有缘人用吧，然后就有了这个不太圆的轮子 [FlexibleButton](https://github.com/murphyzhao/FlexibleButton)。

## 概述

FlexibleButton 是一个基于 C 语言的小巧灵活的按键处理库。该按键库解耦了具体的按键硬件结构，理论上支持轻触按键与自锁按键，并可以无限扩展按键数量。另外，FlexibleButton 使用扫描的方式一次性读取所有所有的按键状态，然后通过事件回调机制上报按键事件。

该按键库使用 C 语言编写，驱动与应用程序解耦，便于灵活应用，比如用户可以方便地在应用层增加按键中断、处理按键功耗、定义按键事件处理方式，而无需修改 FlexibleButton 库中的代码。

另外，使用 C 语言标准库 API 编写，也使得该按键库可以无缝兼容任意的处理器平台，并且支持任意 OS 和 non-OS（裸机程序）。

另外，该按键库核心的按键扫描代码仅有三行，没错，就是经典的**三行按键扫描算法**，出自哪位大神之手就无从得知了，也欢迎知道此高人的读者文后留言哈。算法介绍可以去搜索引擎里查找，这里就不作介绍了。

## 获取

```SHELL
git clone https://github.com/murphyzhao/FlexibleButton.git
```

## 测试

FlexibleButton 库中提供了一个 DEMO `flexible_button_demo.c`，这里基于 RT-Thread OS 进行测试，当然你可以选择使用其他的 OS，或者使用裸机测试，只需要移除 OS 相关的特性即可。

将 FlexibleButton 库放到 RT-Thread BSP 下即可使用 scons 进行编译构建。

## DEMO 程序说明

### 程序入口

```C
int flex_button_main(void)
{
    rt_thread_t tid = RT_NULL;
    user_button_init();
    /* Create background ticks thread */
    tid = rt_thread_create("flex_btn", button_scan, RT_NULL, 1024, 10, 10);
    if(tid != RT_NULL)
    {
        rt_thread_startup(tid);
    }
    return 0;
}
INIT_APP_EXPORT(flex_button_main);
```

如上所示，首先使用 `user_button_init();` 初始化用户按键硬件，并挂载到 FlexibleButton库。然后，使用了 RT-Thread 的 `INIT_APP_EXPORT` 接口导出为上电自动初始化，创建了一个 “flex_btn” 名字的按键扫描线程，线程里扫描检查按键事件。

### 用户初始化代码

 `user_button_init();` 初始化代码如下所示：

```C
static void user_button_init(void)
{
    int i;
    rt_memset(&user_button[0], 0x0, sizeof(user_button));
    user_button[USER_BUTTON_0].usr_button_read = button_key0_read;
    user_button[USER_BUTTON_0].cb = (flex_button_response_callback)btn_0_cb;
    user_button[USER_BUTTON_1].usr_button_read = button_key1_read;
    user_button[USER_BUTTON_1].cb = (flex_button_response_callback)btn_1_cb;
    user_button[USER_BUTTON_2].usr_button_read = button_key2_read;
    user_button[USER_BUTTON_3].usr_button_read = button_keywkup_read;
    rt_pin_mode(PIN_KEY0, PIN_MODE_INPUT); /* set KEY pin mode to input */
    rt_pin_mode(PIN_KEY1, PIN_MODE_INPUT); /* set KEY pin mode to input */
    rt_pin_mode(PIN_KEY2, PIN_MODE_INPUT); /* set KEY pin mode to input */
    rt_pin_mode(PIN_WK_UP, PIN_MODE_INPUT); /* set KEY pin mode to input */
    for (i = 0; i < USER_BUTTON_MAX; i ++)
    {
        user_button[i].status = 0;
        user_button[i].pressed_logic_level = 0;
        user_button[i].click_start_tick = 20;
        user_button[i].short_press_start_tick = 100;
        user_button[i].long_press_start_tick = 200;
        user_button[i].long_hold_start_tick = 300;
        if (i == USER_BUTTON_3)
        {
            user_button[USER_BUTTON_3].pressed_logic_level = 1;
        }
        flex_button_register(&user_button[i]);
    }
}
```

 `user_button_init();` 主要用于初始化按键硬件，设置按键长按和短按的 tick 数（RT-Thread上面默认一个 tick 为 1ms），然后注册按键到 FlexibleButton 库。

### 事件处理代码

```C
static void btn_0_cb(flex_button_t *btn)
{
    rt_kprintf("btn_0_cb\n");
    switch (btn->event)
    {
        case FLEX_BTN_PRESS_DOWN:
            rt_kprintf("btn_0_cb [FLEX_BTN_PRESS_DOWN]\n");
            break;
        case FLEX_BTN_PRESS_CLICK:
            rt_kprintf("btn_0_cb [FLEX_BTN_PRESS_CLICK]\n");
            break;
        case FLEX_BTN_PRESS_DOUBLE_CLICK:
            rt_kprintf("btn_0_cb [FLEX_BTN_PRESS_DOUBLE_CLICK]\n");
            break;
        case FLEX_BTN_PRESS_SHORT_START:
            rt_kprintf("btn_0_cb [FLEX_BTN_PRESS_SHORT_START]\n");
            break;
        case FLEX_BTN_PRESS_SHORT_UP:
            rt_kprintf("btn_0_cb [FLEX_BTN_PRESS_SHORT_UP]\n");
            break;
        case FLEX_BTN_PRESS_LONG_START:
            rt_kprintf("btn_0_cb [FLEX_BTN_PRESS_LONG_START]\n");
            break;
        case FLEX_BTN_PRESS_LONG_UP:
            rt_kprintf("btn_0_cb [FLEX_BTN_PRESS_LONG_UP]\n");
            break;
        case FLEX_BTN_PRESS_LONG_HOLD:
            rt_kprintf("btn_0_cb [FLEX_BTN_PRESS_LONG_HOLD]\n");
            break;
        case FLEX_BTN_PRESS_LONG_HOLD_UP:
            rt_kprintf("btn_0_cb [FLEX_BTN_PRESS_LONG_HOLD_UP]\n");
            break;
    }
}
```

## FlexibleButton 代码说明

### 按键事件定义

按键事件的定义并没有使用 Windows 驱动上的定义，主要是方便嵌入式设备中的应用场景（也可能是我理解的偏差），按键事件定义如下：

```C
typedef enum
{
    FLEX_BTN_PRESS_DOWN = 0,        // 按下事件
    FLEX_BTN_PRESS_CLICK,           // 单击事件
    FLEX_BTN_PRESS_DOUBLE_CLICK,    // 双击事件
    FLEX_BTN_PRESS_SHORT_START,     // 短按开始事件
    FLEX_BTN_PRESS_SHORT_UP,        // 短按抬起事件
    FLEX_BTN_PRESS_LONG_START,      // 长按开始事件
    FLEX_BTN_PRESS_LONG_UP,         // 长按抬起事件
    FLEX_BTN_PRESS_LONG_HOLD,       // 长按保持事件
    FLEX_BTN_PRESS_LONG_HOLD_UP,    // 长按保持的抬起事件
    FLEX_BTN_PRESS_MAX,
    FLEX_BTN_PRESS_NONE,
} flex_button_event_t;
```

### 按键注册接口

使用该接口注册一个用户按键，入参为一个 flex_button_t 结构体实例的地址。

```C
int8_t flex_button_register(flex_button_t *button);
```

### 按键事件读取接口

使用该接口获取指定按键的事件。

```C
flex_button_event_t flex_button_event_read(flex_button_t* button);
````

### 按键扫描接口

按键扫描的核心函数，需要放到应用程序中定时扫描间隔 5-20ms 即可。

```C
void flex_button_scan(void);
```

## 问题和建议

如果您在应用的时候遇到了问题，或者有好的想法和建议，欢迎到这个 [issue](https://github.com/murphyzhao/FlexibleButton/issues/1) 上讨论，谢谢。
