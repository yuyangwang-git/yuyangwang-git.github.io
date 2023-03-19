---
title: "EFR32 入门笔记 5: PWM 呼吸灯"
date: 2023-03-18T20:25:55+08:00
draft: false

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

看起来有点像指数函数？做一个非线性拟合：

![Nonlinear Regression](img/NonlinearRegression.svg#center)

可以看到拟合结果非常好，Lut 满足：

$$
y = 4.44837e^{0.03126x} - 4.58963
$$

挺有意思的一个结果，猜测这样设计，是因为比起线性增加的函数，亮度指数增加的视觉效果会更好，但是这样猜测是否符合实际呢？

过去在光度学里学过，人眼对亮度变化的响应并不是线性的，而是大致呈现为对数关系，换句话说，人眼对弱光的变化较为敏感，而对亮光的变化较为迟钝。

![Weber-Fechner's Law and Stevens' Power Law](img/Bright.png#center)

有很多关于人眼对光照响应的研究，例如上图中蓝线为 Weber-Fechne's Law，认为刺激强度和人类因刺激产生的感觉强度满足公式:

$$
S = 2.3klog_{10}(I)+C
$$

> 这也是为什么天文学中“视星等”采用对数进行定义！

红线为 Stevens' Power Law，满足：

$$
S = kI^a
$$

具体哪一种曲线更符合人眼的真实情况不在本文的讨论之列，有兴趣的话可参考[这篇文章](https://www.telescope-optics.net/eye_intensity_response.htm)。

从上面的讨论我们知道了，人眼对光照强度的响应呈现对数关系。也就是说，如果我们随时间线性提高光源的物理亮度，那么在人眼看来，光源的亮度的增速实际上是先快后慢的，看起来并不够平滑。

如果我们希望人眼感知到的亮度是随时间线性增加的，那显然就需要换一种方法来调整光源的物理亮度，考虑到对数函数的反函数是指数函数，只需要使用指数方式，先慢后快的提高光源物理亮度，就有希望达到上述目的，这可能就是上述 Lut 采用指数函数的原因。

实际上，如果我们把感知亮度作为横轴，光源真实亮度作为纵轴，就可以得到这样一条曲线：

![Perceived Light vs. Measured Light](img/PerceivedLight.svg#center)

> Human factors studies have shown that the eye perceives light in a logarithmic manner, mathematically speaking, in an approximately squared power relationship. Figure 1 illustrates this relationship.
>
>A linear profile is the most intuitive since when the dimming control is set to 50% or at 50% of its slider range then the measured light level is at 50%. However, with a linear profile the changes in light level at the low end of the dimming range are quite large and are sometimes viewed as step changes instead of smooth changes. This is due to the fact that the eye response is logarithmic and these step changes are perceived as being even larger than the measured percentages are. That being said, linear dimming profiles are adequate for many lighting applications that simply require dimming to be used for adjusting the light level for various tasks and times of day. In these applications the light level is adjusted occasionally and left at this level for a long period of time.  These applications actually comprise a majority of dimming requirements.
>
> The best simulation of the eye response is achieved with a logarithmic dimming profile. Figure 1 shows the typical eye response curve but slightly different logarithmic profiles are used by various LED driver manufacturers to enhance dimming performance. In addition to mimicking the eye response better, a logarithmic dimming profile enhances dimming smoothness at the lowest dimming levels. With the advent of the latest LED drivers that can achieve dimming to levels of nearly 0%, the availability of a logarithmic profile becomes even more important. Logarithmic profiles are also valuable for high- lumen-output luminaires (5000 lumens and above) in that they enable the dimming performance to be smoother because of the inherently wide lumen range of the light fixture. Applications requiring artistically ulta-smooth dimming such as theaters will greatly benefit from using logarithmic dimming profiles in all lumen ranges.
>
> — <cite>Linear vs. Logarithmic Dimming — A White Paper[^1]</cite>

[^1]: [Linear vs. Logarithmic Dimming — A White Paper](https://www.pathwaylighting.com/products/downloads/brochure/technical_materials_1466797044_Linear+vs+Logarithmic+Dimming+White+Paper.pdf).
