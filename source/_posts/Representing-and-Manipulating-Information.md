---
title: 数字在计算机的表示和操作
date: 2023-07-08 14:51:22
tags:
math: true
---
# Integer Representations

在学校教科书告诉我们原码，反码，补码的规则来计算正负数在计算机的表示。
>原码：将一个整数，转换成二进制，就是其原码。
如单字节的5的原码为：0000 0101；-5的原码为1000 0101。
反码：正数的反码就是其原码；负数的反码是将原码中，除符号位以外，每一位取反。
如单字节的5的反码为：0000 0101；-5的反码为1111 1010。
补码：正数的补码就是其原码；负数的反码+1就是补码。
如单字节的5的补码为：0000 0101；-5的补码为1111 1011。
在计算机中，正数是直接用原码表示的，如单字节5，在计算机中就表示为：0000 0101。
负数用补码表示，如单字节-5，在计算机中表示为1111 1011。

规则不仅复杂抽象不说，也无助于我们理解无符号和有符号的转换，无符号与有符号的扩展。当我阅读CSAPP Chapter 2 Representing and Manipulating Information，忽然如沐春风，豁然开朗。如下是部分笔记。

## Unsigned and Two’s-Complement Encodings
Assume we have an integer data type of $\omega$ bits. We write a bit vector as either $\vec{x}$, to denote the entire vector, or as $[x_{w-1}, x_{w-2}, ..., x_0 ]$ to denote the individual bits within the vector. Treating $\vec{x}$  as a number written in binary notation, we obtain the unsigned interpretation of $\vec{x}$. We express this interpretation as a function $B2U_w$ (for “binary to unsigned”, length $\omega$)

$$
\begin{equation}
B2U_w(\vec{x}) = \sum_{i = 0}^{w-1} x_i 2^i
\end{equation}
$$

We wish to represent negative values as well by two’s-complement form. This is defined by interpreting the most significant bit of the word to have negative weight. We express this interpretation as a function $B2T_w$(for “binary to two’s-complement” length $\omega$ 	) 

$$
\begin{equation}
B2T_w(\vec{x}) = -x_{w - 1} 2^{w-1} +  \sum_{i = 0}^{w-2} x_i 2^i
\end{equation}
$$

Problem: find the $B2T_w(-5)$

$$
\begin{align}
\because {5 + -5 = 0} \\
0000 \space 0101 + 1\space\underbrace{\vec{x_6, x_5, ..., x_0}}_\text{x} = 0 \\
\underbrace{\vec{x_6, x_5, ..., x_0}}_\text{x} = 1000 \space 000 - 0000 \space 0101 \\
\underbrace{\vec{x_6, x_5, ..., x_0}}_\text{x} = 0111 \space 1011\\
\therefore {B2T_w(-5) = 1111 \space 1011}
\end{align}
$$

### Conversions Between Singed and Unsigned

$$
\begin{gather}
U2B_w(x) = B2U_w^{-1}(x) \\
T2B_w(x) = B2T_w^{-1}(x)
\end{gather}
$$

These functions give the unsigned or two’s-complement bit patterns for a numeric value. Consider the function

$$
\begin{gather}
U2T_w(x) = B2T_w(U2B_w(x))
\end{gather}
$$

which takes a number between 0 and $2^w - 1$ and yields a number between $[-2^{w - 1},\space 2^{w - 1}]$, where the two numbers have identical bit representations, except that the argument is unsigned, while the result has a two’s-complement representation. Conversely, the function

$$
\begin{gather}
T2U_w(x) = B2U_w(T2B_w(x))
\end{gather}
$$

yields the unsigned number having the same bit representation as the two’s-complement value of x.

To get a better understanding of the relation between a signed number $x$ and its unsigned counterpart $T2U_w(x)$, we can use the fact that they have identical bit representations to derive a numerical relationship.

$$
\begin{align}
B2U_w(\vec{x}) - B2T_w(\vec{x}) = x_{w-1}(2^{w-1} - - 2^{w-1}) = x_{w-1}2^w \\
\end{align}
$$

This gives a relationship $\textcolor{red}{B2U_w(\vec{x}) = x_{w-1}2^w + B2T_w(\vec{x})}$. If we let $x = B2T_w(\vec{x})$, we then have

$$
\begin{equation}
B2U_w(T2B_w(x)) = T2U_w(x) = x_{w-1}2^w + x
\end{equation}
$$

This relationship is useful for proving relationships between unsigned and two’s-complement arithmetic. In the two’s-complement representation of $x$, bit $x_{w-1}$ determines whether or not $x$ is negative, giving

