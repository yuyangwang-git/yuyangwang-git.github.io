---
title: "EFR32 入门笔记 3: 串口通信"
date: 2023-03-13T21:31:51+08:00
draft: false

summary: 看明白官方项目结构后也没多难，官方文档写的很清晰
description: 本文参考官方例程 Platform - I/O stream USART Bare-metal

tags: ["C", "EFR32"]
categories: ["embedded"]

cover: 
    image: 'img/cover.jpg'
---

## USART 外设

### 基本使用

首先是安装 Service - I/O Stream: USART

![I/O: stream USART](img/IOStreamUSART.jpg#center)

注意一定不要手贱把名称填写为：`INSTANCES`，不然编译会报错：

```plain
error: conflicting types for 'sl_iostream_usart_init_instances'
```

~~这是因为和库里某个函数重名了，可能官方也没想到还能这么重名，精准踩坑✓~~

然后根据需要调整串口的配置，比如流量控制（RTS & CTS），波特率，引脚啥啥啥的。

![I/O: stream USART](img/IOStreamConfig.jpg#center)

参考 [上一篇文章](https://wangyuyang.me/posts/efr32%E5%85%A5%E9%97%A8%E7%AC%94%E8%AE%B02-%E7%82%B9%E4%BA%AEled%E7%81%AF/) 的思路，新建空项目，并创建如下几个文件：

```bash
## 已有文件
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

前面都是无关紧要的封装，接下来几行才是重点，在 `app_usart_communication.c` 中填入：

```c
// app_usart_communication.c

#include <string.h>
#include "sl_iostream.h"
#include "sl_iostream_handles.h"
#include "sl_iostream_init_instances.h"

void app_usart_communication_init(void)
{
    const char str1[] = "Hello World!\r\n";
    const char str2[] = "This is EFR32BG22.\r\n";

    // 串口发送数据的第一种形式
    sl_iostream_write(sl_iostream_inst_handle, str1, strlen(str1));

    // 第二种形式，指定默认 stream 后，使用 SL_IOSTREAM_STDOUT 代指 default stream
    sl_iostream_set_default(sl_iostream_inst_handle);
    sl_iostream_write(SL_IOSTREAM_STDOUT, str2, strlen(str2));
}

void app_usart_communication_process_action(void)
{
    // this function is left blank intentionally
}
```

这样就可以使用串口完成数据发送了。

### sl_iostream_inst_handle 是怎么来的？

这里对于 `sl_iostream_inst_handle` 的来历可能有点迷，翻了一下程序结构，该变量是通过头文件 `sl_iostream_handles.h` 引入的，该头文件又引入了`sl_iostream_init_usart_instances.h`

``` c
// sl_iostream_handles.h

#include "sl_iostream_init_usart_instances.h"
```

继续向上查看：

```c
// sl_iostream_init_usart_instances.h

extern sl_iostream_t *sl_iostream_inst_handle;   // line 16
```

如果我们修改 instance name（我们前面用的名称为 `inst`），可以看到该变量名也会发生变化，即始终为 `sl_iostream_InstanceName_handle` 。

实际使用时不用管这么多，只要保证传入的 handle 名称正确即可。

~~早几年刚接触编程的时候觉得 handle 这个概念怎么这么抽象，现在觉得这个比喻还挺巧妙的~~

### 默认 handle 是怎么实现的？

查看 `sl_iostream_write()` 函数的实现：

```c
// sl_iostream.c

sl_status_t  sl_iostream_write(sl_iostream_t *stream,
                               const void *buffer,
                               size_t buffer_length)
{
  // 注意这里
  if (stream == SL_IOSTREAM_STDOUT) {
    stream = sl_iostream_get_default();
  }

  if ((stream != NULL) && (stream->write != NULL)) {
    return stream->write(stream->context, buffer, buffer_length);
  } else {
    return SL_STATUS_INVALID_CONFIGURATION;
  }
}
```

其实就是函数帮我们查阅了默认值到底是个啥。

### 数据是怎样发送的，中断 or 阻塞？

`sl_iostream_write()` 实际上是使用阻塞式的方法发送数据，查看其调用栈就可以很容易的验证这一点：

```c
// em_usart.c

void USART_Tx(USART_TypeDef *usart, uint8_t data)
{
  /* Check that transmit buffer is empty */
  while (!(usart->STATUS & USART_STATUS_TXBL)) {
  }
  usart->TXDATA = (uint32_t)data;
}
```

可以看出数据是通过 while 循环发送的。

### 能否使用 printf() 函数发送数据？

如果觉得 `sl_iostream_write()` 参数太多，用起来有点麻烦，想简化调用，可以使用 `printf()` 函数来发送数据。但还需要我们再额外做一些准备工作，具体可以参考 [这篇文章](https://community.silabs.com/s/article/Understanding-the-IO-Stream-module)，简单的说，就是我们要把 `printf()` 的输出重定向到串口上。

为此，只需安装 Serviece - I/O Stream: Retarget STDIO

![I/O: stream: Retarget STDIO](img/IOStreamRetargetSTDIO.jpg#center)

安装完成就可以直接使用 `printf()` 函数了：

```c
#include <stdio.h>
#include <string.h>
#include "sl_iostream.h"
#include "sl_iostream_handles.h"
#include "sl_iostream_init_instances.h"

void app_usart_communication_init(void)
{
    const char str1[] = "Hello World!\r\n";
    const char str2[] = "This is EFR32BG22.\r\n";

    // 发送数据的第一种形式
    sl_iostream_write(sl_iostream_inst_handle, str1, strlen(str1));

    // 第二种形式，指定默认 stream 后，使用 SL_IOSTREAM_STDOUT 代指 default stream
    sl_iostream_set_default(sl_iostream_inst_handle);
    sl_iostream_write(SL_IOSTREAM_STDOUT, str2, strlen(str2));

    // 第三种形式，使用 printf()
    printf("Printf uses the default stream, as long as iostream_retarget_stdio is included.\r\n");
}

void app_usart_communication_process_action(void)
{
    // this function is left blank intentionally
}
```

这里笔者很好奇为啥安装了这个服务后就能调用 `printf()`，按惯例，查看调用栈，发现项目中多了一个文件 `sl_iostream_retarget_stdio.c`：

```c
// sl_iostream_retarget_stdio.c

int _write(int file, const char *ptr, int len)
{
  sl_status_t status;

  (void)file;
  status = sl_iostream_write(SL_IOSTREAM_STDOUT, ptr, (size_t)len);
  EFM_ASSERT(status == SL_STATUS_OK);

  return len;
}
```

所以说 `printf()` 的底层就是 `sl_iostream_write()` 函数，只不过指定了默认输出流而已。
