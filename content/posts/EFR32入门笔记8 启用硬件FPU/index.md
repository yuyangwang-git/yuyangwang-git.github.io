---
title: "EFR32 入门笔记 8: 启用硬件 FPU"
date: 2023-04-05T11:10:11+08:00
draft: true

summary: 信号处理速度++
description: 

tags: ["C", "EFR32"]
categories: ["embedded"]

cover:
    image: 'img/cover.jpg'
---

## 如何判断 SoC 有没有硬件 FPU 和 DSP 单元？

很简单，看数据手册：

![EFR32 Block Diagram](img/EFR32BlockDiagram.svg#center)

Core 部分写的很清楚：`ARM CortexTM M33 processor with DSP extensions, FPU and TrustZone`。

## SoC / MCU 有没有硬件 FPU 单元与其采用的架构有关吗？

答案是有关系，以最常见的 Cortex-M 架构为例，并非所有 Cortex-M 内核处理器都具有硬件 FPU 和 DSP 单元，详细的列表可以在 [Arm Cortex-M Processor Comparison Table](https://developer.arm.com/documentation/102787/latest/) 中找到：

![Arm Cortex-M Processor Comparison Table](img/Comparison_Table.svg#center)

可以看到 EFR32BG22 所采用的 Cortex-M33 内核是可以选配 Floating-Point Unit 和 Digital Signal Processing Extension 的。

> 需要注意，Cortex-M33 只能选配单精度（Single-Precision）单元。

![Cortex-M33](img/Cortex-M33.svg#center)

## 如何使用硬件 FPU 加速浮点运算？

### 背景知识

在 [Arm Cortex-M33 Devices Generic User Guide](https://developer.arm.com/documentation/100235/0004/the-cortex-m33-peripherals/floating-point-unit/code-sequence-for-enabling-the-fpu?lang=en) 中以汇编形式给出了 FPU 的启动代码：

```armasm
CPACR   EQU  0xE000ED88
LDR     R0,  =CPACR          ; Read CPACR
LDR     r1, [R0]             ; Set bits 20-23 to enable CP10 and CP11 coprocessors
ORR     R1, R1, #(0xF << 20)
STR     R1, [R0]             ; Write back the modified value to the CPACR
DSB
ISB                          ; Reset pipeline now the FPU is enabled.
```

从 User Guide 里可以看到，芯片复位后，FPU 会被默认禁用，需要我们先配置 CPACR 寄存器开启 FPU 单元，然后才能调用 FPU 相关汇编指令，上面这段汇编代码实际上就是在配置 CPACR 寄存器。

换句话说，使用 FPU 加速浮点运算实际上需要我们做两件事：

1. 在程序中，配置 CPACR 寄存器以开启硬件 FPU 功能
2. 在程序中，使用浮点运算指令（`vadd` 等）完成浮点运算

但我们写C语言程序时，做浮点运算都是直接使用加减乘除等数学运算符，C 语言中并没有汇编层面的“浮点运算指令”。实际上是编译器决定了数学运算最终在汇编层面是如何实现的。因此，我们需要调整编译器设置，来确保编译出的程序中自动嵌入了 FPU 的启动代码并使用浮点运算指令完成浮点运算。

### 实操部分

第一步，我们要先确认编译器版本，因为不同的编译器的配置方法也略有区别。

笔者使用的是 Simplicity Studio 5，在创建 Project 时，IDE / Toolchain 选择了：`Simplicity IDE / GNU ARM v10.3.1`，所以我使用的编译器就是 `GNU ARM`，也即 `gcc`，版本 `v10.3.1`。

> Keil, IAR 等 IDE 默认使用其它编译器，如 ARMCC。

第二步，
