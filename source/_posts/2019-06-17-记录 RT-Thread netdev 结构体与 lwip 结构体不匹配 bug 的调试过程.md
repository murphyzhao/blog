
---
title: '记录 RT-Thread netdev 结构体与 lwip 结构体不匹配 bug 的调试过程'
tags:
  - 'RT-Thread netdev'
  - 'lwip'
categories: 物联网
date: 2019-06-28
index_img: https://gitee.com/zhaojuntao/PictureBed/raw/master/blog/ArticleImage/2020-06-03-network2.png
---

本文主要记录在使用 RT-Thread Netdev 组件的时候遇到的一个结构体不匹配的 bug。

## 背景

本次 bug 只要涉及 4 个文件：

- netdev.h：定义了 `struct netdev` 数据结构
- netdev.c：netdev 源码实现，这里主要涉及 **netdev_low_level_set_link_status** 接口
- netif.c：lwip 网卡相关接口，这里主要涉及 **netif_set_link_up** 接口
- ethernetif.c：lwip 网卡初始化部分代码。这里注册 netdev 设备，涉及 **netdev_add** 接口

**netdev 数据结构如下：**

```
struct netdev
{
    rt_slist_t list; 
    
    char name[RT_NAME_MAX];                            /* network interface device name */
    ip_addr_t ip_addr;                                 /* IP address */
    ip_addr_t netmask;                                 /* subnet mask */
    ip_addr_t gw;                                      /* gateway */
    ip_addr_t dns_servers[NETDEV_DNS_SERVERS_NUM];     /* DNS server */
    uint8_t hwaddr_len;                                /* hardware address length */
    uint8_t hwaddr[NETDEV_HWADDR_MAX_LEN];             /* hardware address */
    
    uint16_t flags;                                    /* network interface device status flag */
    uint16_t mtu;                                      /* maximum transfer unit (in bytes) */
    const struct netdev_ops *ops;                      /* network interface device operations */
    
    netdev_callback_fn status_callback;                /* network interface device flags change callback */
    netdev_callback_fn addr_callback;                  /* network interface device address information change callback */

#ifdef RT_USING_SAL
    void *sal_user_data;                               /* user-specific data for SAL */
#endif /* RT_USING_SAL */
    void *user_data;                                   /* user-specific data */
};
```

另外，运行程序的设备没有调试接口，只有 UART！**只能通过串口打日志！**

## bug 现象

ethernetif.c 中中调用 `static int netdev_add(struct netif *lwip_netif)` 注册一个 netdev 设备（malloc 方式创建一个设备），并基于 lwip netif 初始化 netdev 数据结构。

此时，netdev 数据结构中的各成员初值配置无误，一切正常。

当 lwip 网卡初始化完成，设置 linkup 状态，调用函数 `void netif_set_link_up(struct netif *netif)`。**netif_set_link_up** 函数再调用 `void netdev_low_level_set_link_status(struct netdev *netdev, rt_bool_t is_up)` 接口，设置 netdev 设备 linkup 状态。

在 **netdev_low_level_set_link_status** 函数中操作 netdev 数据结构成员，然后就**出现了问题**！

**问题是 netdev 数据结构成员的值与初始化的值不一样！**

## 分析

一般这种问题，我们会理所当然地认为 **netdev** 数据结构所在的内存块被 xxx 写穿了。

然后就有了下面的一通操作：

1. 对该数据结构所有可能被写穿的地方进行了排查，无解！
2. 加 log，检查 netdev 数据结构从哪里开始出现问题。这个时候发现有的 log 打印输出是对的（ethernetif.c 中是对的），有的地方是错的（netdev.c 中是错的）！
3. 猜想：代码优化所致？
4. 编译器优化？加 `volatile`，不行！
5. 设置局部优化，依旧不行！

    ```
    #pragma GCC push_options
    #pragma GCC optimize ("O0")
    /* TODO Your code */
    #pragma GCC pop_options
    ```

6. dump **netdev** 数据结构所在内存（乍一看，内存中的数据好像是正确的）
7. 回顾：内存数据正确，编译器优化问题也排除了，内存写穿的可能性很小！**奇了怪了**
8. 经同事提醒，可能引入了另外一个 netdev 数据结构！思索了下，不太可能存在数据成员一致的另外一个 netdev 结构体。
9. 既然没办法，就验证下正常 log 输出的地方和异常 log 输出的地方的数据结构的大小是否一致，数据成员的地址是否一致即可
10. 验证结果惊喜！两边数据结构大小不一致！**Why？** 
11. 再次检查 netdev 数据结构，并 dump netdev 数据结构内存，发现了猫腻（结构体最后一个成员的位置不对，没有在内存的最后一块地址）。那么说明结构体确实不匹配
12.  检查发现，netdev 结构中使用的 **ip_addr_t** 存在问题

在使用 ipv4 的时候，是 ip4_addr_t；在使用 ipv6 的时候，是 ip6_addr_t；
检查我的配置，发现 lwipopts.h 同时开启了 IPV4 和 IPV6，而 RT-Thread 的 Netdev 只支持 IPV4。

So，那么问题明朗了！总结见下。

13. 总结

在 ethernetif.c 中创建的 netdev 设备使用的 ip_addr_t 为：

```
typedef struct _ip_addr {
  union {
    ip6_addr_t ip6;
    ip4_addr_t ip4;
  } u_addr;
  u8_t type;
} ip_addr_t;
```

在 netdev.c 中，使用的 ip_addr_t 为：

```
typedef struct ip4_addr
{
    uint32_t addr;
} ip4_addr_t;

typedef ip4_addr_t ip_addr_t;
```

因此，ethernetif.c 和 netdev.c 操作的数据结构不一致，所以在 ethernetif.c 中打印输出的数据成员是正确的（因为在 ethernetif.c 中创建的 netdev 设备对象）。
解决方案，升级 RT-Thread Netdev 组件，以支持 IPV6 和 IPV4 共用的情况。

 ## 总结

这个问题确实很坑！以我现在的经验，在没有调试器的情况下，定位这类问题相对困难。因此，通过本文记录下分析这类问题可能的手段。（中间我还反汇编看了下汇编代码，然并卵，没有用）
本文的 bug 分析过程直接耦合我的应用场景，如果你遇到上述的 BUG 现象，那么能记得检查结构体是否存在不一致的情况。很有可能您就避免了整个上述的分析过程（都是泪 :cry: ）。
