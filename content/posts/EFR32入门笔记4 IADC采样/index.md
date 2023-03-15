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

参考文档 [AN1189: Incremental Analog to Digital Converter (IADC)](https://www.silabs.com/documents/public/application-notes/an1189-efr32-iadc.pdf)

按惯例，先安装必须的依赖包：

![IADC](img/IADC.jpg#center)

和 USART 等外设不同，Sillicon Labs 并未给 IADC 提供相应的 Service 或 Driver，因此必须在底层外设库 `emlib` 上做开发。

### 预备知识

IADC 有两种工作模式：Single Channel Mode 和 Scan Mode，前者仅转换一个通道（或者说引脚）上的信号，而后者可以转换多个通道上的信号。

> Conversion results are written to the output data FIFO associated with the queue. Single queue results are written to the single data FIFO and scan queue results are written to the scan data FIFO. The two queues are identical in operation, but independent. Each FIFO can store up to four entries.

ADC 转换结果将会被写入到一个 FIFO 队列当中，该队列的容量为 4。

> The SHOWID bit controls whether the conversion channel ID is included in the output data word. This option is primarily used with the scan FIFO to help software determine which channel each conversion result came from. If SHOWID is enabled for single conversions, the ID will always be set to 0x20.

用户可以自行决定是否要为 ADC 转换结果增加一个 ID，该 ID 的作用是指示该结果是从哪个通道中读出的。而对于 Single Channel Mode，ID 将始终被设置为 0x20。

IADC 输出结果的分辨率由两个参数 $OversamplingRatio$ 和 $DigitalAveraging$ 决定：

$$Output Resolution = 11 + log_2(OversamplingRatio \times DigitalAveraging)$$

$OversamplingRatio$ 为过采样比（OSR），提高过采样比可以改善 ADC 的积分非线性（INL）和微分非线性（DNL）误差，并减小噪声的影响——显然此时转换速度也会下降。该过程为一个模拟过程。

$Digital Averaging$ 为移动平均，硬件会自动对转换结果（1、2、4、8 或 16 个采样结果）求平均值，求完均值后再将结果写入 FIFO 队列。该过程为一个数字过程。

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

## 基于中断的使用方式

待完善。
