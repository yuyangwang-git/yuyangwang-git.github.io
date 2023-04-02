---
title: "EFR32 入门笔记 7: BLE 点亮呼吸灯"
date: 2023-04-02T20:05:11+08:00
draft: false

summary: 阻塞主循环不是一个好习惯
description: 

tags: ["C", "EFR32"]
categories: ["embedded"]

cover:
    image: 'img/cover.jpg'
---

## 通过 BLE 控制呼吸灯

前面的文章已经讲过通过 BLE 远程控制 LED 灯的方法，也讲过 PWM 驱动呼吸灯的方法：

```c
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
```

上面这段 PWM 驱动呼吸灯的程序使用了 `sl_sleeptimer_delay_millisecond()` 函数，由于该函数是阻塞式的，如果直接写在蓝牙程序里，会发现 EFR32 的蓝牙无法正常工作。

实际上，在基于空模板创建出的 `SL_WEAK void app_process_action()` 函数中，注释也非常清楚的写明了 `Do not call blocking functions from here!`：

```c
SL_WEAK void app_process_action(void)
{
  /////////////////////////////////////////////////////////////////////////////
  // Put your additional application code here!                              //
  // This is called infinitely.                                              //
  // Do not call blocking functions from here!                               //
  /////////////////////////////////////////////////////////////////////////////
}
```

因此，我们要修改一下上面这段 PWM 驱动代码，替换掉所有可能阻塞 CPU 的函数。

### 非阻塞式的呼吸灯驱动程序

回顾前面的呼吸灯驱动代码，程序实际上只有两个部分：

1. 让 LED 灯逐渐变亮
2. 让 LED 灯逐渐变暗

换句话说，LED 灯始终在两个状态（逐渐变亮和逐渐变暗）之间进行切换，那么很容易的就能想到，我们可以构建一个有限状态机来实现呼吸灯的控制功能：

```c
SL_WEAK void app_process_action()
{
  // 定义呼吸灯的两种状态
  static enum
  {
      brighten, darken
  } led_state;

  // 有限状态机
  switch (led_state)
  {
      case brighten:
          // 控制 LED 逐渐变亮的代码
          break;

      case darken:
          // 控制 LED 逐渐变暗的代码
          break;

      default:
          break;
    }
}
```

接下来加一点点功能：

```c
sl_sleeptimer_timer_handle_t timer;
static bool timeout_flag = true;

// 定时器超时回调
void timeout(sl_sleeptimer_timer_handle_t *handle, void *count)
{
    (void)&handle;
    sl_pwm_set_duty_cycle(&sl_pwm_led0, pwm_lut[*(uint8_t*)count]);
    timeout_flag = true;
}

// 非阻塞式 PWM 驱动程序
void led_pwm_process_action()
{
  static uint8_t count;
  static enum
  {
    brighten, darken
  } led_state;

  if (!timeout_flag)
  {
    return;
  }

  timeout_flag = false;
  switch (led_state)
  {
    case brighten:
      if (++count == 100)
      {
        led_state = darken;
        sl_sleeptimer_start_timer_ms(&timer, 190, timeout, &count, 0,
                                     SL_SLEEPTIMER_NO_HIGH_PRECISION_HF_CLOCKS_REQUIRED_FLAG);
        break;
      }
      sl_sleeptimer_start_timer_ms(&timer, 6, timeout, &count, 0,
                                   SL_SLEEPTIMER_NO_HIGH_PRECISION_HF_CLOCKS_REQUIRED_FLAG);
      break;

    case darken:
      if (--count == 0)
      {
        led_state = brighten;
        sl_sleeptimer_start_timer_ms(&timer, 190, timeout, &count, 0,
                                     SL_SLEEPTIMER_NO_HIGH_PRECISION_HF_CLOCKS_REQUIRED_FLAG);
        break;
      }
      sl_sleeptimer_start_timer_ms(&timer, 6, timeout, &count, 0,
                                   SL_SLEEPTIMER_NO_HIGH_PRECISION_HF_CLOCKS_REQUIRED_FLAG);
      break;

    default:
        break;
    }
}

// 主循环
SL_WEAK void app_process_action()
{
  led_pwm_process_action();
}
```

最后根据需要，继续增加 BLE 控制 LED 灯开关的功能就可以了。
