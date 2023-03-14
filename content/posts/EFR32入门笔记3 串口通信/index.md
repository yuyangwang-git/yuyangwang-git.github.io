---
title: "EFR32 入门笔记 3: 串口通信"
date: 2023-03-13T21:31:51+08:00
draft: true

summary: 看明白官方项目结构后也没多难，官方文档写的很清晰
description: 本文参考官方例程 Platform - I/O stream USART Bare-metal

tags: ["C", "EFR32"]
categories: ["embedded"]

cover: 
    image: 'img/cover.jpg'
---

## USART 外设的使用

首先是安装 Service - I/O Stream: USART

![I/O: stream USART](img/IOStreamUSART.png#center)

注意一定不要手贱把名称填写为：`INSTANCES`，不然编译会报错：

```bash
error: conflicting types for 'sl_iostream_usart_init_instances'
```

~~这是因为和库里某个函数重名了，可能官方也没想到还能这么重名~~

然后根据需要调整串口的配置，比如流量控制（RTS & CTS），波特率，引脚啥啥啥的。

![I/O: stream USART](img/IOStreamUSART.png#center)

参考[上一篇文章](https://wangyuyang.me/posts/efr32%E5%85%A5%E9%97%A8%E7%AC%94%E8%AE%B02-%E7%82%B9%E4%BA%AEled%E7%81%AF/)的思路，新建空项目，并创建如下几个文件：

```bash
## 已有
main.c
app.c
app.h

## 新建文件
app_usart_communication.c
app_usart_communication.h
```

继续保持 `main.c` 和 `app.h` 不变，在 `app.c` 中填入：

```c
// app.c

#include "app_usart_communication.h"

void app_init(void)
{
  app_usart_communicationt_init();
}

void app_process_action(void)
{
  app_usart_communication_process_action();
}

```

在 `app_usart_communication.h` 中填入：

```c
// app_usart_communication.h

#ifndef APP_USART_COMMUNICATION_H
#define APP_USART_COMMUNICATION_H

void app_usart_communication_init(void);

void app_usart_communication_action(void);

#endif  // APP_USART_COMMUNICATION_H
```

接下来就是重点了，在 `app.c` 中填入：

```c
void app_usart_communication_init(void)
{

}

void app_usart_communication_process_action(void)
{

}
```

待更。
