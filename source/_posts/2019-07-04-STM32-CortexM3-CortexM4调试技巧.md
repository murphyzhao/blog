
---
title: 'Cortex-M3/4 调试技巧、STM32 调试手段'
tags:
  - 'ARM'
  - 'Cortex M3'
  - 'Cortex M4'
  - 'STM32'
  - 'STM32 调试方法'
categories: 物联网
date: 2019-07-04
index_img: https://gitee.com/zhaojuntao/PictureBed/raw/master/blog/ArticleImage/STAndSTM32.png
---

# Cortex-M3/4 一些调试技巧

今天主要总结下这段时间在没有 **调试器** 情况下，解决 bug 的一些辅助调试手段。

在没有 **调试器** 的情况下，进行代码调试的手段就只有 log 大法，为了能够尽可能详细地输出有用的调试信息，往往我们需要将 **调用栈** 、R0- R15 寄存器、SCB、中断状态、线程状态等信息打印出来，然后配合 **反汇编** 进行调试跟踪代码。这会用到一些特殊的函数（内链汇编函数），下面将介绍我用到的汇编函数，这些都基于 GUN GCC 工具链。

## 获取中断号

获取中断号是通过系统控制块 SCB 寄存器来获取，参考如下：
```
#define SCB_ICSR_VECTACTIVE_Pos             0                                             /*!< SCB ICSR: VECTACTIVE Position */
#define SCB_ICSR_VECTACTIVE_Msk            (0x1FFUL << SCB_ICSR_VECTACTIVE_Pos)           /*!< SCB ICSR: VECTACTIVE Mask */

uint32_t get_current_irq()
{
    return (((SCB->ICSR & SCB_ICSR_VECTACTIVE_Msk) >> SCB_ICSR_VECTACTIVE_Pos) - 16);
}
```

## 查看设置中断优先级

- NVIC_SetPriority(IRQn_Type IRQn) 设置指定中断优先级
- NVIC_GetPriority(IRQn_Type IRQn) 查看指定中断优先级

## 判断当前工作状态（中断状态 or 线程状态）

获取工作状态是在中断状态还是线程状态，利用了 **主栈指针** MSP 和 **线程栈** 指针 PSP，如果在中断状态下，MSP == PSP；相反，如果在线程状态下，MSP != PSP。

```
__attribute__( ( always_inline ) ) static inline uint32_t _get_msp(void) {
    register uint32_t result;
    __asm volatile ("MRS %0, msp\n" : "=r" (result) );
    return(result);
}
__attribute__( ( always_inline ) ) static inline uint32_t _get_psp(void) {
    register uint32_t result;
    __asm volatile ("MRS %0, psp\n" : "=r" (result) );
    return(result);
}

if (_get_msp() == _get_psp()) /* in isr */
{

}
else /* in task */
{

}
```

当然，也可以直接通过检查 CONTROL 寄存器第 1 位来确定当前在线程模式还是处理模式。

## 查看当前特权等级

可以通过检查 CONTROL 和 IPSR 寄存器来确定当前是否处于特权等级。

- CONTROL 寄存器第 0 位为标识线程模式中的特权等级
- IPSR 寄存器存储异常 / 中断编号，非 0 则为处理模式，处理模式属于特权等级

这里使用 CMSIS 接口来进行判断：

```
int is_privileged_mode (void)
{
    if (_get_IPSR() != 0 )
    {
        return 1; /* 特权等级 */
    }
    else if ((_get_CONTROL() & 0x01) == 0)
    {
        return 1; /* 特权等级 */
    }
    else
    {
        return 0; /* 非特权等级 */
    }
}
```

## 查看调用者函数（通过 LR 寄存器）

LR 寄存器保存着被调用函数的返回地址，拿到这个返回地址后，在反汇编文件中，查找该地址，即可得到调用者函数。使用该函数的时候，必须是函数入口处的第一个函数，如果在此函数（__get_LR）之前调用了其他函数，那么得到的 LR 指针就不再是原来调用者函数返回地址了。
**注意：**
在 Cortex-M3/4 平台下，LR 寄存器保存的返回地址通常是**基数**，这是因为 Cortex-M 平台有些跳转 / 调用操作需要将地址的第 0 位置 1 以表示操作模式为 Thumb 状态，因此在使用该地址的时候，请自动将基数地址减一，然后到反汇编文件中查找。

```
__attribute__( ( always_inline ) ) static inline uint32_t __get_LR(void)
{
    register uint32_t result;

    __ASM volatile ("mov %0, lr\n" : "=r" (result) );
    return(result);
}
```

## 反汇编

```
arm-none-eabi-objdump -d xxx.elf > xxx.asm
```

## 导出符号表

```
arm-none-eabi-objdump -t xxx.elf > xxx.sym
```

## 说明

以上代码基于 Cortex-M4 SOC，GNU GCC gcc-arm-none-eabi-4_8-2014q3 工具链版本编译测试。

### 关于 LR 寄存器

链接寄存器（LR）用于函数或子程序调用时 **返回地址** 的保存，当执行了函数或子程序调用后，LR 的数值会自动更新。因此，在执行函数或子程序调用前，需要将 LR 寄存器压入栈中保存，否则 LR 当前值丢失，无法返回最初的函数调用返回地址。
