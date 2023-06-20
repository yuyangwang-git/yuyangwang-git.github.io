---
title: "EFR32入门笔记9 互补 PWM 输出"
date: 2023-05-21T13:27:23+08:00
draft: true

summary: Gecko Platform 的 PWM Driver 似乎无法直接输出互补 PWM，还是得回到 emlib 库手动编写相关代码
description: 

tags: ["C", "EFR32"]
categories: ["embedded"]

cover:
    image: 'img/cover.png'
---

## 输出互补 PWM

```C
// app.c

#include "em_cmu.h"
#include "em_timer.h"

void app_init(void)
{
    uint16_t freq;
    uint8_t duty;

    freq = 10 * 1000;
    duty = 5;

    CMU_Clock_TypeDef timer_clock = cmuClock_TIMER0;
    CMU_ClockEnable(timer_clock, true);
    CMU_ClockEnable(cmuClock_GPIO, true);

    // Set CC0 Pin
    GPIO_PinModeSet(gpioPortB, 1, gpioModePushPull, 0);
    // Set CDTI0 Pin
    GPIO_PinModeSet(gpioPortB, 2, gpioModePushPull, 0);

    // Route TIMER0 CC0 output
    GPIO->TIMERROUTE[0].ROUTEEN  = GPIO_TIMER_ROUTEEN_CC0PEN | GPIO_TIMER_ROUTEEN_CCC0PEN;
    GPIO->TIMERROUTE[0].CC0ROUTE = (gpioPortB << _GPIO_TIMER_CC0ROUTE_PORT_SHIFT)
                        | (1 << _GPIO_TIMER_CC0ROUTE_PIN_SHIFT);
    GPIO->TIMERROUTE[0].CDTI0ROUTE = (gpioPortB << _GPIO_TIMER_CDTI0ROUTE_PORT_SHIFT)
                        | (2 << _GPIO_TIMER_CDTI0ROUTE_PIN_SHIFT);

    // Configure TIMER frequency
    uint32_t top = (CMU_ClockFreqGet(timer_clock) / freq) - 1U;
    TIMER_TopSet(TIMER0, top);
    
    // Set initial duty cycle
    TIMER_CompareSet(TIMER0, 0, top * duty / 100);
    
    // Set CC channel parameters
    TIMER_InitCC_TypeDef channel_init = TIMER_INITCC_DEFAULT;
    channel_init.mode = timerCCModePWM;
    channel_init.cmoa = timerOutputActionToggle;
    channel_init.edge = timerEdgeBoth;
    channel_init.outInvert = 0;
    TIMER_InitCC(TIMER0, 0, &channel_init);
    
    // Set DTI parameters
    TIMER_InitDTI_TypeDef init_DTI = TIMER_INITDTI_DEFAULT;
    init_DTI.activeLowOut = true;
    init_DTI.invertComplementaryOut = false;
    init_DTI.riseTime = 1;
    init_DTI.fallTime = 1;
    TIMER_InitDTI(TIMER0, &init_DTI);
    
    // Initialize TIMER
    TIMER_Init_TypeDef timer_init = TIMER_INIT_DEFAULT;
    TIMER_Init(TIMER0, &timer_init);
}
```
