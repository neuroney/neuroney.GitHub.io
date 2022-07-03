---
title: QAP 详解
mathjax: true
date: 2022-07-01 15:53:34
categories: Cryptography
---

本篇文章是学习 Quadratic Arithmetic Program（QAP）时，所做的一些记录。

QAP 是一种计算模型。在设计 ZK-snark 时，我们需要将一个 NP 的问题转换为一种特定的计算形式，这种形式就是 QAP。QAP 的通常工作流程是：程序（program） $\rightarrow$ 算数电路（arithmetic circuit） $\rightarrow$ 秩为1的约束系统（rank-1 constraint system，R1CS）$\rightarrow$ QAP。

<!-- more -->

将一个可执行程序、代码转换为一个 QAP，是很复杂的过程。我们可以使用如 [ZoKrates](https://zokrates.github.io/rng_tutorial.html) 和 [libsnark](https://github.com/scipr-lab/libsnark) 这样的编译程序，将给定的代码程序转换为 QAP。本篇文章，仅以简单的多项式为例，来说明 QAP 的转换流程。 

## 一 、从多项式到算数电路

考虑如下的多项式函数：

$$F: \mathbb{F}_{11}\times \mathbb{F}_{11}\times \mathbb{F}_{11}\times \mathbb{F}_{11} \rightarrow \mathbb{F}_{11} \mapsto (x_1+x_2)\cdot (x_3\cdot x_4)$$

在域 $\mathbb{F}_{11}$ 上的算数电路是有向的非循环图。上述函数可以表达为如下的算数电路：

<img src="https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/QAP%20%E8%AF%A6%E8%A7%A3/Example%20circuit.png" alt="Example circuit" style="zoom:45%;" />

- 该电路中有两个乘法门，分别记为 $r_5$ 和 $r_6$，对应输出线 $c_5$ 和 $c_6$
- 记变量集合为 $\{c_1,c_2,c_3,c_4,c_5,c_6\}$
    - $c_1,c_2,c_3,c_4$ 是电路的输入变量
    - $c_6$ 是电路的输出变量
    - $c_5$ 是电路产生的中间值变量

如果 $\{c_1,c_2,c_3,c_4,c_5,c_6\}$ 可以满足这个电路，即电路上的运算都是正确的，那么就称 $\{c_1,c_2,c_3,c_4,c_6\}$ 为一组有效的赋值（valid assignment），即电路的输入与输出集合，否则就是一组无效的、非法的赋值。

例如赋值集合 $\{1,2,2,2,1\}$ 是一组有效的赋值，我们总可以找到这样的集合 $\{c_5\}=\{4\}$ 来使得变量集合 $\{c_1,c_2,c_3,c_4,c_5,c_6\}=\{1,2,2,2,4,1\}$ 满足这个电路。

我们的目的就是通过将这样的算数电路转换为等价的 QAP 形式，来检验 $\{c_1,c_2,c_3,c_4,c_6\}$ 是否是一个合法的赋值。

## 二、从算数电路到 R1CS

R1CS 是三个向量组 $(\mathcal{V},\mathcal{W},\mathcal{Y})$ 所构成的一个序列, R1CS 的解是一个向量 $\mathcal{C}$。其中 $\mathcal{V}=\{v_{1},v_2,\dots,v_m\},\mathcal{W}=\{w_1,w_2,\dots,w_m\},\mathcal{Y}=\{y_{1},y_2,\dots,y_m\}, \mathcal{C}=\{c_{1},c_2,\dots,c_m\}$ 且 $(\mathcal{C}\circ v_i) * (\mathcal{C}\circ w_i) - (\mathcal{C}\circ y_i)=0, i\in\{1,\dots,m\}$，$\circ$ 代表着两个向量对应位置点乘后将所得乘积求和。  

比如$v_1=\{0,0,1,0,0,0\},w_1=\{0,0,0,1,0,0\},y_1=\{0,0,0,0,1,0\}, \mathcal{C}=\{1,2,2,2,4,1\}$ 就是一组满足的 R1CS。这表示着检查 $c_3*c_4-c_5$，即对乘法门 $r_5$ 的检查。若结果为 $0$，则赋值 $\mathcal{C}$ 通过了 $r_5$ 的检查，否则不通过。

<img src="https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/QAP%20%E8%AF%A6%E8%A7%A3/A%20satisdied%20R1CS.png" alt="A satisdied R1CS" style="zoom:67%;" />

为了得到可以表示上一节所示电路的 R1CS，我们对上述的电路作如下转换：

- 让 $\mathcal{V}$ 表示每个乘法门的左输入，如果 $c_k$ 是 $r_g$ 的左输入，那么 $v_k(r_g) = 1$，否则 $v_k(r_g) = 0$
- 让 $\mathcal{W}$ 表示每个乘法门的右输入，如果 $c_k$ 是 $r_g$ 的右输入，那么 $w_k(r_g) = 1$，否则 $w_k(r_g) = 0$
- 让 $\mathcal{Y}$ 表示每个乘法门的输出，如果 $c_k$ 是 $r_g$ 的输出，那么 $y_k(r_g) = 1$，否则 $y_k(r_g) = 0$

例如，乘法门 $r_6$ 的左输入是 $c_1$ 和 $c_2$，右输入是 $c_5$，输出是 $c_6$。那么其对应的集合中的表示为：$v_1(r_6)=v_2(r_6)=w_5(r_6)=y_6(r_6)=1$，其余值都是 0。

注意：这里输入的左右关系对应图中运算门的上下关系。我们对加法门做压缩处理，统一作为下一个乘法门的某个输入，不单独表示为一个 R1CS 约束（在本文的例子中，$c_1$ 和 $c_2$ 一起作为 $r_6$ 的左输入，而不单独表示加法门为一个新的约束条件）。[Vitalik 的博客](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649)中将加法作为了单独的约束条件来处理。

这样，我们就可以得到上述电路的 R1CS 形式：

![Example R1CS](https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/QAP%20%E8%AF%A6%E8%A7%A3/Example%20R1CS.png)

## 三、从 R1CS 到 QAP

我们需要将 $v_1,v_2,\cdots, y_5,y_6$ 转化为多项式的形式。对于这里的每一个多项式，我们知道其在对应乘法门 $r_5$ 和 $r_6$ 上的取值，那么我们可以使用拉格朗日插值法来确定这个多项式（最高次数为乘法门的数量减去 1）。不妨设 $r_5=5,r_6=7$，那么：

$$\begin{aligned}v_1&=\frac{x-r_6}{r_5-r_6}\times 0+\frac{x-r_5}{r_6-r_5}\times 1=6x+3 \\ 
v_2&=\frac{x-r_6}{r_5-r_6}\times 0+\frac{x-r_5}{r_6-r_5}\times 1=6x+3 \\
v_3&=\frac{x-r_6}{r_5-r_6}\times 1+\frac{x-r_5}{r_6-r_5}\times 0=5x+9 \\
v_4&=\frac{x-r_6}{r_5-r_6}\times 0+\frac{x-r_5}{r_6-r_5}\times 0=0 \\
v_5&=\frac{x-r_6}{r_5-r_6}\times 0+\frac{x-r_5}{r_6-r_5}\times 0=0 \\
v_6&=\frac{x-r_6}{r_5-r_6}\times 0+\frac{x-r_5}{r_6-r_5}\times 0=0 \end{aligned}$$

注意：所有的运算都在 $\mathbb{F}_{11}$ 上完成。

同样地，我们可以完成对向量组 $\mathcal{W}$ 和 $\mathcal{Y}$ 的构造，即 $\mathcal{V}=\{6x+3,6x+3,5x+9,0,0,0\}, \mathcal{W}=\{0,0,0,5x+9,6x+3,0\},$ 
$\mathcal{Y}=\{0,0,0,0,5x+9,6x+3\}$。

## 四、检验 QAP

现在可以通过使用对这些向量组点乘的方式，一次性检验所有的 R1CS 条件。

![Polynomial checking R1CS](https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/QAP%20%E8%AF%A6%E8%A7%A3/Polynomial%20checking%20R1CS.png)

因为点积检查其实就是多项式的一系列加法和乘法，结果也将是一个多项式。如果我们在乘法门对应的 $x$ 上对结果多项式做评估，其结果为 0 ；如果使用了其他的 $x$ 值，其结果必然是一个非零的值。但结果多项式本身并不一定需要为 0，在非乘法门对应的 $x$ 上做评估时，会有一些无法预料的结果。

对结果多项式正确性的检验，我们通常不直接使用对应乘法门的值，来计算其结果，而是通过一种整除性的检验方法。我们定义 $t(x)=\prod_g(x-r_g)$，是最小的在每个乘法门出都为 0 的多项式。这里有一个很直观的解释，如果 $P(x)=V(x)*W(x)-Y(x)=H*t(x)$，即 $P(x)$ 是 $t(x)$ 的倍数多项式($H$ 是一个整多项式)，那么 $P(x)$ 的零点包含 $t(x)$ 的零点，即包含所有乘法门的解，满足每一个 R1CS 约束。 

现在，则可以用如下多项式来检查 QAP 的正确性：

$$\begin{aligned}V(x)&=6x+5\\ W(x)&=x+8\\ Y(x)&=4x+6\\ P(x)&=6x^2+5x+1\\
t(x)&=x^2+10x+2\\ H&=P(x)/t(x)=6\end{aligned}$$

这里的 $H$ 是没有余数的，即对这个 QAP，以及其所表示的电路而言， $\{1,2,2,2,1\}$ 是一组合法的赋值。如果我们尝试另一组变量赋值 $\mathcal{C}'=\{1,1,1,1,2,3\}$，那么：

$$\begin{aligned}V(x)&=6x+4\\ W(x)&=6x+4\\ Y(x)&=6x+5\\ P(x)&=3x^2+9x\\
t(x)&=x^2+10x+2\\ H&=P(x)/t(x)=3+\frac{x+5}{x^2+10x+2}\end{aligned}$$

这里的 $H$ 是有余数的，不是整多项式，即 $=\{1,1,1,1,3\}$ 不是一组合法的赋值。

## 五、QAP 的定义

下面给出 QAP 的形式化定义，这里的定义参照 [GGPR13] 的方案：

一个在域 $\mathbb{F}$ 上定义的的 QAP $Q$ 包括一个目标多项式 $t(x)$ 和三个多项式集合 $\mathcal{V}=\{v_k(x):k\in\{0,\cdots,m\},\mathcal{W}=w_k(x):k\in\{0,\cdots,m\},\mathcal{Y}=\{y_k(x):k\in \{0,\cdots,m\}\}$ 都在 $\mathbb{F}(x)$ 中定义。

函数 $f$ 具有下标为 $1,\cdots,n$ 的 $n$ 个输入，以及下标为 $m-n'+1,\cdots,m$ 的 $n'$ 个 输出。如果一个 $Q$ 可以计算函数 $f$，那么下述陈述为真：

$c_1,\cdots,c_n,c_{m-n'+1},\cdots,c_m \in \mathbb{F}^{n+n'}$  是 $f$ 的一组有效的输入输出变量赋值当且仅当存在 $(c_{n+1},\cdots,c_{m-n'})\in \mathbb{F}^{m-n-n'}$ 满足 $t(x)$ 整除 $(v_0(x)+\sum_{k=1}^{m}c_k\cdot v_k(x))\cdot (w_0(x)+\sum_{k=1}^{m}c_k\cdot w_k(x))$
$- (y_0(x)+\sum_{k=1}^{m}c_k\cdot y_k(x))$。$Q$ 的大小（size）为 $m$，其次数（degree）为 $\deg(t(x))$。

这里假设所有的 $v_k(x),w_k(x),y_k(x)$ 的次数最高为 $\deg(t(x))-1$，因为它们都可以可以模 $t(x)$ 从而不影响最后整除性的检验。

注意：在本文的例子中 $v_0(x)=w_0(x)=y_0(x)=0$。
