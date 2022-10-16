---
title: "ADC 量化噪声"
date: 2022-10-13T16:01:24+08:00
draft: true

summary: 在一些 ADC 的应用笔记中常看到信噪比计算公式 SNR = 6.02N + 1.76dB，恰巧又在同学的课题里看到了该公式，于是深挖了一下这个公式的来龙去脉．
description: 

tags: ["ADC"]
categories: ["embedded", "signal processing"]

cover: 
    image: 'img/quantization.svg'
---

## 采样与量化过程

![analog](img/analog.svg#center)

模拟信号可视为时间相关的连续函数（如上图），该函数在任意时刻的值被称为信号的幅度．因此，在将连续的模拟信号转换为离散化的数字信号过程中，涉及到两个步骤：

1. 在连续的时间轴上对信号进行离散化，即采样（sampling）
2. 在连续的幅度轴上对信号进行离散化，即量化（quantization）

将模拟信号转换为数字信号的过程必然伴随着信息的丢失（信号失真），为保证数字信号能尽可能准确的反应原始模拟信号的特征，就需要提高采样及量化的精度．

采样的精度由采样率（sampling rate）决定，量化的精度则由量化位数（对应 ADC 的转换位数）决定．

在实践中受限于成本等因素，我们不能无限制的提高信号的采样率和量化位数．所幸，当采样率满足一定条件时，我们可以从采样结果中无失真地恢复原始信号．

> **对于频谱分布在 $(0, f_h)$ 之间的带限信号**，当采样频率 $f_s$ 满足 $f_s>2f_h$ 时，可以无失真的恢复原始信号．
>
>  — <cite>Nyquist 低通采样定律</cite>

![quantization error](img/error.svg#center)

与采样不同，量化过程一定会引入无法恢复的量化误差（上图给出了封面中量化结果的误差曲线，即 $y = signal - quantization$），本文将讨论量化误差的产生及其对信号质量的影响．

> **为什么量化误差无法恢复？**
>
> 量化在输入与输出之间建立起一种从多到少的映射，多个不同的输入将对应同一个输出，该过程伴随大量原始信息丢失，无法仅靠输出值恢复出原有的输入值，因此量化是一个不可逆的过程．
>
> **采样过程也伴随信息的丢失，但为什么信号仍然可以被无失真的恢复？**
>
> 在产生这一疑问之前，对 Nyquist 低通采样定律成立的前提条件没有一个更清晰的认识．
>
> 如果我们已知函数上**若干等间距点$(x_i, y_i)$**，能从这些点求出函数的解析式吗？当然不能，因为我们可以构造出无穷多个满足$y_i=f(x_i)$的函数$f(x)$．
>
> 如果增加前提条件，限制该函数为正弦函数呢？依然不能求出解析式，因为还是可以构造出无穷多个经过这些点正弦函数，只需保证这些正弦函数的周期成倍数关系即可．
>
> 现在回过头来看采样定理，**“频谱分布在 $(0, f_h)$ 之间的带限信号”**，我们把信号分解为若干具有不同频率的正弦分量，其中最高频率分量的频率记为 $f_h$ ，是一个有限值．这是一个非常强的限制条件，强到现实中几乎找不到这样的理想带限信号．
>
> 在这一前提下，信号的最高频率分量也只是一个频率小于等于 $f_h$ 的正弦波，所以只要采样频率大于 $2f_h$，自然可以计算出该信号的波形．
>
> **能否突破采样定理的限制？**
>
> 在前面的论述中，刻意强调了**等间距**的表述，这是因为采样定理中已经指定采样频率为恒定值 $f_s$．那如果采取**非均匀采样**策略，能否以较低的采样频率恢复出更高频的信号呢？这就是 Terence Tao 等在 2007 年提出的**压缩感知**（compressed sensing）理论中回答的问题——当信号具有稀疏性时，能以远低于采样定理所要求的采样数重建原始信号．

## 量化

正如前面所讨论的，量化是将在数轴上连续信号的幅值近似为位数有限的离散值的过程（如封面），该过程由量化器（quantizer）实现．

实际上，我们所熟知的四舍五入算法就是一种最基本的量化器，该算法将带有小数部分的输入映射到了最接近的整数，构成了一种 Mid-Tread 型均匀量化器．

> 量化可分为均匀量化和非均匀量化，本文仅讨论最常见的均匀量化．均匀量化的量化级为定值．

> 量化级，采样位数，采样位深等词语都是同一个概念的不同表述方法．

均匀量化器包含 “Mid-Tread” 和 “Mid-Rise” 两种类型，二者的图像可参考下图：

![quantization type](img/quantization_types.jpg#center "Mid-Rise 与 Mid-Tread 量化类型")

> 这两种类型的命名与原点的位置有关，如果我们把量化形成的阶梯状信号比作楼梯，Mid-Tread 指原点落在楼梯踏步中间时的情形，而 Mid-Rise 指原点落在楼梯梯面时的情形．
>
> Tread 和 Riser 指楼梯踏步（stair tread）和楼梯梯面（stair riser）
>
> ![stairway system components](img/stairway.png#center)
>
> Tread: the part you step on.
>
> Riser: the vertical part behind the tread. Combined with the tread, this creates the step.
>
>  — <cite>Defining the components of a commercial stairway system[^1]</cite>

[^1]: [How To Design a Commercial Stairway](https://blog.manningtoncommercial.com/how-to-design-a-commercial-stairway).

绝大多数 ADC 都属于 Mid-Tread 型量化器．以德州仪器的 12 位 ADC TLC2543-EP 为例，厂商在数据手册中给出了器件的输入输出关系：

![transfer_characteristic](img/transfer_characteristic.svg#center)

该芯片为一个典型的 Mid-Tread 量化器，当该芯片的 $V_{ref+}$ 和 $V_{ref-}$ 两个引脚被恰当连接时，芯片的 LSB 为 1.2mV．

> n-bit ADC 的转换结果对应 $2^n$ 个数字量，每个数字量对应一个模拟电平，最低位数字量所对应的模拟电平被称为最小有效位（Least Significant Bit，LSB），n 也被称为 ADC 的分辨率．

### 量化误差

> 本节将介绍一个量化误差的近似计算模型，尽管该模型是在一些假设的基础上提出的，但依然可以用于绝大多数信号的信噪比分析．

TLC2543-EP 在特定工况下 LSB 为 1.2mV，即芯片无法捕获小于 1.2mV 的电压变化．这就意味着芯片的输出与真实值之间存在量化误差 $e_{q}$，此时对于任意的输入信号（当然，输入信号的大小不能超过 ADC 量程），芯片产生的量化误差的最大值 $e_{qmax}$ 满足：

$$
e_q = Input - Output \leqslant e_{qmax} = \frac{1}{2} LSB
$$

显然，量化误差的大小与输入信号是直接相关的，但是如果我们考虑到两个事实：

1. 实际使用 ADC 采样时，会将 ADC 的动态范围与输入信号的最大幅度进行匹配以获得最大分辨率，**此时输入信号将横跨若干 LSB**；

> ~~相信没有傻子会干出浪费 ADC 分辨率的蠢事来~~，采样微弱交流信号时会使用前置放大器或调整 ADC 的参考电压．

2. 为了尽可能的提高分辨率，我们会倾向于选择具有更高位数的 ADC．

就可以对量化误差做进一步的简化，将其视为一个叠加在原始信号上的锯齿状噪声，且我们近似的认为该噪声与输入无关：

![s](img/ideal_quantization_error.svg#center)

实际上这种假设对于绝大多数信号都是成立的，对于图示的锯齿状噪声，我们可以计算其 RMS：

$$
{RMS}_{noise} = \sqrt{\dfrac{{LSB}^2}{12}} = \dfrac{LSB}{\sqrt{12}}
$$

> * ${RMS}^2$ 为均方值（对幅值的平方求平均值），反映了信号功率；
>
> * $RMS$ 为均方根，也即有效值，二者均可用于信噪比的计算．

上式可以看出，该模型下，量化噪声的功率只与 ADC 的最小分辨率有关，而与采样频率无关．

接下来就可以计算 ADC 的理想信噪比：

$$
SNR = 20\lg\dfrac{{RMS}\_{signal}}{{RMS}_{noise}}
$$

为了给出 ADC 信噪比的计算公式，我们可以假设输入信号为正弦波，其峰-峰值恰覆盖 ADC 的量程，即：

$$
V_{input} = \dfrac{2^{n}}{2} LSB \cdot \sin(\omega x)
$$

而正弦波的有效值为：

$$
RMS_{signal}= \dfrac{2^n}{2 \sqrt{2}}LSB
$$

故理想信噪比为：

$$
\begin{aligned}
    SNR &= 20\lg\dfrac{{RMS}\_{signal}}{{RMS}_{noise}} \\\
    &= 20\lg 2^n + 20\lg\sqrt{\dfrac{3}{2}} \\\
    &\approx 6.02N + 1.76dB
\end{aligned}
$$

### 如何最小化量化误差的影响

1. 选择具有更高分辨率的 ADC，如 16bit 甚至 24bit 的型号．
2. 过采样 + 数字滤波

待完善．
