
---
title: 'TCP Client 客户端示例程序'
tags:
  - 'TCP Client'
  - 'TCP 客户端'
  - 'LWIP'
  - 'RT-Thread'
categories: 物联网
date: 2019-06-20
index_img: https://gitee.com/zhaojuntao/PictureBed/raw/master/blog/ArticleImage/2020-06-03-network.png
---

# TCP client 示例程序

> 最近总有人问我要 TCP 的客户端代码，就拿手上用来测试的代码分享出来吧。

**关键词：**

```
TCP 客户端代码
TCP client 代码
LWIP TCP 客户端程序
LWIP TCP client 代码
```

每一次调试网络相关的代码都需要一段最简单的 TCP 测试程序，后来就把这个程序记录到了代码片段，今天在自己的博客里再次记录下，希望更方便查找。

## 简单的 TCP 客户端程序源码

本代码在 RT-Thread OS 下测试使用，并验证通过。

- 如果定义了 **RT_USING_SAL** 宏，那么使用 RT-Thread SAL 组件提供的 socket 封装，需要引用 SAL 相关头文件

    ```
    #include <sys/socket.h>
    #include <netdb.h>
    ```

- 如果没有定义 **RT_USING_SAL** 宏，直接使用 lwip socket，包含 lwip 相关头文件

    ```
    #include "lwip/sockets.h"
    #include "lwip/netdb.h"
    #include "lwip/sys.h"
    ```

---

源码如下所示：

```
#include <stdio.h>
#include <string.h>
#include <rtthread.h>

#ifdef RT_USING_SAL
#include <sys/socket.h>
#include <netdb.h>
#else
#include "lwip/sockets.h"
#include "lwip/netdb.h"
#include "lwip/sys.h"
#endif

#define SAL_TCP_TEST_HOST    "your host or ip addr"  /* 输入你的 TCP server 域名或者 ip 地址 */
#define SAL_TCP_TEST_PORT    (80u)                   /* 输入你的 TCP Server 断口号 */
#define SAL_TEST_BUFSZ       (1024u)

static const char *send_tcp_req_data = "Hi, I am from tcp client.";

static void _sal_tcp_test(void)
{
    int ret, i;
    char *recv_data;
    struct hostent *host;
    int sock = -1, bytes_received;
    struct sockaddr_in server_addr;

    /* 通过函数入口参数url获得host地址（如果是域名，会做域名解析） */
    host = gethostbyname(SAL_TCP_TEST_HOST);
    if (!host)
    {
        rt_kprintf("gethostbyname failed!\r\n");
        return;
    }
    else
    {
        rt_kprintf("gethostbyname success!\r\n");
    }

    recv_data = rt_calloc(1, SAL_TEST_BUFSZ);
    if (recv_data == RT_NULL)
    {
        rt_kprintf("No memory\n");
        return;
    }

    /* 创建一个socket，类型是SOCKET_STREAM，TCP 协议, TLS 类型 */
    if ((sock = socket(AF_INET, SOCK_STREAM, 0)) < 0)
    {
        rt_kprintf("Socket error\n");
        goto __exit;
    }
    else
    {
        rt_kprintf("Socket pass\n");
    }
    
    /* 初始化预连接的服务端地址 */
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SAL_TCP_TEST_PORT);
    server_addr.sin_addr = *((struct in_addr *)host->h_addr);
    rt_memset(&(server_addr.sin_zero), 0, sizeof(server_addr.sin_zero));

    rt_kprintf("will connect...\n");
    if (connect(sock, (struct sockaddr *)&server_addr, sizeof(struct sockaddr)) < 0)
    {
        rt_kprintf("Connect fail!\n");
        goto __exit;
    }
    else
    {
        rt_kprintf("Connect pass!\n");
    }
    /* 发送数据到 socket 连接 */
    ret = send(sock, send_tcp_req_data, strlen(send_tcp_req_data), 0);
    if (ret <= 0)
    {
        rt_kprintf("send error,close the socket.\n");
        goto __exit;
    }
    else
    {
        rt_kprintf("send pass!\n");
    }
    
    /* 接收并打印响应的数据，使用加密数据传输 */
    bytes_received = recv(sock, recv_data, SAL_TEST_BUFSZ  - 1, 0);
    if (bytes_received <= 0)
    {
        rt_kprintf("received error,close the socket.\n");
        goto __exit;
    }

    rt_kprintf("recv data:\n");
    for (i = 0; i < bytes_received; i++)
    {
        rt_kprintf("%c", recv_data[i]);
    }

__exit:
    if (recv_data)
        rt_free(recv_data);

    if (sock >= 0)
        closesocket(sock);
}

static void sal_tcp_test(void)
{
    rt_thread_t tid;
    tid = rt_thread_create("sal_tcp", _sal_tcp_test, RT_NULL, 4096, 23, 5);
    if (tid)
        rt_thread_startup(tid);
    else
    {
        rt_kprintf("sal_tcp thread create failed!\r\n");
    }
}

#ifdef FINSH_USING_MSH
#include <finsh.h>
MSH_CMD_EXPORT(sal_tcp_test, SAL TCP function test);
#endif /* FINSH_USING_MSH */
```

## 运行

在 RT-Thread MSH 中可以直接使用 Finsh 命令运行：

```
msh /> sal_tcp_test
```

如果不使用 RT-Thread，只需要将 RT-Thread 特性移除即可。
