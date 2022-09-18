---
title: "STM32启动流程"
date: 2022-09-16T20:46:17+08:00
draft: true

summary: 梳理一下 STM32 复位后发生了啥
description: 

tags: ["STM32"]
categories: ["embedded"]

cover: 
    image: 'img/cover.jpg'
---

## STM32启动流程

所有的 STM32 能且只能从以下 3 个区域启动：

* **主 Flash 存储器**（main flash memory），用户编写的程序通常都会储存在这里；
* **系统 Flash 存储器**（system flash memory），芯片生产时厂商在其中烧录了一段特殊的程序，该程序也被称之为 bootloader；
* **SRAM**（static random access memory），在调试时可以把程序烧录到 SRAM 而非 Flash 中，这样既可以提高下载速度，又可以减少 Flash 损耗。

> 系统 Flash 存储器 和 主 Flash 存储器并非两块物理上独立的 Flash，而是 STM32 内部 Flash 上人为划分出的两个区域。
>
![STM32 Embedded Flash Memory](img/flash.svg#center)

> SRAM 在 x86 架构的 CPU 中被用作寄存器和 Cache，DRAM 被用作内存。
>
> 而 STM32 等 MCU 使用 SRAM 而非 DRAM 作为内存，主要原因是 DRAM 需要反复刷新，故功耗和发热量远高于 SRAM，且刷新时无法访问，影响系统的实时性；
>
> — <cite>*Quora[^1]*</cite>

[^1]: [Why does Arduino (Atmega) use SRAM instead of DRAM?](https://www.quora.com/Why-does-Arduino-Atmega-use-SRAM-instead-of-DRAM).

### 启动起始位置是如何确定的？

不同型号的 STM32 的启动配置略有不同，但都可以在芯片对应的 reference manual 中找到相关内容的介绍，以 STM32L47xxx 为例，其启动方式由芯片上两个引脚 BOOT1 和 BOOT0 决定。

![Boot Modes](img/bootmodes.svg#center)

### 如果从系统存储器（System Memory）启动

> 本节内容在 AN2606 中有详细描述。
>
> — <cite>*AN2606 - STM32 microcontroller system memory boot mode[^2]*</cite>

[^2]: [STM32 microcontroller system memory boot mode](https://www.st.com/resource/zh/application_note/an2606-stm32-microcontroller-system-memory-boot-mode-stmicroelectronics.pdf).

前面提到，ST 在生产时向系统存储器中写入了 bootloader，该 bootloader 的主要任务就是控制芯片通过片上外设（USART，USB，I2C，SPI 等）将用户编写的程序下载到片内主存储器中。

bootloader 支持的片上外设和对应的 Appication Note 可参考下表：

| Protocol | Application Note |
|----------|------------------|
| CAN      | AN3154           |
| USART    | AN3155           |
| USB DFU  | AN3156           |
| I²C      | AN4221           |
| SPI      | AN4286           |
| FDCAN    | AN5405           |

> 这种启动方式主要在设计最小系统时需要考虑。

### 如果从主存储器（Main Memory）启动

这部分内容和 Cortex-M 架构有关，懒得往下写了，待更。

<!-- https://community.st.com/s/article/faq-stm32-boot-process -->