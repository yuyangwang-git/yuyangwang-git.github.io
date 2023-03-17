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

按惯例，先安装必需的依赖包：

![IADC](img/IADC.jpg#center)

和 USART 等外设不同，Sillicon Labs 并未给 IADC 提供相应的 Service 或 Driver，因此必须在底层外设库 `emlib` 上做开发。

### 预备知识

IADC 有两种工作模式：Single Channel Mode 和 Scan Mode，前者仅转换一个通道（或者说引脚）上的信号，而后者可以转换多个通道上的信号。

> Conversion results are written to the output data FIFO associated with the queue. Single queue results are written to the single data FIFO and scan queue results are written to the scan data FIFO. The two queues are identical in operation, but independent. Each FIFO can store up to four entries.

ADC 转换结果将会被写入到一个 FIFO 队列当中，该队列的容量为 4。

> The SHOWID bit controls whether the conversion channel ID is included in the output data word. This option is primarily used with the scan FIFO to help software determine which channel each conversion result came from. If SHOWID is enabled for single conversions, the ID will always be set to 0x20.

用户可以自行决定是否要为 ADC 转换结果增加一个 ID，该 ID 的作用是指示该结果是从哪个通道中读出的。而对于 Single Channel Mode，ID 将始终被设置为 0x20。

IADC 输出结果的分辨率由两个参数 $OversamplingRatio$ 和 $DigitalAveraging$ 决定：

$$
Output Resolution = 11 + log_2(OversamplingRatio \times DigitalAveraging)
$$

$OversamplingRatio$ 为过采样比（OSR），提高过采样比可以改善 ADC 的积分非线性（INL）和微分非线性（DNL）误差，并减小噪声的影响——显然此时转换速度也会下降。该过程为一个模拟过程。

$Digital Averaging$ 为移动平均，硬件会自动对转换结果（1、2、4、8 或 16 个采样结果）求平均值，求完均值后再将结果写入 FIFO 队列。该过程为一个数字过程。

> 什么是微分非线性（differential nonlinearity）和积分非线性（integral nonlinearity）？
>
> Analog-to-Digital Converter (ADC) Differential Non-Linearity (DNL) is defined as the maximum and minimum difference in the step width between the actual transfer function and the perfect transfer function. Non-linearity produces quantization steps with varying widths, some narrower and some wider.
>
> ![ADC DNL](img/ADCDNL.png#center)
>
> Analog-to-Digital Converter (ADC) Integral Non-Linearity (INL) is defined as the maximum vertical difference between the actual and the ideal curve. It indicates the amount of deviation of the actual curve from the ideal transfer curve.
>
> ![ADC INL](img/ADCINL.png#center)
>
> — <cite>ADC Differential Non-linearity[^1]</cite>

[^1]: [ADC Differential Non-linearity - Microchip Developer Helper](https://microchipdeveloper.com/adc:adc-differential-nonlinearity).

### 基于轮询的使用方式

创建 `Empty C Project`，其它文件保持不变，`app.c` 填入以下内容：

```c
#include <stdio.h>
#include "em_cmu.h"
#include "em_iadc.h"

#define CLK_SRC_ADC_FREQ        20000000  // CLK_SRC_ADC
#define CLK_ADC_FREQ            10000000  // CLK_ADC, ≤ 10 MHz, 根据 analog gain 调整

static volatile int32_t sample;
static volatile double singleResult;

void app_init(void)
{
    // Initialization structures
    IADC_Init_t init = IADC_INIT_DEFAULT;
    IADC_AllConfigs_t initAllConfigs = IADC_ALLCONFIGS_DEFAULT;
    IADC_InitSingle_t initSingle = IADC_INITSINGLE_DEFAULT;

    init.srcClkPrescale = IADC_calcSrcClkPrescale(IADC0, CLK_SRC_ADC_FREQ, 0); // 计算 CLK_SRC_ADC 时钟信号的预分频比
    init.warmup = iadcWarmupKeepWarm;

    initAllConfigs.configs[0].reference = iadcCfgReferenceInt1V2;              // 采用内部 1.21V 基准源
    initAllConfigs.configs[0].vRef = 1210;                                     // 基准源电压 毫伏
    initAllConfigs.configs[0].osrHighSpeed = iadcCfgOsrHighSpeed2x;            // 2x 过采样率
    initAllConfigs.configs[0].analogGain = iadcCfgAnalogGain0P5x;              // 0.5x 输入增益
    initAllConfigs.configs[0].adcClkPrescale = IADC_calcAdcClkPrescale(IADC0, CLK_ADC_FREQ,
                                                                       0, iadcCfgModeNormal,
                                                                       init.srcClkPrescale);  // 计算ADC_CLK 时钟信号的预分频比

    // Single input structure
    IADC_SingleInput_t singleInput = IADC_SINGLEINPUT_DEFAULT;

    singleInput.posInput = iadcPosInputPortAPin0;          // ADC 正输入引脚
    singleInput.negInput = iadcNegInputGnd;                // ADC 负输入引脚（单端测量直接接地）

    // Enable Clock
    CMU_ClockEnable(cmuClock_IADC0, true);
    CMU_ClockEnable(cmuClock_GPIO, true);
    CMU_ClockSelectSet(cmuClock_IADCCLK, cmuSelect_FSRCO);

    IADC_reset(IADC0);

    // Allocate the analog bus for ADC0 inputs
    // GPIO_ABUSALLOC, GPIO_BBUSALLOC, GPIO_CDBUSALLOC 对应 GPIO A, GPIO B, GPIO C
    // EVEN, ODD 对应偶数引脚，奇数引脚
    // Single-End 模式可以使用任意引脚
    // Differential 模式 positive input 必须为 even pin, negative input 必须为 odd pin
    // 例如, Single-End 下使用 PB 1 作为输入，则此处配置为: GPIO->BBUSALLOC |= GPIO_BBUSALLOC_BODD0_ADC0
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
    if ((IADC0->STATUS & (_IADC_STATUS_CONVERTING_MASK | _IADC_STATUS_SINGLEFIFODV_MASK)) != IADC_STATUS_SINGLEFIFODV)
    {
        ;
    }
    else
    {
        // 转换完成
        sample = IADC_pullSingleFifoResult(IADC0).data;
        singleResult = (sample / 0xFFF) * (1.21 / 0.5);
        printf("x=%lu\r\n", sample);
    }
}

```

## 基于中断的使用方式

待完善。
