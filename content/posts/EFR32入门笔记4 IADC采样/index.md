---
title: "EFR32 入门笔记 4: IADC 采样"
date: 2023-03-15T16:32:30+08:00
draft: true

summary: 试一试 EFR32 的片内 16-bit ADC
description: 有手就行

tags: ["C", "EFR32"]
categories: ["embedded"]

cover: 
    image: 'img/cover.jpg'
---

## IADC 外设

按惯例，先安装必须的依赖包：

![IADC](img/IADC.jpg#center)

和 USART 等外设不同，Sillicon Labs 并未给 IADC 提供相应的 Service 或 Driver，因此必须在底层外设库 `emlib` 上做开发。

### 基于轮询的使用方式

创建 `Empty C Project`，其它文件保持不变，`app.c` 填入以下内容：

```c
#include <stdio.h>
#include "em_cmu.h"
#include "em_iadc.h"

#define CLK_SRC_ADC_FREQ        20000000  // CLK_SRC_ADC
#define CLK_ADC_FREQ            10000000  // CLK_ADC - 10 MHz max in normal mode

static volatile int32_t sample;
static volatile double singleResult;

void app_init(void)
{
    // Initialization structures
    IADC_Init_t init = IADC_INIT_DEFAULT;
    IADC_AllConfigs_t initAllConfigs = IADC_ALLCONFIGS_DEFAULT;
    IADC_InitSingle_t initSingle = IADC_INITSINGLE_DEFAULT;

    init.srcClkPrescale = IADC_calcSrcClkPrescale(IADC0, CLK_SRC_ADC_FREQ, 0);
    init.warmup = iadcWarmupKeepWarm;

    initAllConfigs.configs[0].reference = iadcCfgReferenceInt1V2;
    initAllConfigs.configs[0].vRef = 1210;
    initAllConfigs.configs[0].osrHighSpeed = iadcCfgOsrHighSpeed2x;
    initAllConfigs.configs[0].analogGain = iadcCfgAnalogGain0P5x;
    initAllConfigs.configs[0].adcClkPrescale = IADC_calcAdcClkPrescale(IADC0,
                                                                        CLK_ADC_FREQ,
                                                                        0,
                                                                        iadcCfgModeNormal,
                                                                        init.srcClkPrescale);

    // Single input structure
    IADC_SingleInput_t singleInput = IADC_SINGLEINPUT_DEFAULT;

    singleInput.posInput = iadcPosInputPortAPin0;;
    singleInput.negInput = iadcNegInputGnd;

    // Enable Clock
    CMU_ClockEnable(cmuClock_IADC0, true);
    CMU_ClockEnable(cmuClock_GPIO, true);
    CMU_ClockSelectSet(cmuClock_IADCCLK, cmuSelect_FSRCO);

    IADC_reset(IADC0);

    // Allocate the analog bus for ADC0 inputs
    GPIO->ABUSALLOC |= GPIO_ABUSALLOC_AEVEN0_ADC0;

    // Initialize IADC
    // Initialize a single-channel conversion
    IADC_init(IADC0, &init, &initAllConfigs);
    IADC_initSingle(IADC0, &initSingle, &singleInput);
}

void app_process_action(void)
{
    IADC_command(IADC0, iadcCmdStartSingle);

    // 等待 ADC 转换完成
    while((IADC0->STATUS & (_IADC_STATUS_CONVERTING_MASK
                | _IADC_STATUS_SINGLEFIFODV_MASK)) != IADC_STATUS_SINGLEFIFODV);
    
    sample = IADC_pullSingleFifoResult(IADC0).data;

    printf("x=%lu\r\n", sample);
}

```

待完善。