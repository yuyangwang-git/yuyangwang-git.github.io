---
title: "元件参数误差对 CMRR 的影响"
date: 2022-10-23T18:34:17+08:00
draft: false

summary: 一个比较麻烦的模电计算题
description: 

cover: 
    image: 'img/differential_amplifier.svg'
---

## Question

> An op-amp differential amplifier is built using four identical resistors, each having a tolerance of ±5%. Calculate the worst possible CMRR.

## Answer

![differential amplifier](img/differential_amplifier.svg#center)

考虑 $CMRR$ 的倍数定义：

$$
CMRR = \frac{A_d}{A_c}
$$

为了求解该问题，我们可以先求出电路的共模增益 $A_c$ 及差模增益 $A_d$，然后计算电路的 $CMRR$．

为了求出 $A_c$ 及 $A_d$，就要找出运放的输入输出关系，对于具有差分输入端 $u_+$ 及 $u_-$ 的理想运算放大器，设其开环电压增益为 $A_{uo}$ ，则其输入输出满足：

$$
U_o = A_{uo}(U_+ - U_-)
$$

利用分压定律很容易得到：

$$
\begin{aligned}
U_+ &= \frac{R_2}{R_1 + R_2}U_{i+} \\\
U_- &= \frac{R_4}{R_3 + R_4}U_{i-} + \frac{R_3}{R_3 + R_4}U_o  \\\
\end{aligned}
$$

带入并化简得：

$$
\begin{aligned}
U_o &= A_{uo} (\frac{R_2}{R_1 + R_2}U_{i+} - \frac{R_4}{R_3 + R_4}U_{i-} - \frac{R_3}{R_3 + R_4}U_o) \\\

U_o + A_{uo}\frac{R_3}{R_3 + R_4}U_o &= A_{uo}(\frac{R_2}{R_1 + R_2}U_{i+} - \frac{R_4}{R_3 + R_4}U_{i-})\\\

U_o &= \frac{A_{uo} (\frac{R_2}{R_1 + R_2}U_{i+} - \frac{R_4}{R_3 + R_4}U_{i-})}{1 + A_{uo}\frac{R_3}{R_3 + R_4}} \\\

U_o &= \frac{\frac{R_2}{R_1 + R_2}U_{i+} - \frac{R_4}{R_3 + R_4}U_{i-}}{\frac{1}{A_{uo}} + \frac{R_3}{R_3 + R_4}} \\\
\end{aligned}
$$

显然，对于理想运放，其开环增益是无穷大的，那么有：

$$
\begin{aligned}
\lim\limits_{A_{uo} \to +\infty} U_o &= \frac{\frac{R_2}{R_1 + R_2}U_{i+} - \frac{R_4}{R_3 + R_4}U_{i-}}{\frac{R_3}{R_3 + R_4}} \\\
    &= \frac{\frac{(R_3 + R_4)R_2}{R_1 + R_2}U_{i+} - R_4U_{i-}}{R_3} \\\
    &= \frac{(R_3 + R_4)R_2}{(R_1 + R_2)R_3}U_{i+} - \frac{R_4}{R_3} U_{i-}
\end{aligned}
$$

接下来，我们的目标是求解运放的共模增益 $A_c$ 及差模增益 $A_d$，假设输入信号为共模信号（即直流信号），不妨设：

$$
U_{i+} = U_{i-} = U_i
$$

显然有：

$$
A_c = \frac{U_o}{U_i} = \frac{(R_3 + R_4)R_2}{(R_1 + R_2)R_3} - \frac{R_4}{R_3}
$$

假设输入信号为差模信号（即不含直流分量的交流信号），不妨设：

$$
\begin{aligned}
U_{i-} &= -0.5U_i \\\
U_{i+} &= +0.5U_i
\end{aligned}
$$

自然有：

$$
A_d = \frac{U_o}{U_i} = -\frac{1}{2}\frac{R_4}{R_3}-\frac{(R_3 + R_4)R_2}{2(R_1 + R_2)R_3}
$$

回到定义：

$$
\begin{aligned}
CMRR &= \frac{A_d}{A_c} \\\

&= \frac{1}{2} \frac{\frac{R_4}{R_3}+\frac{(R_3 + R_4)R_2}{(R_1 + R_2)R_3}}{\frac{R_4}{R_3} - \frac{(R_3 + R_4)R_2}{(R_1 + R_2)R_3}} \\\

&= \frac{1}{2} \frac{1 + \frac{(R_3 + R_4)R_2}{(R_1 + R_2)R_4}}{1 - \frac{(R_3 + R_4)R_2}{(R_1 + R_2)R_4}}
\end{aligned}
$$

问题转化为求 $CMRR$ 的最小值问题，不妨记：

$$
\begin{aligned}
k &= \frac{(R_3 + R_4)R_2}{(R_1 + R_2)R_4} \\\
&= \frac{(\frac{R_2R_3}{R_1R_4} + \frac{R_2}{R_1})}{1 + \frac{R_2}{R_1}} \\\
&= \frac{\frac{R_3}{R_4} + 1}{\frac{R_1}{R_2} + 1}
\end{aligned}
$$

那么有：

$$
CMRR = \frac{k + 1}{2 - 2k} = - \frac{1}{2} + \frac{1}{ 1 - k }
$$

只需求 $k$ 最小值，显然，当 $R_1$ 和 $R_4$ 取最大值 $1.05R$，且 $R_2$ 和 $R_3$ 取最小值 $0.95R$ 时得最值，即得：

$$
CMRR = 10
$$