$$
\begin{equation}
T2U_w(x) = 
\begin{cases}
x + 2^w, & x \lt 0\\
x, & x \ge 0
\end{cases}
\end{equation}
$$
![](https://pic.imgdb.cn/item/64a90fd21ddac507ccc20279.png)
Going in the other direction, we wish to derive the relationship between an unsigned number $x$ and its signed counterpart $U2T_w(x).$ If we let $x = B2U_x(\vec{x})$ , then $\vec{x} = U2B(x)$ we have

$$
\begin{equation}
B2T_w(U2B_w(x)) = U2T_w(x) = -x_{w-1}2^w + x
\end{equation}
$$

In the unsigned representation of $x$, bit $x_{w-1}$ determines whether or not $x$ is greater than or equal to $2^{w-1}$, giving

$$
\begin{equation}
U2T_w(x) = 
\begin{cases}
x, & x \lt 2^{w-1}\\
x - 2^w, & x \ge 2^{w-1}
\end{cases}
\end{equation}
$$

![](https://pic.imgdb.cn/item/64a90fd11ddac507ccc20139.png)

## 2.2.5 Expanding the Bit Representation of a Number

zero extension: To convert an unsigned number to a larger data type, we simply add leading 0s to the representation.

sign extension: To convert an signed(two’s-complement) number to a larger data type, , add leading most significant bit to the representation.

$$
\begin{align}
B2T_{w+1}([x_{w-1}, x_{w-1}, x_{w-2},...x_0]) &= -x_{w-1}2^w + [\sum_0^{w-1}x_i2^i] \\
&=  -x_{w-1}2^w + x_{w-1}2^{w-1}  +[\sum_0^{w-2}x_i2^i] \\
& = -x_{w-1}2^{w-1}  +[\sum_0^{w-2}x_i2^i] \\
& = B2T_w([x_{w-1},x_{w-2},...x_0]
\end{align}
$$

## Truncating Numbers

$$
\begin{align}
B2U_w([x_{w-1}, x_{w-2},...x_0]) \space mod \space  2^k &= [\sum_0^{w-1}x_i2^i]\space mod\space  2^k \\
&= [\sum_0^{k-1}x_i2^i] \space mod \space 2^k \\
& = [\sum_0^{k-1}x_i2^i] \\
& = B2U_w([x_{k-1},x_{k-2},...x_0]
\end{align}
$$

### 2.4.1 Fractional Binary Numbers

Fractional binary notation:

$$
\begin{equation}
b = b_mb_{m-1}...b_1b_0.b_{-1}...b_{-n} = \sum_{i=-n}^mi^i * b_i
\end{equation}
$$

This notation is not efficient for representing $5 * 2^{100}$, which would consist of the bit pattern 101 followed by 100 zeros. Instead, we would like to represent numbers in a form $x * 2^y$ by giving the values of $x$ and $y$.

The IEEEE floating-point standard represents a number in a form $v = (-1)^s * M * 2^E$

- The sign s determines whether the number is negative ( s = 1) or positive (s = 0).
- The significant M is a fractional binary number that ranges either between 1 and $2 - \epsilon$ or between 0 and $1 - \epsilon$
- The exponent E weights the value by a (possibly negative ) power of 2.

The bit representation of a floating-point number is divided into three fields to encode these values:

- The single sign bit s directly encodes the sign s.
- The k-bit exponent field $exp = 3_{k-1}...e_1e_0$ encodes the exponent E.
- The n-bit fraction field $frac = f_{n - 1}...f_1f_0$ encodes the significant M.

Normalized Values

The bit pattern of $exp$ is neither all 0s nor all 1s. The exponent value is $E = e - Bias$ where $e$ is the unsigned number having bit representation $e = e_{k-1}...e_1e_0$, and Bias is a bias value $bias = 2^{k - 1} - 1$. $M = 1 + 0.f_{n-1}...f_1f_0$

Denormalized Values

When the exponent field is all 0s.

- $E = 1 - Bias$
- $M = f$

Special Values
- Infinity: When the exponent field is all 1s and the fraction field is all 0s.
- NaN: When the exponent field is all 1s but the fraction field is nonzero.

Problem 2.33: k = 2, n = 2,  $bias = 2^{2 - 1} -1 = 1$

| Bits | e | E | f | M | V |
| --- | --- | --- | --- | --- | --- |
| 0 00 00 | 0 | 0 ( 1 - bias) | 0 | 0 | 0 |
| 0 00 01 | 0 | 0 ( 1 - bias) | 1/4 | 1/4 | 1/4 |
| 0 00 10 | 0 | 0 ( 1 - bias) | 2/4 | 2/4 | 2/4 |
| 0 00 11 | 0 | 0 ( 1 - bias) | 3/4 | 3/4 | 3/4 |
| 0 01 00 | 1 | 0 ( e - bias) | 0 | 4/4 | 4/4 |
| 0 01 01 | 1 | 0 | 1/4 | 5/4 | 5/4 |
| 0 01 10 | 1 | 0 | 2/4 | 6/4 | 6/4 |
| 0 01 11 | 1 | 0 | 3/4 | 7/4 | 7/4 |
| 0 10 00 | 2 | 1 | 0 | 4/4 | 8/4 |
| 0 10 01 | 2 | 1 | 1/4 | 5/4 | 10/4 |
| 0 10 10 | 2 | 1 | 2/4 | 6/4 | 12/4 |
| 0 10 11 | 2 | 1 | 3/4 | 7/4 | 14/4 |
| 0 11 00 | - | - | - | - | $+\infty$ |
| 0 11 01 | - | - | - | - | NaN |