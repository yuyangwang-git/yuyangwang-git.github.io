---
title: "EFR32 入门笔记 5: PWM 驱动 LED 灯"
date: 2023-03-18T20:25:55+08:00
draft: true

summary: Silicon Labs 的自动化工具还挺好用的
description: 本文参考官方例程 Platform - Blink PWM Bare-metal

tags: ["C", "EFR32"]
categories: ["embedded"]

cover:
    image: 'img/cover.jpg'
---

## PWM

### 基本使用

安装 Service - Sleep Timer：

![Sleep Timer](img/SleepTimer.png#center)

以及：

![PWM](img/PWM.png#center)

创建 instance 并修改配置：

![PWM Config](img/PWMConfig.png#center)

`PB02` 将为 PWM 的输出引脚。

> CC 为 Capture/Compare
>
> CDTI 应该是指 Dead Time Insertion，不清楚这里 “C” 的含义。

然后修改 `app.c` 文件：

```c
// app.c

#include "sl_pwm.h"
#include "sl_pwm_instances.h"
#include "sl_sleeptimer.h"

uint8_t pwm_lut[] = {
  0, 1, 1, 1, 2, 2, 2, 2, 2, 2,
  2, 3, 3, 3, 3, 3, 4, 4, 4, 4,
  5, 5, 5, 5, 6, 6, 6, 7, 7, 7,
  8, 8, 8, 9, 9, 10, 10, 10, 11, 11,
  12, 12, 13, 13, 14, 15, 15, 16, 17, 17,
  18, 19, 19, 20, 21, 22, 23, 23, 24, 25,
  26, 27, 28, 29, 30, 31, 32, 34, 35, 36,
  37, 39, 40, 41, 43, 44, 46, 48, 49, 51,
  53, 54, 56, 58, 60, 62, 64, 66, 68, 71,
  73, 75, 78, 80, 83, 85, 88, 91, 94, 97,
  100,
};

void app_init(void)
{
    // Enable PWM output
    sl_pwm_start(&sl_pwm_led);
}

void app_process_action(void)
{
  for (uint8_t i = 0; i < 100; i++) {
    sl_pwm_set_duty_cycle(&sl_pwm_led, pwm_lut[i]);
    sl_sleeptimer_delay_millisecond(6);
    if (i == 0) {
      sl_sleeptimer_delay_millisecond(190);
    }
  }
  for (uint8_t i = 100; i > 0; i--) {
    sl_pwm_set_duty_cycle(&sl_pwm_led, pwm_lut[i]);
    sl_sleeptimer_delay_millisecond(6);
    if (i == 100) {
      sl_sleeptimer_delay_millisecond(190);
    }
  }
}
```

编译，烧录，结束。

### Lut 数组是怎么来的？

有点好奇官方例程里的这个 Lut 数组是用什么函数生成的：

```c
uint8_t pwm_lut[] = {
  0, 1, 1, 1, 2, 2, 2, 2, 2, 2,
  2, 3, 3, 3, 3, 3, 4, 4, 4, 4,
  5, 5, 5, 5, 6, 6, 6, 7, 7, 7,
  8, 8, 8, 9, 9, 10, 10, 10, 11, 11,
  12, 12, 13, 13, 14, 15, 15, 16, 17, 17,
  18, 19, 19, 20, 21, 22, 23, 23, 24, 25,
  26, 27, 28, 29, 30, 31, 32, 34, 35, 36,
  37, 39, 40, 41, 43, 44, 46, 48, 49, 51,
  53, 54, 56, 58, 60, 62, 64, 66, 68, 71,
  73, 75, 78, 80, 83, 85, 88, 91, 94, 97,
  100,
};
```

拖到 Origin 里 plot 一下：

![PWM Lut](img/Lut.svg#center)

看起来有点像指数函数？估计比起线性函数，视觉效果会稍微好一些。
