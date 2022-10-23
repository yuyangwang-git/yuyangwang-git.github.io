---
title: "元件误差对 CMRR 的影响"
date: 2022-10-23T18:34:17+08:00
draft: true

summary: 一个比较麻烦的模电计算题
description: 
---

## Qusetion

> An op-amp differential amplifier is built using four identical resistors, each having a tolerance of ±5%. Calculate the worst possible CMRR.

## Answer

![differential amplifier](img/differential_amplifier.svg#center)

对于具有差分输入端 $u_+$ 及 $u_-$ 的理想运算放大器，设其开环电压增益为 $A_uo$ ，则其输入输出满足：

$$
U_o = A_{uo}(U_+ - U_-)
$$

而我们知道：

$$
\begin{aligned}
U_+ &= \frac{R_2}{R_1 + R_2}U_{i+} \\\
U_- &= \frac{R_4}{R_3 + R_4}(U_{i-} - U_{o}) \\\
\end{aligned}
$$

带入并化简得：

$$
\begin{aligned}
U_o &= A_{uo} (\frac{R_2}{R_1 + R_2}U_{i+} - \frac{R_4}{R_3 + R_4}(U_{i-} - U_{o})) \\\
    &= A_{uo} (\frac{(R_3 + R_4)R_2}{R_1 + R_2}U_{i+} - \frac{(R_1 + R_2)R_4}{R_3 + R_4}(U_{i-} - U_{o})) \\\
U_o(1- A_{uo}\frac{(R_1 + R_2)R_4}{R_3 + R_4}) &= A_{uo} (\frac{(R_3 + R_4)R_2}{R_1 + R_2}U_{i+} - \frac{(R_1 + R_2)R_4}{R_3 + R_4}U_{i-}) \\\
U_o &= \frac{A_{uo} (\frac{(R_3 + R_4)R_2}{R_1 + R_2}U_{i+} - \frac{(R_1 + R_2)R_4}{R_3 + R_4}U_{i-})}{(1- A_{uo}\frac{(R_1 + R_2)R_4}{R_3 + R_4})} \\\
U_o &= \frac{\frac{(R_3 + R_4)R_2}{R_1 + R_2}U_{i+} - \frac{(R_1 + R_2)R_4}{R_3 + R_4}U_{i-}}{\frac{1}{A_{uo}} - \frac{(R_1 + R_2)R_4}{R_3 + R_4}} \\\
\end{aligned}
$$

显然，对于理想运放，其开环增益是无穷大的，那么有：

$$
\begin{aligned}
\lim_{A_{uo} \to +\infty}U_o &= \frac{\frac{(R_1 + R_2)R_4}{R_3 + R_4}U_{i-} - \frac{(R_3 + R_4)R_2}{R_1 + R_2}U_{i+}}{\frac{(R_1 + R_2)R_4}{R_3 + R_4}} \\\
    &= \frac{(R_1 + R_2)R_4U_{i-} - \frac{(R_3 + R_4)^2R_2}{R_1 + R_2}U_{i+}}{(R_1 + R_2)R_4} \\\
    &= U_{i-} - \frac{(R_3 + R_4)^2R_2}{(R_1 + R_2)^2R_4}U_{i+}
\end{aligned}
$$

接下来，我们的目标是求解运放的共模增益 $A_c$ 及差模增益 $A_d$，假设输入信号为共模信号（即直流信号），不妨设：

$$
U_{i+} = U_{i-} = U_i
$$

显然有：

$$
\begin{aligned}
A_c &= \frac{U_o}{U_i} \\\
    &= 1 - \frac{(R_3 + R_4)^2R_2}{(R_1 + R_2)^2R_4}
\end{aligned}
$$

假设输入信号为差模信号（即不含直流分量的交流信号），不妨设：

$$
\begin{aligned}
U_{i-} &= 0 \\\
U_{i+} &= U_i
\end{aligned}
$$

自然有：

$$
\begin{aligned}
A_d &= \frac{U_o}{U_i}\\\
    &= -\frac{(R_3 + R_4)^2R_2}{(R_1 + R_2)^2R_4}
\end{aligned}
$$

回到定义：

$$
CMRR = \frac{A_d}{A_c} = \frac{\frac{(R_3 + R_4)^2R_2}{(R_1 + R_2)^2R_4}}{\frac{(R_3 + R_4)^2R_2}{(R_1 + R_2)^2R_4} - 1}
$$

问题转化为求 $CMRR$ 的最小值问题，不妨记：

$$
k = \frac{(R_3 + R_4)^2R_2}{(R_1 + R_2)^2R_4}
$$

那么有：

$$
CMRR = \frac{k}{k - 1} = \frac{1}{1 - \frac{1}{k}}
$$

只需求 $k$ 最大值，显然，当 $R_3$ 和 $R_4$ 取最大值 $1.05R$，且 $R_1$ 和 $R_2$ 取最小值 $0.95R$ 时得最值，即：

$$
k = \frac{(2.1R)^2\times0.95R}{(1.9R)^2\times1.05R} \approx 1.105263 ...
$$

带回定义，可得：

$$
CMRR = 10.5
$$
