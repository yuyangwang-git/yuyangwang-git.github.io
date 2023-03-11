---
title: "EFR32 入门笔记 2: 点亮 LED 灯"
date: 2023-03-10T22:28:48+08:00
draft: true

summary: 前面简单讲了开发环境的搭建和官方模板的结构，这里尝试写一个简单的点灯程序
description: 

tags: ["C", "EFR32"]
categories: ["embedded"]

cover: 
    image: 'img/cover.jpg'
---

## LED 闪烁程序 v1.0 —— 不使用定时器

创建一个 `Empty C Project`，其它文件不用动，直接修改 `app.c` 就行：

```c
// app.c

#include "em_cmu.h"
#include "em_gpio.h"

typedef struct
{
    GPIO_Port_TypeDef port;
    uint8_t pin;
} LED;

LED led_0;

// LED 闪烁函数
// 通过循环实现延时
void led_blink(LED *led)
{
    for (int i = 0; i < 10000; i++)
    {
        GPIO_PinOutToggle(led->port, led->pin);
    }
}

void app_init(void)
{
    led_0.port = gpioPortA;
    led_0.pin = 1;

    CMU_ClockEnable(cmuClock_GPIO, true);                            // 开启 GPIO 时钟
    GPIO_PinModeSet(led_0.port, led_0.pin, gpioModePushPull, 0);
}

void app_process_action(void)
{
    led_blink(&led_0);
}
```

