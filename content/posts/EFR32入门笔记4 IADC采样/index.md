---
title: "EFR32 入门笔记 4: IADC 采样"
date: 2023-03-15T16:32:30+08:00
draft: false

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
// app.c

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

先放源码：

```c
// app.c

#include <stdio.h>
#include "em_cmu.h"
#include "em_emu.h"                         // 增加 EMU, 降低系统能耗
#include "em_iadc.h"

#define CLK_SRC_ADC_FREQ        20000000    // CLK_SRC_ADC
#define CLK_ADC_FREQ            10000000    // CLK_ADC, ≤ 10 MHz, 根据 analog gain 调整

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

    // Use the FSRC0 as the IADC clock so it can run in EM2
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
    
    // Clear any previous interrupt flags
    IADC_clearInt(IADC0, _IADC_IF_MASK);

    // Enable single-channel done interrupts
    IADC_enableInt(IADC0, IADC_IEN_SINGLEDONE);

    // Enable IADC interrupts
    NVIC_ClearPendingIRQ(IADC_IRQn);
    NVIC_EnableIRQ(IADC_IRQn);
}

void app_process_action(void)
{
    IADC_command(IADC0, iadcCmdStartSingle);
    
    EMU_EnterEM2(false);                         // 进入 EM2 模式等待转换完成
    // or EMU_EnterEM1();

    printf("x=%lu\r\n", sample);
}

void IADC_IRQHandler(void)
{
    sample = IADC_pullSingleFifoResult(IADC0).data;
    singleResult = (sample / 0xFFF) * (1.21 / 0.5);

    IADC_clearInt(IADC0, IADC_IF_SINGLEDONE);
}
```

这里其实主要增加了以下几行：

```c
#include "em_emu.h"
```

没啥好说的，目的是为了调用 `EMU_EnterEM2()` 函数。

```c
    // Clear any previous interrupt flags
    IADC_clearInt(IADC0, _IADC_IF_MASK);

    // Enable single-channel done interrupts
    IADC_enableInt(IADC0, IADC_IEN_SINGLEDONE);

    // Enable IADC interrupts
    NVIC_ClearPendingIRQ(IADC_IRQn);
    NVIC_EnableIRQ(IADC_IRQn);
```

开启中断，常规操作了。

```c
void app_process_action(void)
{
    IADC_command(IADC0, iadcCmdStartSingle);
    
    EMU_EnterEM2(false);                         // 进入 EM2 模式等待转换完成
    // or EMU_EnterEM1();

    printf("x=%lu\r\n", sample);
}

void IADC_IRQHandler(void)
{
    sample = IADC_pullSingleFifoResult(IADC0).data;
    singleResult = (sample / 0xFFF) * (1.21 / 0.5);

    IADC_clearInt(IADC0, IADC_IF_SINGLEDONE);
}
```

中断处理函数也没啥好说的，中断列表全写在设备头文件 `efr32bg22c112f352gm32.h` 里：