没啥好说的，很简单，GPIO 相关 API 可参考[官方文档](https://docs.silabs.com/gecko-platform/4.2/emlib/api/efr32xg22/group-gpio)

## LED 闪烁程序 v2.0 —— 使用定时器

前面的程序采用循环来实现延时功能，不仅难以精确控制时间，阻塞式的代码也会显著降低系统的响应性，下面采用定时器来重写上述程序。

这里不准备直接使用 EFR32 内部硬件 Timer，而选择 Gecko SDK 中提供的抽象程度更高的 Service —— Sleep Timer。

> The Sleeptimer driver provides software timers, delays, timekeeping and date functionalities using a low-frequency real-time clock peripheral.
>
> All Silicon Labs microcontrollers equipped with the RTC or RTCC peripheral are currently supported. Only one instance of this driver can be initialized by the application.
>
> — <cite>Sleep Timer Official Documents[^1]</cite>

[^1]: [Sleep Timer Official Documents](https://docs.silabs.com/gecko-platform/latest/service/api/group-sleeptimer).

在使用前，首先要在 Simplicity Studio 中安装 Sleep Timer Components：

![Install Sleep Timer](img/SleepTimer.png#center)

安装完成后，`Project_DIR\gecko_sdk_4.2.2\platform\service` 文件夹下多了一个文件夹 `sleeptimer`

![Project Explorer](img/ProjectExplorer.png#center)

接下来，修改 `app.c` 中的程序，其它文件保持不变：

```c
//app.c

#include "em_cmu.h"
#include "em_gpio.h"
#include "sl_sleeptimer.h"   // 增加头文件

typedef struct
{
    GPIO_Port_TypeDef port;
    uint8_t pin;
} LED;

LED led_0;
bool timeout_flag;
unsigned int delay_ms;
sl_sleeptimer_timer_handle_t timer;

// Sleep Timer 超时回调函数
void timeout(sl_sleeptimer_timer_handle_t *handle, void *data)
{
    (void)&handle;           // 因为我们的回调根本用不到这两个参数，如果不写这两行
    (void)&data;             // 会提示 warning: unused parameter 'handle' 'data'
    timeout_flag = true;
}

void app_init(void)
{
    led_0.port = gpioPortA;
    led_0.pin = 1;
    delay_ms = 500;

    CMU_ClockEnable(cmuClock_GPIO, true);                            // 开启 GPIO 时钟
    GPIO_PinModeSet(led_0.port, led_0.pin, gpioModePushPull, 0);

    // 启用 sleep timer 定时器
    sl_sleeptimer_start_periodic_timer_ms(&timer, delay_ms, timeout, NULL, 0, 
                                          SL_SLEEPTIMER_NO_HIGH_PRECISION_HF_CLOCKS_REQUIRED_FLAG);
}

void app_process_action(void)
{
    if (timeout_flag == true)
    {
        GPIO_PinOutToggle(led_0.port, led_0.pin);
        timeout_flag = false;
    }
}

```

编译完成后，烧录程序，将看到 LED 灯闪烁。

### 为什么没有手动初始化 Sleep Timer Service，程序就可以正确运行？

笔者在直接调用 `sl_sleeptimer_start_periodic_timer_ms()` 函数时产生了上述疑问，于是带着疑问去翻阅了官方文档。根据官方文档，Sleep Timer 的初始化函数 `sl_sleeptimer_init()` 会在启动代码中被调用。

那么我们回到 `main.c` 文件：

```c
// main.c

......

  // Initialize Silicon Labs device, system, service(s) and protocol stack(s).
  // Note that if the kernel is present, processing task(s) will be created by
  // this call.
  sl_system_init();

......
```

继续查看 `sl_system_init()` 的实现：

```c
// sl_system_init.c

#include "sl_event_handler.h"

void sl_system_init(void)
{
  sl_platform_init();
  sl_driver_init();
  sl_service_init();         // 这里有一行 service 初始化程序
  sl_stack_init();
  sl_internal_app_init();
}
```

继续查看 `sl_service_init()` 的实现：

```c
// sl_event_handler.c

void sl_service_init(void)
{
  sl_sleeptimer_init();      // 多了这一行
}
```

会发现 Simplicity Studio 已经在安装时帮我们增加了 Sleep Timer 的初始化代码，因此无需手动调用初始化函数 `sl_sleeptimer_init()`。

## LED 闪烁程序 v3.0 —— 优化项目结构

在这一节内容中，不打算给程序增加任何新功能，而是模仿 Gecko SDK Programming Model 的风格，重写前述程序。

项目文件夹下新建文件：`blink.c`，`blink.h`：

![Project Explorer](img/ProjectExplorer.png#center)

然后写入以下内容：

```c
// blink.h

#ifndef BLINK_H
#define BLINK_H

void blink_init(void);

void blink_process_action(void);

#endif // BLINK_H
```

```c
// blink.c
#include "em_cmu.h"
#include "em_gpio.h"
#include "sl_sleeptimer.h"

typedef struct
{
    GPIO_Port_TypeDef port;
    uint8_t pin;
} LED;

LED led_0;
bool timeout_flag;
unsigned int delay_ms;
sl_sleeptimer_timer_handle_t timer;

// Sleep Timer 超时回调函数
void timeout(sl_sleeptimer_timer_handle_t *handle, void *data)
{
    (void)&handle;           // 因为我们的回调根本用不到这两个参数，如果不写这两行
    (void)&data;             // 会提示 warning: unused parameter 'handle' 'data'
    timeout_flag = true;
}

void blink_init(void)
{
    led_0.port = gpioPortA;
    led_0.pin = 1;
    delay_ms = 500;

    CMU_ClockEnable(cmuClock_GPIO, true);                            // 开启 GPIO 时钟
    GPIO_PinModeSet(led_0.port, led_0.pin, gpioModePushPull, 0);

    // 启用 sleep timer 定时器
    sl_sleeptimer_start_periodic_timer_ms(&timer, delay_ms, timeout, NULL, 0, 
                                          SL_SLEEPTIMER_NO_HIGH_PRECISION_HF_CLOCKS_REQUIRED_FLAG);
}

void blink_process_action(void)
{
    if (timeout_flag == true)
    {
        GPIO_PinOutToggle(led_0.port, led_0.pin);
        timeout_flag = false;
    }
}
```

接着修改 `app.c` 文件：

```c
// app.c

#include "blink.h"

void app_init(void)
{
    blink_init();
}

void app_process_action(void)
{
    blink_process_action();            // 有一点 RTOS 的感觉了
}
```

简单总结一下，`app.c` 和 `app.h` 是用户程序的顶层入口，考虑到每一个程序都可以拆分为若干子程序，这里将每个（本例仅一个）子程序单独形成文件，有点类似若干不同的“进程”。

除了上述内容外，还可以做进一步的修改，待更。