```c
// efr32bg22c112f352gm32.h

/** Interrupt Number Definition */
typedef enum IRQn{
  /******  Cortex-M Processor Exceptions Numbers ******************************************/
  NonMaskableInt_IRQn    = -14,             /*!< -14 Cortex-M Non Maskable Interrupt      */
  HardFault_IRQn         = -13,             /*!< -13 Cortex-M Hard Fault Interrupt        */
  MemoryManagement_IRQn  = -12,             /*!< -12 Cortex-M Memory Management Interrupt */
  BusFault_IRQn          = -11,             /*!< -11 Cortex-M Bus Fault Interrupt         */
  UsageFault_IRQn        = -10,             /*!< -10 Cortex-M Usage Fault Interrupt       */
  SVCall_IRQn            = -5,              /*!< -5  Cortex-M SV Call Interrupt           */
  DebugMonitor_IRQn      = -4,              /*!< -4  Cortex-M Debug Monitor Interrupt     */
  PendSV_IRQn            = -2,              /*!< -2  Cortex-M Pend SV Interrupt           */
  SysTick_IRQn           = -1,              /*!< -1  Cortex-M System Tick Interrupt       */

  /******  EFR32BG22 Peripheral Interrupt Numbers ******************************************/

  CRYPTOACC_IRQn         = 0,  /*!<  0 EFR32 CRYPTOACC Interrupt */
  TRNG_IRQn              = 1,  /*!<  1 EFR32 TRNG Interrupt */
  PKE_IRQn               = 2,  /*!<  2 EFR32 PKE Interrupt */
  SMU_SECURE_IRQn        = 3,  /*!<  3 EFR32 SMU_SECURE Interrupt */
  SMU_S_PRIVILEGED_IRQn  = 4,  /*!<  4 EFR32 SMU_S_PRIVILEGED Interrupt */
  SMU_NS_PRIVILEGED_IRQn = 5,  /*!<  5 EFR32 SMU_NS_PRIVILEGED Interrupt */
  EMU_IRQn               = 6,  /*!<  6 EFR32 EMU Interrupt */
  TIMER0_IRQn            = 7,  /*!<  7 EFR32 TIMER0 Interrupt */
  TIMER1_IRQn            = 8,  /*!<  8 EFR32 TIMER1 Interrupt */
  TIMER2_IRQn            = 9,  /*!<  9 EFR32 TIMER2 Interrupt */
  TIMER3_IRQn            = 10, /*!< 10 EFR32 TIMER3 Interrupt */
  TIMER4_IRQn            = 11, /*!< 11 EFR32 TIMER4 Interrupt */
  RTCC_IRQn              = 12, /*!< 12 EFR32 RTCC Interrupt */
  USART0_RX_IRQn         = 13, /*!< 13 EFR32 USART0_RX Interrupt */
  USART0_TX_IRQn         = 14, /*!< 14 EFR32 USART0_TX Interrupt */
  USART1_RX_IRQn         = 15, /*!< 15 EFR32 USART1_RX Interrupt */
  USART1_TX_IRQn         = 16, /*!< 16 EFR32 USART1_TX Interrupt */
  ICACHE0_IRQn           = 17, /*!< 17 EFR32 ICACHE0 Interrupt */
  BURTC_IRQn             = 18, /*!< 18 EFR32 BURTC Interrupt */
  LETIMER0_IRQn          = 19, /*!< 19 EFR32 LETIMER0 Interrupt */
  SYSCFG_IRQn            = 20, /*!< 20 EFR32 SYSCFG Interrupt */
  LDMA_IRQn              = 21, /*!< 21 EFR32 LDMA Interrupt */
  LFXO_IRQn              = 22, /*!< 22 EFR32 LFXO Interrupt */
  LFRCO_IRQn             = 23, /*!< 23 EFR32 LFRCO Interrupt */
  ULFRCO_IRQn            = 24, /*!< 24 EFR32 ULFRCO Interrupt */
  GPIO_ODD_IRQn          = 25, /*!< 25 EFR32 GPIO_ODD Interrupt */
  GPIO_EVEN_IRQn         = 26, /*!< 26 EFR32 GPIO_EVEN Interrupt */
  I2C0_IRQn              = 27, /*!< 27 EFR32 I2C0 Interrupt */
  I2C1_IRQn              = 28, /*!< 28 EFR32 I2C1 Interrupt */
  EMUDG_IRQn             = 29, /*!< 29 EFR32 EMUDG Interrupt */
  EMUSE_IRQn             = 30, /*!< 30 EFR32 EMUSE Interrupt */
  AGC_IRQn               = 31, /*!< 31 EFR32 AGC Interrupt */
  BUFC_IRQn              = 32, /*!< 32 EFR32 BUFC Interrupt */
  FRC_PRI_IRQn           = 33, /*!< 33 EFR32 FRC_PRI Interrupt */
  FRC_IRQn               = 34, /*!< 34 EFR32 FRC Interrupt */
  MODEM_IRQn             = 35, /*!< 35 EFR32 MODEM Interrupt */
  PROTIMER_IRQn          = 36, /*!< 36 EFR32 PROTIMER Interrupt */
  RAC_RSM_IRQn           = 37, /*!< 37 EFR32 RAC_RSM Interrupt */
  RAC_SEQ_IRQn           = 38, /*!< 38 EFR32 RAC_SEQ Interrupt */
  RDMAILBOX_IRQn         = 39, /*!< 39 EFR32 RDMAILBOX Interrupt */
  RFSENSE_IRQn           = 40, /*!< 40 EFR32 RFSENSE Interrupt */
  PRORTC_IRQn            = 41, /*!< 41 EFR32 PRORTC Interrupt */
  SYNTH_IRQn             = 42, /*!< 42 EFR32 SYNTH Interrupt */
  WDOG0_IRQn             = 43, /*!< 43 EFR32 WDOG0 Interrupt */
  HFXO0_IRQn             = 44, /*!< 44 EFR32 HFXO0 Interrupt */
  HFRCO0_IRQn            = 45, /*!< 45 EFR32 HFRCO0 Interrupt */
  CMU_IRQn               = 46, /*!< 46 EFR32 CMU Interrupt */
  AES_IRQn               = 47, /*!< 47 EFR32 AES Interrupt */
  IADC_IRQn              = 48, /*!< 48 EFR32 IADC Interrupt */
  MSC_IRQn               = 49, /*!< 49 EFR32 MSC Interrupt */
  DPLL0_IRQn             = 50, /*!< 50 EFR32 DPLL0 Interrupt */
  PDM_IRQn               = 51, /*!< 51 EFR32 PDM Interrupt */
  SW0_IRQn               = 52, /*!< 52 EFR32 SW0 Interrupt */
  SW1_IRQn               = 53, /*!< 53 EFR32 SW1 Interrupt */
  SW2_IRQn               = 54, /*!< 54 EFR32 SW2 Interrupt */
  SW3_IRQn               = 55, /*!< 55 EFR32 SW3 Interrupt */
  KERNEL0_IRQn           = 56, /*!< 56 EFR32 KERNEL0 Interrupt */
  KERNEL1_IRQn           = 57, /*!< 57 EFR32 KERNEL1 Interrupt */
  M33CTI0_IRQn           = 58, /*!< 58 EFR32 M33CTI0 Interrupt */
  M33CTI1_IRQn           = 59, /*!< 59 EFR32 M33CTI1 Interrupt */
  EMUEFP_IRQn            = 60, /*!< 60 EFR32 EMUEFP Interrupt */
  DCDC_IRQn              = 61, /*!< 61 EFR32 DCDC Interrupt */
  EUART0_RX_IRQn         = 62, /*!< 62 EFR32 EUART0_RX Interrupt */
  EUART0_TX_IRQn         = 63, /*!< 63 EFR32 EUART0_TX Interrupt */
} IRQn_Type;
```
