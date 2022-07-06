---
title: zk-SNARK 协议详解
mathjax: true
date: 2022-07-03 20:26:53
categories: Cryptography
---

本篇文章是学习 zk-SNARK 协议时，所做的一些记录。

<!-- more -->

## zk-SNARK

Zero-Knowledge Succinct Non-interactive Argument of Knowledge（zk-SNARK）是零知识证明系统的一个实用的密码学实现。

- Zero-knowledge：零知识性，除了所做的陈述（statement）的真实性外，不透露任何其他与陈述有关的信息。
- Succinct：简洁性，证明（proof）应当是简短的，易存储的，容易验证的。
- Non-interactive：非交互性，在构建和存储证明的过程中，不需要类似挑战/问答的轮询。即该证明可以被计算和公开验证。
- Argument of Knowledge：Argument 指的是对给定陈述，我们可以给出证明，该证明需要满足完备性（Completeness），即为真的陈述有证明，为假的陈述没有证明。Knowledge 指的是证明这在对陈述制造一个可信的证明时，必须知道一个证据（witness），witness 在密码学中指的是可以验证给定陈述为真的证据。那么，在这里，argument of knowledge 指的是一种可以对指定证据制造可信证明的能力

注：在密码学专用名词中，argument 和 proof 是相对的概念，一般 argument 指的是计算安全的（computationally sound），即假定敌手的算力是多项式有限的；而 proof 则是统计安全的（statistical sound），在假定敌手是算力无限的情况下的安全。

从实用的角度上，一个 zk-SNARK 提供了一种对给定计算验证正确性，却不泄露所使用的值的证明方式。这里，给定的计算一般指 NP 问题。

## Polynomial

多项式是 zk-SNARK 构造的核心。

由算数基本定理（Fundamental Theorem of Algebra）保证，一个次数（degree）为 $d$ 的多项式，最多有 $d$ 个零点。亦即两个不同的多项式（次数相同）最多有 $d$ 个交点。

如果一个证明者声称他知道某个次数为 $d$ 的多项式，我们可以通过评估（evaluate）这个多项式在特定点的值来验证这个声明是否正确，考虑点的范围是 1 到 $10^{77}$，在特定点，其评估相同的概率最多为 $\frac{d}{10^{77}}$，是可忽略的（negligible）。因此我们可以构建一个简单的协议 1 来验证这个声明：

- 验证者（Verifier）随机选取 $x$，并在本地评估多项式 $P'$ 在 $x$ 点的值 $P'(x)$
- 验证者将 $x$ 发送给证明者（Prover），并要求其返回其所声称的多项式在 $x$ 上的评估
- 证明者计算 $x$ 在其所生成的多项式 $P$ 上的值，$P(x)$ 并发送给验证者
- 验证者检查 $P'(x)$ 与 $P(x)$ 是否相等，如果相等，那么证明者的陈述可以以一个极高的概率所证明。 

但在这个协议中，多项式 $P$ 的信息完全地公开了。我们不希望公开多项式的信息。考虑这样的场景，证明者声称自己知道一个次数为 3，有两个根分别为 1 和 2 的多项式 $P$。

不妨 $t(x)=(x-1)(x-2)$，即后文中的目标多项式（target polynomial）。由算数基本定保证，$P(x)=H(x)\cdot t(x)$，即这里必然存在这样的多项式 $H(x)$ 满足这个等式。也就是说，$P(x)$ 包含 $t(x)$，$P(x)$ 有 $t(x)$ 的全部零点。在不公开 $P(x)$，仅陈述 $t(x)$ 的条件下，一个很自然的证明方法就是找到 $H(x)=\frac{P(x)}{t(x)}$，其中除法没有余数，$H(x)$ 是一个整多项式。

那么我们可以更新协议 1 为协议 2:

- 验证者（Verifier）随机选取数 $r$，并计算（评估） $t=t(r)$，并将 $r$ 发送给证明者
- 证明者计算 $H(x)=\frac{P(x)}{t(x)}$ 并计算 $h=H(r)$ 和 $p=P(r)$，将 $h$ 和 $p$ 发送给验证者
- 验证者检查 $h\cdot t$ 与 $p$ 是否相等，如果想等，那么 $P(x)$ 确实包含 $t(x)$。

协议 2 可以在不泄漏多项式本身的情况下，完成对拥有某些性质的多项式的检查。但仍存在以下的问题：

- 证明者可以只发送使得 $h\cdot t=p$ 等式成立的 $h$ 和 $t$ 使得验证通过，而不需要真的知道多项式 $P$。
- 因为证明者知道随机的点 $r$，证明者可以构造任意的多项式 $P'$，使得 $P'(r)=t(r)\cdot H(r)$。
- 证明者可以通过构造更高次数的多项式来作弊。

## Homomorphic Encryption

从上一节中，$r$ 是明文传输的，这样证明者就可以根据 $r$ 来作弊。Homomorphic encryption（HE）可以将 $r$ 加密后传输，但不影响对 $r$ 进行相关的线性（加法、乘法）运算。一个比较经典的 HE 构造方法，就是在有限域内的指数运算，其逆运算是困难的，由 Diffie-Hellman 假设来保证。即加密方法为：$E(v)=g^v\pmod{n}$，$v$ 就是我们想要加密的变量。它可以支持加密后的变量做加法和乘法运算：

- 乘法：$(g^i)^j=g^{ij}\pmod{n}$
- 加法：$g^i\cdot g^j=g^{i+j}\pmod{n}$

同样地，对于一个多项式 $P(x)=x^3-3x^2+2x$。我们可以知道：

$$\begin{aligned}E(P(x))&=g^{x^3-3x^2+2x}\\
&=g^{x^3}\cdot g^{-3x^2} \cdot g^{2x}\\
&=(g^{x^3})^1\cdot (g^{x^2})^{-3} \cdot (g^{x})^2\\
&=(E(x^3))^1\cdot (E(x^2))^{-3}\cdot (E(x))^2\end{aligned}$$

因此我们只要知道 $E(x),E(x^2),E(x^3)$ 的值，就可以计算出加密后的多项式在 $x$ 点的取值。对于一个次数为 $d$ 的多项式，我们可以更新协议 2 为协议 3：

1. 验证者
    - 选取生成元 $g$ 
    - 选取随机数 $s$
    - 计算 $s^i,i\in[0,d]$ 的同态加密值，$E(s^i)=g^{s^i}$
    - 直接计算目标多项式 $t(x)$ 在 $s$ 点的取值，$t(s)$
    - 向证明者发送 $E(s^0),E(s^1),\cdots,E(s^d)$
2.  证明者
    - 计算多项式 $H(x)=\frac{P(x)}{t(x)}$
    - 使用接收到的加密值集合，和已知的 $P(x)$ 的系数集合 $c_0,\cdots,c_d$，计算 $g^p=E(P(s))=g^{P(s)}=g^{c_0s^0+c_1s^1+\cdots+c_ds^d}=\prod_{i=0}^d(E(s^i))^{c_i}$
    - 计算 $g^h=E(H(s))=g^{H(s)}$
    - 将 $g^p$ 和 $g^h$ 发送给验证者
3. 验证者
    - 检查 $g^p$ 是否等于 $(g^h)^{t(s)}$

## Knowledge of Exponent Assumption

在协议 3 中，证明者不知道 $s$ 的取值，因此其构造不合法的 $P$ 的难度增加了，但证明者依旧可以找到某些 $z_p$ 和 $z_h$ 使得 $z_p=(z_h)^{t(s)}$，从而代替原本要发送的 $g^p$ 和 $g^h$。例如，对某个随机数 $r$，证明者可选取 $z_h=g^r, z_p=(g^{t(s)})^r$，$g^{t(s)}$ 可以通过 $t(x)$ 和加密后的 $s$ 指数幂集合算出。因此验证者需要验证 $g^p$ 和 $g^h$ 是只使用给定的加密后的 $s$ 指数幂集合算出，而非证明者随机生成的值。

只用给定的加密后的$s$ 指数幂集合 $g^{s^0},\cdots,g^{s^d}$ 进行，其结果必然是 $(g^{s^i})^c$ 的形式 ，对某些随机的 $c$ 值。一种检验方法，即是如果对 $g^{s^i}$ 做了运算，则对一个 $g^{s^i}$ 的偏移（shifted）集合 $g^{\alpha s^i}$ 做同样的运算。 

这就是 [Dam91] 所介绍的 Knowledge of Exponent Assumption（KEA），其详细的过程是：

- Alice 拥有值 $a$，她要求 Bob 返回一个 $a$ 的指数次幂，唯一要求是该返回值必须由 $a$ 计算出，她所做的过程如下：
    - 随机选取值 $\alpha$
    - 计算 $a'=a^{\alpha}\pmod{n}$
    - 将 $(a,a')$ 发送给 Bob，要求 Bob 返回 $(b,b')$，$b'$ 和 $b$ 仍然满足 $\alpha$-偏移的关系
- 因为 Bob 除了暴力搜索的方法外无法从 $(a,a')$ 中获取 $\alpha$，所以 Bob 的构造只能用如下的方法：
    - 随机选取值 $c$
    - 计算 $b=(a)^c\pmod{n}$ 和 $b'=(a')^c\pmod{n}$
    - 返回 $(b,b')$
- Alice 检查 $b^{\alpha}$ 和 $b'$ 是否相等

上述流程保证了 Bob 确实对给定的数值（i.e. $a,a'$）使用了同样的指数运算（i.e. $c$）；以及 Bob 只能够使用 Alice 提供的数值去满足 $\alpha$-偏移；以及 Bob 确实知道（know）他所应用的指数 $c$，因为 $b$ 和 $b'$ 使用相同的指数 $c$ 计算而来；以及 Alice 并不知道 $c$ 且 Bob 不知道 $\alpha$。

将 KEA 添加到协议 3 中，可以限制（restrict）证明者只能使用验证者发送的 $g^{s^i}$ 集合，因此，可以构造协议 4 如下：

1. 验证者
    - 选取生成元 $g$ 
    - 随机选取 $s,\alpha$
    - 计算 $s^i,i\in[0,d]$ 的同态加密值，$E(s^i)=g^{s^i}$
    - 计算 $s^i$ 的 $\alpha$-偏移，即 $g^{\alpha s^i}$ 
    - 直接计算目标多项式 $t(x)$ 在 $s$ 点的取值，$t(s)$
    - 向证明者发送 $g^{s^0},g^{s^1},\cdots,g^{s^d},g^{\alpha s^0},g^{\alpha s^1},\cdots,g^{\alpha s^d}$
2.  证明者
    - 计算多项式 $H(x)=\frac{P(x)}{t(x)}$
    - 使用接收到的加密值集合，和已知的 $P(x)$ 的系数集合 $c_0,\cdots,c_d$，计算 $g^p=g^{P(s)}=g^{c_0s^0+c_1s^1+\cdots+c_ds^d}=\prod_{i=0}^d(g^{s^i})^{c_i}$
    - 使用接收到的加密值的偏移集合，和已知的 $P(x)$ 的系数集合 $c_0,\cdots,c_d$，计算 $g^{p'}=g^{\alpha P(s)}=g^{c_0\alpha s^0+c_1\alpha s^1+\cdots+c_d\alpha s^d}=\prod_{i=0}^d(g^{\alpha s^i})^{c_i}$
    - 计算 $g^h=E(H(s))=g^{H(s)}$
    - 将 $g^p,g^{p'}$ 和 $g^h$ 发送给验证者
3. 验证者
    - 检查 $g^p$ 是否等于 $(g^h)^{t(s)}$
    - 检查 $g^{p'}$ 是否等于 $(g^p)^{\alpha}$

## Non-interactivity

至此，我们已经构造出了一个交互式的协议，交互式意味着，证明只可以由最初的验证者来验证，而无法被别人所验证。这样的协议是无法被其他人所信任的，因为：

- 验证者可以和证明者串通，共享随机数 $s,\alpha$，这样来制作一个假的证明
- 证明者自身可以同样构造一个假的证明
- 验证者需要存储 $\alpha,t(s)$ 直到证明结束，在这期间，有泄露参数的风险

如果证明者只想是特定的验证者（designated verifier）验证证明的过程，且证明过程无法被其他人所复用，那么交互式的协议是有效的。但如果证明者试图想多个验证者证明，那么与每个验证者都需要执行一次协议，这样的效率是不高的。因此，我们还需要其他工具来使得协议所需的参数是可重用的、公开的、可信赖的、不容易滥用的。

首先要做的就是对  $\alpha,t(s)$ 进行加密，我们同样可以使用前文提到的 HE 来对这两个进行加密。但是 HE 并不支持对两个加密后的变量做乘法运算，即 $(g
^p)^{\alpha}$ 无法通过 $g^p$ 和 $g^{\alpha}$ 来计算。

在这里，要对两个加密后的值做乘法，需要引入一个新的密码学工具双线性映射（bilinear map），表示为 $e(g^*,g^*)$。给定两个加密后的值 （i.e. $g^a,g^b$），可以将其映射到它们乘积的结果（i.e. $e(g,g)^{ab}$）。

双线性映射的一个很重要的性质就是，它满足一种类似的指数幂的乘法性质：

$$e(g^a,g^b)=e(g^b,g^a)=e(g^{ab},g^1)=\cdots=e(g^i,g^j)\quad \mathrm{if}\quad ij=ab$$

以及一种类似的指数幂加法性质：

$$e(g^a,g^b)\cdot e(g^c,g^d)=e(g,g)^{ab+cd}$$

有了双线性映射，我们就可以将协议 4 中的验证者之行的第一阶段改为由可信方生成随机数 $\alpha, s$，加密后公布 （$g^{t(s)},g^\alpha,g^{s},g^{\alpha{s^i}};i\in[0,d]$）并销毁随机数。通常我们把这一串数值叫做 commen reference string（CRS）。

现在我们可以构造协议 5 如下：

- 设置（Setup）

    - 选取生成元 $g$ 和双线性映射 $e$
    - 选取随机数 $s, \alpha$

    - 计算 $g^\alpha, \{g^{s^i}\}_{i\in[d]},\{g^{\alpha s^i}\}_{i\in[0,d]}$ 

    - 计算 $g^{t(s)}$

    - 证明密钥（proving key）或评估密钥（evaluation key）：$\{g^{s^i}\}_{i\in[d]},\{g^{\alpha s^i}\}_{i\in[0,d]}$  
    - 验证密钥（verification key）：$(g^{t(s)},g^\alpha)$

- 证明（Proving）

    - 已知多项式 $P(x)$ 的系数集合 $\{c_i\}_{i\in[0,d]}$，$P(x)=\sum_{i=0}^d c_ix^i$
    - 计算多项式 $H(x)=\frac{P(x)}{t(x)}$

    - 使用 $\{g^{s^i}\}_{i\in[d]}$ 计算 $g^{P(s)}$ 和 $g^{H(s)}$ 

    - 使用 $\{g^{\alpha s^i}\}_{i\in[d]}$ 计算 $g^{\alpha P(s)}$

    - 生成证明 $\pi=(g^{P(s)},g^{H(s)},g^{\alpha P(s)})$

- 验证（Verification）

    - 收取到证明 $\pi=(g^p,g^h,g^{p'})$
    - 检查多项式限制（polynomial restriction），即是否使用了给定的多项式计算，$e(g^{p'},g)=e(g^p,g^\alpha)$
    - 检查多项式系数，即证明者是否知道 $P(x)$，亦即 $p$ 和 $t(s)\cdot h$ 的关系,$e(g^p,g)=e(g^{t(s)},g^h)$

注：如果双线性映射的结果是可以重用的，那么这个协议也是不安全的，因为证明者可以使用 $g^{p'}=e(g^p,g^\alpha)$ 来通过多项式限制检查：

$$e(e(g^{p},g^{\alpha}),g)=e(g^p,g^{\alpha})$$

### Multi-party Commen Reference String (optional)

但上文的 CRS 是由唯一的可信方生成的，为了达到不依赖于特定的第三方的目的，我们可以使用如下的方法来生成 CRS，假设有三个参与方 A，B，C 参与协议的构建：

- A 随机选取 $s_A,\alpha_A$，并公布她的 CRS：
    $$(g^{s^i_A},g^{\alpha_A},g^{\alpha_A s^i_A})$$
- B 随机选取 $s_B,\alpha_B$，通过对 A 的 CRS 计算自己的 CRS：
    $$((g^{s^i_A})^{s^i_B},(g^{\alpha_A})^{\alpha_B},(g^{\alpha_A s^i_A})^{\alpha_B s^i_B})=(g^{(s_As_B)^i},g^{\alpha_A \alpha_B},g^{\alpha_A \alpha_B (s_As_B)^i})$$
    并公布计算结果 AB 的 CRS：
    $$(g^{s^i_{AB}},g^{\alpha_{AB}},g^{\alpha_{AB} s^i_{AB}})$$
- C 随机选取 $s_C,\alpha_C$，通过对 AB 的 CRS 计算自己的 CRS：
    $$((g^{s^i_{AB}})^{s^i_C},(g^{\alpha_{AB}})^{\alpha_C},(g^{\alpha_{AB} s^i_{AB}})^{\alpha_C s^i_C})=(g^{(s_As_Bs_C)^i},g^{\alpha_A \alpha_B \alpha_C},g^{\alpha_A \alpha_B \alpha_C (s_As_Bs_C)^i})$$
    并公布计算结果 ABC 的 CRS：
    $$(g^{s^i_{ABC}},g^{\alpha_{ABC}},g^{\alpha_{ABC} s^i_{ABC}})$$

最后 $(g^{s^i_{ABC}},g^{\alpha_{ABC}},g^{\alpha_{ABC} s^i_{ABC}})$ 就可以作为三方共同的 CRS，并且三方各自都不知道随机参数 $s^i=s^i_A s^i_B s^i_C$ 和 $\alpha=\alpha_A \alpha_B \alpha_C$ 的值。但 A，B，C 其中的某一方可以使用任意的值来代替本该要计算的值来作弊，为了检验 CRS 的参与方每一方都使用了规定的数值计算（B 使用 A 的 CRS 计算 AB，C 使用 AB 的 CRS 来计算 ABC）我们可以将双线性映射用来检查 B 和 C 是否使用了正确的参数，比如 Bob 可以公开她加密后的参数：

$$(g^{s^i_B},g^{\alpha_B},g^{\alpha_B s^i_B})|_{i\in[d]}\quad \mathrm{where}\quad [d]=\{1,\cdots,d\}$$

这样就可以验证 Bob 公布的 CRS 是否是由 A 的 CRS 计算而来，对 $i\in[0,d]$ 检查：

- $e(g^{s^i_{AB}},g)=e(g^{s^i_A},g^{s^i_B})$
- $e(g^{\alpha_{AB}},g)=e(g^{\alpha_A},g^{\alpha_B})$
- $(g^{\alpha_{AB} s^i_{AB}},g)=e(g^{\alpha_A s^i_A},g^{\alpha_B s^i_B})$

同样地，C 也需要证明她的 CRS 是使用 AB 的 CRS 得到的。

## Quadratic Arithmetic Program

关于 QAP 的详细内容，见[上篇博客](https://www.neoyuan.xyz/QAP%E8%AF%A6%E8%A7%A3/)。QAP 最终将我们想要验证的计算问题转化为了多项式的整除性检查。

现在应用 QAP 到协议 4 上，即证明者声称其计算的 $F$ 函数结果是正确的。那么他需要将 $F$ 转化为 QAP $Q$，包括一个目标多项式 $t(x)$ 和三个多项式集合 $\mathcal{V}=\{v_i(x)\},\mathcal{W}=w_i(x)\},\mathcal{Y}=\{y_i(x)\},i\in [m]$。（在本文中设 $v_0(x)=w_0(x)=y_0(x)=0$，可以不做考虑）我们可以构造协议 6 如下：

- 设置（Settup）

    - 选取生成元 $g$ 和双线性映射 $e$
    - 选取随机数 $s, \alpha_v,\alpha_w,\alpha_y$
    - 根据公布的函数 $F$，计算其 QAP 形式，包括一个目标多项式 $t(x)$ 和三个多项式集合 $\mathcal{V}=\{v_i(x)\},\mathcal{W}=w_i(x)\},\mathcal{Y}=\{y_i(x)\},i\in [m]$，其大小（size）为 $m$，次数为 $d$

    - 计算 $\{g^{s^i}\}_{i\in[d]},\{g^{v_i(s)},g^{w_i(s)},g^{ y_i(s)}\}_{i\in[d]},\{g^{\alpha_v v_i(s)},g^{\alpha_w w_i(s)},g^{\alpha_y y_i(s)}\}_{i\in[d]}$
    - 计算 $g^t=g^{t(s)}$

    - 证明密钥：$(\{g^{s^i}\}_{i\in[d]},\{g^{v_i(s)},g^{w_i(s)},g^{ y_i(s)},g^{\alpha_v v_i(s)},g^{\alpha_w w_i(s)},g^{\alpha_y y_i(s)}\}_{i\in[m]})$
    - 验证密钥：$(g^1,g^{\alpha_v},g^{\alpha_w},g^{\alpha_y},g^{t})$

- 证明（Proving）

    - 根据公布的函数 $F$，计算其 QAP 形式，包括一个目标多项式 $t(x)$ 和三个多项式集合 $\mathcal{V}=\{v_i(x)\},\mathcal{W}=w_i(x)\},\mathcal{Y}=\{y_i(x)\},i\in [m]$，其大小（size）为 $m$，次数为 $d$
    - 根据输入 $u$，计算 $F(u)$，以及满足 QAP 的合法赋值 $\{c_i\}_{i\in[m]}$
    - 计算多项式 $$V(x)=\sum_{i=1}^mc_i\cdot v_i(x),W(x)=\sum_{i=1}^mc_i\cdot w_i(x),Y(x)=\sum_{i=1}^mc_i\cdot y_i(x)$$
    - 计算多项式 $H(x)=\frac{V(x)\cdot W(x)-Y(x)}{t(x)}$
    - 使用 $\{g^{s^i}\}_{i\in[d]}$ 计算 $g^{H(s)}$ 

    - 计算操作数多项式（operand polynomials） $$g^{V(s)}=\prod_{i=1}^m(g^{v_i(s)})^{c_i},g^{W(s)}=\prod_{i=1}^m(g^{w_i(s)})^{c_i},,g^{Y(s)}=\prod_{i=1}^m(g^{y_i(s)})^{c_i}$$

    - 计算偏移多项式 $$g^{\alpha_v V(s)}=\prod_{i=1}^m(g^{\alpha_v v_i(s)})^{c_i},g^{\alpha_w W(s)}=\prod_{i=1}^m(g^{\alpha_w w_i(s)})^{c_i},,g^{\alpha_y Y(s)}=\prod_{i=1}^m(g^{\alpha_y y_i(s)})^{c_i}$$

    - 生成证明 $\pi_F=(g^{V(s)},g^{W(s)},g^{Y(s)},g^{\alpha V(s)},g^{\alpha W(s)},g^{\alpha Y(s)},g^{H(s)} )$

- 验证（Verification）

    - 收取到证明 $\pi=(g^{V},g^{W},g^{Y},g^{ V'},g^{W'},g^{Y'},g^{H} )$
    - 检查多项式限制（polynomial restriction），即是否使用了给定的多项式计算，$e(g^{V'},g)=e(g^V,g^{\alpha_v}),e(g^{W'},g)=e(g^W,g^{\alpha_w}),e(g^{Y'},g)=e(g^Y,g^{\alpha_y})$
    - 检查多项式系数，即证明者是否使用了正确的系数来计算 $F(u)$，亦即 $V\cdot W-Y$ 和 $t\cdot h$ 的关系，$e(g^V,g^W)=e(g^{t},g^h)\cdot e(g^Y,g)$

注：如果使用同一个 $\alpha$ 来代替 $\alpha_v,\alpha_w,\alpha_y$，则无法保证证明者正确地构造了  $V,W,Y$，如证明者可以重用操作多项式，交换操作多项式的顺序，使用其他的操作数来伪造 $\alpha$-偏移。

## Variable Consistency Across Operands

对任意一个变量 $c_i$，它需要被计算到一个变量多项式（variable polynomial）中，即 $(g^{v_i(s)})^{c_i},(g^{w_i(s)})^{c_i},(g^{y_i(s)})^{c_i}$。因为对操作多项式使用的检验是分别进行的（三个 $\alpha$-偏移检验），但并没有保证每一个变量 $c_i$ 都被使用到了相对应的变量多项式上，所以我们可以添加一个 $\beta$-偏移来对每一个变量 $c_i$ 所对应的变量多项式同时检验，例如计算 $g^{v_i(s)+w_i(s)+y_i(s)}$ 及其偏移 $g^{\beta (v_i(s)+w_i(s)+y_i(s))}$ 。那么要对 $c_i$ 及其对应多项式的检验可以写为如下形式：$$(g^{V_i(s)})^{c_{V,i}},(g^{W_i(s)})^{c_{W,i}},(g^{Y_i(s)})^{c_{Y,i}},(g^{\beta(V_i(s)+W_i(s)+Y_i(s))})^{c_{\beta,i}}$$

如果每一个变量多项式都使用了正确的变量 $c_i$，即 $c_{V,i}=c_{W,i}=c_{Y,i}=c_{\beta,i},i\in[m]$，那么：$$e(g^{c_{V,i}\cdot V_i(s)}\cdot g^{c_{W,i}\cdot W_i(s)}\cdot g^{c_{Y,i}\cdot Y_i(s)},g^{\beta})=e(g^{c_{\beta,i}\cdot \beta(V_i(s)+W_i(s)+Y_i(s))},g)$$

但这样的检验并不安全，取 $V_i(x)=W_i(x),c_{\beta,i}=c_{Y,i},c_{V,i}=2c_{Y,i}-c_{W,i}$，即可通过验证。针对这个问题，用 $\beta_v,\beta_w,\beta_y$ 来代替 $\beta$。我们可以对协议 6 添加如下部分（协议 7a）：

- 设置
    - …… 
    - 随机选取 $\beta_v,\beta_w,\beta_y$ 
    - 计算变量一致性多项式（variable consistency polynomial）$\{g^{\beta_v v_i(s)+\beta_w w_i(s)+\beta_y y_i(s)}\}_{i\in[m]}$ 并添加到证明密钥中
    - 将 $(g^{\beta_v},g^{\beta_w},g^{\beta_y})$ 添加到验证密钥中
- 证明
    - ……
    - 将赋值变量分配给对应的变量一致性多项式：$\{g^{z_i(s)}\}_{i\in [m]}=\{(g^{\beta_v v_i(s)+\beta_w w_i(s)+\beta_y y_i(s)})^{c_i}\}_{i\in[m]}$
    - 计算 $g^{Z(s)}=\prod_{i=1}^m g^{z_i(s)}=g^{\beta_v V(s)+\beta_w W_i(s)+\beta_y Y_i(s)}$ 并添加到证明 $\pi_F$ 中
- 验证
    - ……
    - 检查操作多项式的一致性：$$e(g^V,g^{\beta_v})\cdot e(g^W,g^{\beta_w}) e(g^Y,g^{\beta_y}) = e(g^Z,g) \Leftrightarrow e(g,g)^{\beta_vV+\beta_wW+\beta_yY}=e(g,g)^Z$$

但因为证明者可以知道 $g^{\alpha_v}$ 和 $g^{\beta_v}$，他依然可以构造假的 $\mathcal{V,W,Y}$ 来通过 $\alpha$-偏移检验和一致性检验。这里需要引入一个随机变量 $\gamma$ 来使得用 $g^{\beta_v},g^{\beta_w},g^{\beta_y}$ 直接计算 $g^{Z(s)}$ 是冲突的，即用 $g^{\beta_v \gamma},g^{\beta_w \gamma},g^{\beta_y \gamma}$ 来代替。即构造如下的协议 7b：

- 设置
    - …… 
    - 随机选取 $\beta_v,\beta_w,\beta_y,\gamma$ 
    - 计算变量一致性多项式（variable consistency polynomial）$\{g^{\beta_v v_i(s)+\beta_w w_i(s)+\beta_y y_i(s)}\}_{i\in[m]}$ 并添加到证明密钥中
    - 将 $(g^{\beta_v\gamma},g^{\beta_w\gamma},g^{\beta_y\gamma},g^{\gamma})$ 添加到验证密钥中
- 证明
    - ……
    - 将赋值变量分配给对应的变量一致性多项式：$\{g^{z_i(s)}\}_{i\in [m]}=\{(g^{\beta_v v_i(s)+\beta_w w_i(s)+\beta_y y_i(s)})^{c_i}\}_{i\in[m]}$
    - 计算 $g^{Z(s)}=\prod_{i=1}^m g^{z_i(s)}=g^{\beta_v V(s)+\beta_w W_i(s)+\beta_y Y_i(s)}$ 并添加到证明 $\pi_F$ 中
- 验证
    - ……
    - 检查操作多项式的一致性：$$e(g^V,g^{\beta_v\gamma})\cdot e(g^W,g^{\beta_w\gamma}) e(g^Y,g^{\beta_y\gamma}) = e(g^Z,g^{\gamma})$$

但协议 8a 为验证密钥添加了 4 个元素，[Par+13] 则通过选取不同生成元的方式来减少了验证密钥数量，并增强了变量多项式的安全性，构造如下的协议 7c：

- 设置
    - …… 
    - 随机选取 $r_v,r_w,\beta,\gamma$，
    - 计算 $r_y=r_v\cdot r_w$ 和操作多项式生成元（operand generator）$g_v=g^{r_v},g_w=g^{r_w},g_y=g^{r_y}$  
    - 证明密钥：$(\{g^{s^i}\}_{i\in[d]},\{g_v^{v_i(s)},g_w^{w_i(s)},g_y^{ y_i(s)},g_v^{\alpha_v v_i(s)},g_w^{\alpha_w w_i(s)},g_y^{\alpha_y y_i(s)},g_v^{\beta v_i(s)}\cdot g_w^{\beta w_i(s)}\cdot g_y^{\beta y_i(s)}\}_{i\in[m]})$
    - 验证密钥：$(g^1,g^{\alpha_v},g^{\alpha_w},g^{\alpha_y},g_y^{t},g^{\beta \gamma},g^\gamma)$
- 证明
    - ……
    - 计算 $g^{Z(s)}=\prod_{i=1}^m (g_v^{\beta v_i(s)}\cdot g_w^{\beta w_i(s)}\cdot g_y^{\beta y_i(s)})^{c_i}$ 并添加到证明 $\pi_F$ 中
- 验证
    - ……
    - 检查多项式限制（polynomial restriction），即是否使用了给定的多项式计算：$$e(g_v^{V'},g)=e(g_v^V,g^{\alpha_v}),e(g_w^{W'},g)=e(g_w^W,g^{\alpha_w}),e(g_y^{Y'},g)=e(g_y^Y,g^{\alpha_y})$$
    - 检查多项式系数，即证明者是否使用了正确的系数来计算 $F(u)$，亦即 $V\cdot W-Y$ 和 $t\cdot h$ 的关系，$$e(g_v^V,g_w^W)=e(g_y^{t},g^h)\cdot e(g_y^Y,g) \Rightarrow e(g,g)^{r_vr_w VW}=e(g,g)^{r_vr_wth+r_vr_wy}$$
    - 检查操作多项式的一致性：$$e(g_v^V\cdot g_w^W\cdot g_y^Y,g^{\beta \gamma}) = e(g^Z,g^{\gamma})$$

那么到这一节为止，我们可以构造协议 8 （协议 6 + 协议 7c）如下

- 设置（Settup）

    - 选取生成元 $g$ 和双线性映射 $e$
    - 选取随机数 $s, \alpha_v,\alpha_w,\alpha_y,r_v,r_w,\beta,\gamma$
    - 计算 $r_y=r_v\cdot r_w$ 和操作多项式生成元（operand generator）$g_v=g^{r_v},g_w=g^{r_w},g_y=g^{r_y}$  
    - 根据公布的函数 $F$，计算其 QAP 形式，包括一个目标多项式 $t(x)$ 和三个多项式集合 $\mathcal{V}=\{v_i(x)\},\mathcal{W}=w_i(x)\},\mathcal{Y}=\{y_i(x)\},i\in [m]$，其大小（size）为 $m$，次数为 $d$

    - 证明密钥：$(\{g^{s^i}\}_{i\in[d]},\{g_v^{v_i(s)},g_w^{w_i(s)},g_y^{ y_i(s)},g_v^{\alpha_v v_i(s)},g_w^{\alpha_w w_i(s)},g_y^{\alpha_y y_i(s)},g_v^{\beta v_i(s)}\cdot g_w^{\beta w_i(s)}\cdot g_y^{\beta y_i(s)}\}_{i\in[m]})$
    - 验证密钥：$(g^1,g^{\alpha_v},g^{\alpha_w},g^{\alpha_y},g_y^{t},g^{\beta \gamma},g^\gamma)$

- 证明（Proving）

    - 根据公布的函数 $F$，计算其 QAP 形式，包括一个目标多项式 $t(x)$ 和三个多项式集合 $\mathcal{V}=\{v_i(x)\},\mathcal{W}=w_i(x)\},\mathcal{Y}=\{y_i(x)\},i\in [m]$，其大小（size）为 $m$，次数为 $d$
    - 根据输入 $u$，计算 $F(u)$，以及满足 QAP 的合法赋值 $\{c_i\}_{i\in[m]}$
    - 计算多项式 $$V(x)=\sum_{i=1}^mc_i\cdot v_i(x),W(x)=\sum_{i=1}^mc_i\cdot w_i(x),Y(x)=\sum_{i=1}^mc_i\cdot y_i(x)$$
    - 计算多项式 $H(x)=\frac{V(x)\cdot W(x)-Y(x)}{t(x)}$
    - 使用 $\{g^{s^i}\}_{i\in[d]}$ 计算 $g^{H(s)}$
    - 计算 $g^{Z(s)}=\prod_{i=1}^m (g_v^{\beta v_i(s)}\cdot g_w^{\beta w_i(s)}\cdot g_y^{\beta y_i(s)})^{c_i}$ 

    - 计算操作数多项式（operand polynomials） $$g_v^{V(s)}=\prod_{i=1}^m(g_v^{v_i(s)})^{c_i},g_w^{W(s)}=\prod_{i=1}^m(g_w^{w_i(s)})^{c_i},g_y^{Y(s)}=\prod_{i=1}^m(g_y^{y_i(s)})^{c_i}$$

    - 计算偏移多项式 $$g_v^{\alpha_v V(s)}=\prod_{i=1}^m(g_v^{\alpha_v v_i(s)})^{c_i},g_w^{\alpha_w W(s)}=\prod_{i=1}^m(g_w^{\alpha_w w_i(s)})^{c_i},g_y^{\alpha_y Y(s)}=\prod_{i=1}^m(g_y^{\alpha_y y_i(s)})^{c_i}$$

    - 生成证明 $\pi_F=(g_v^{V(s)},g_w^{W(s)},g_y^{Y(s)},g_v^{\alpha V(s)},g_w^{\alpha W(s)},g_y^{\alpha Y(s)},g^{H(s)},g^{Z(s)} )$

- 验证（Verification）

    - 收取到证明 $\pi=(g_v^{V},g_w^{W},g_y^{Y},g_v^{ V'},g_w^{W'},g_y^{Y'},g^{H},g^Z )$
    - 检查多项式限制（polynomial restriction），即是否使用了给定的多项式计算：$$e(g_v^{V'},g)=e(g_v^V,g^{\alpha_v}),e(g_w^{W'},g)=e(g_w^W,g^{\alpha_w}),e(g_y^{Y'},g)=e(g_y^Y,g^{\alpha_y})$$
    - 检查多项式系数，即证明者是否使用了正确的系数来计算 $F(u)$，亦即 $V\cdot W-Y$ 和 $t\cdot h$ 的关系，$$e(g_v^V,g_w^W)=e(g_y^{t},g^h)\cdot e(g_y^Y,g) \Rightarrow e(g,g)^{r_vr_w VW}=e(g,g)^{r_vr_wth+r_vr_wy}$$
    - 检查操作多项式的一致性：$$e(g_v^V\cdot g_w^W\cdot g_y^Y,g^{\beta \gamma}) = e(g^Z,g^{\gamma})$$

## Public Inputs and One

协议 8 可以证明，证明者掌握了 $F$ 的相关信息，并按照正确地流程计算了结果，但无法保证证明者是使用验证者所提供的输入去计算的。因此，可以让验证者指定证明者使用一些值进行计算。

考虑证明者要计算的加密值 $g_v^{V(s)},g_w^{W(s)},g_y^{Y(s)}$，由同态加密的性质可以保证，验证者可以对加密值加上其他的加密值，例如 $g_v^{L(s)}\cdot g_v^{l_v(s)}=g_v^{L(s)+l_v(s)}$。当然我们也可以从中减去一些加密值。我们将 $m$ 个变量多项式分为两部分，即 QAP 中所划分的输入/输出相关部分 $\{1,\cdots,N\}$，以及非输入/输出相关部分 $\{N+1,\cdots,m\}$。让证明者只使用非输入/输出部分进行计算，验证者使用输入/输出部分进行验证。这样，我们就可以构造协议 9：

- 设置（Settup）

    - 选取生成元 $g$ 和双线性映射 $e$
    - 选取随机数 $s, \alpha_v,\alpha_w,\alpha_y,r_v,r_w,\beta,\gamma$
    - 计算 $r_y=r_v\cdot r_w$ 和操作多项式生成元（operand generator）$g_v=g^{r_v},g_w=g^{r_w},g_y=g^{r_y}$  
    - 根据公布的函数 $F$，$F$ 有 $N$ 个输入输出变量，计算其 QAP 形式，包括一个目标多项式 $t(x)$ 和三个多项式集合 $\mathcal{V}=\{v_i(x)\},\mathcal{W}=w_i(x)\},\mathcal{Y}=\{y_i(x)\},i\in [m]$，其大小（size）为 $m$，次数为 $d$
    - 令 $I_{mid}=\{N+1,\cdots,m\}$

    - 证明密钥：$(\{g^{s^i}\}_{i\in[d]},\{g_v^{v_i(s)},g_w^{w_i(s)},g_y^{ y_i(s)},g_v^{\alpha_v v_i(s)},g_w^{\alpha_w w_i(s)},g_y^{\alpha_y y_i(s)},g_v^{\beta v_i(s)}\cdot g_w^{\beta w_i(s)}\cdot g_y^{\beta y_i(s)}\}_{i\in I_{mid}})$
    - 验证密钥：$(g^1,g^{\alpha_v},g^{\alpha_w},g^{\alpha_y},g_y^{t},g^{\beta \gamma},g^\gamma,\{g_v^{v_i(s)},g_w^{w_i(s)},g_y^{ y_i(s)}\}_{i\in[0,N]})$

- 证明（Proving）

    - 根据公布的函数 $F$，计算其 QAP 形式，包括一个目标多项式 $t(x)$ 和三个多项式集合 $\mathcal{V}=\{v_i(x)\},\mathcal{W}=w_i(x)\},\mathcal{Y}=\{y_i(x)\},i\in [m]$，其大小（size）为 $m$，次数为 $d$
    - 根据输入 $u$，计算 $F(u)$，以及满足 QAP 的合法赋值 $\{c_i\}_{i\in[m]}$
    - 计算多项式 $$V(x)=\sum_{i=1}^mc_i\cdot v_i(x),W(x)=\sum_{i=1}^mc_i\cdot w_i(x),Y(x)=\sum_{i=1}^mc_i\cdot y_i(x)$$
    - 计算多项式 $H(x)=\frac{V(x)\cdot W(x)-Y(x)}{t(x)}$
    - 使用 $\{g^{s^i}\}_{i\in[d]}$ 计算 $g^{H(s)}$
    - 计算 $g^{Z(s)}=\prod_{i\in I_{mid}} (g_v^{\beta v_i(s)}\cdot g_w^{\beta w_i(s)}\cdot g_y^{\beta y_i(s)})^{c_i}$ 

    - 计算操作数多项式（operand polynomials） $$g_v^{V(s)}=\prod_{i\in I_{mid}}(g_v^{v_i(s)})^{c_i},g_w^{W(s)}=\prod_{i\in I_{mid}}(g_w^{w_i(s)})^{c_i},g_y^{Y(s)}=\prod_{i\in I_{mid}}(g_y^{y_i(s)})^{c_i}$$

    - 计算偏移多项式 $$g_v^{\alpha_v V(s)}=\prod_{i\in I_{mid}}(g_v^{\alpha_v v_i(s)})^{c_i},g_w^{\alpha_w W(s)}=\prod_{i\in I_{mid}}(g_w^{\alpha_w w_i(s)})^{c_i},g_y^{\alpha_y Y(s)}=\prod_{i\in I_{mid}}(g_y^{\alpha_y y_i(s)})^{c_i}$$

    - 生成证明 $\pi_F=(g_v^{V(s)},g_w^{W(s)},g_y^{Y(s)},g_v^{\alpha V(s)},g_w^{\alpha W(s)},g_y^{\alpha Y(s)},g^{H(s)},g^{Z(s)} )$

- 验证（Verification）

    - 收取到证明 $\pi=(g_v^{V_{mid}},g_w^{W_{mid}},g_y^{Y_{mid}},g_v^{ V_{mid}'},g_w^{W_{mid}'},g_y^{Y_{mid}'},g^{H},g^Z )$
    - 计算 $g_v^{V}=g_v^{v_0(s)}\cdot \prod_{i\in [N]} (g_v^{v_{i}(s)})^{c_i} \cdot g_v^{V_{mid}}$，$g_w^{W}=g_w^{w_0(s)}\cdot \prod_{i\in [N]} (g_w^{w_{i}(s)})^{c_i} \cdot g_w^{W_{mid}} $，$g_y^{Y}=g_y^{y_0(s)}\cdot \prod_{i\in [N]} (g_y^{y_{i}(s)})^{c_i} \cdot g_y^{Y_{mid}} $
    - 检查多项式限制（polynomial restriction），即是否使用了给定的多项式计算：$$e(g_v^{V_{mid}'},g)=e(g_v^{V_{mid}},g^{\alpha_v}),e(g_w^{W'_{mid}},g)=e(g_w^{W_{mid}},g^{\alpha_w}),e(g_y^{Y'_{mid}},g)=e(g_y^{Y_{mid}},g^{\alpha_y})$$
    - 检查多项式系数，即证明者是否使用了正确的系数来计算 $F(u)$，亦即 $V\cdot W-Y$ 和 $t\cdot h$ 的关系，$$e(g_v^V,g_w^W)=e(g_y^{t},g^h)\cdot e(g_y^Y,g)$$
    - 检查操作多项式的一致性：$$e(g_v^{V_{mid}}\cdot g_w^{W_{mid}}\cdot g_y^{Y_{mid}},g^{\beta \gamma}) = e(g^Z,g^{\gamma})$$

注：${v_0(x)},{w_0(x)},{y_0(x)}$ 是在对某些函数做转换时，一些常数多项式的约束条件无法用 QAP 来表示而引入的，其值在转换的过程中会自然的计算出，而不需要另外的赋值。

当协议 9 中的 $\alpha$-偏移，使用的是同一个 $\alpha$ 时，即 $\alpha=\alpha_v=\alpha_w=\alpha_y$，此时协议 9 就是 [Par+13] 所构造的 Pinocchio 协议。

## Zero Knowledge

接下来，考虑将零知识的性质添加到协议上，即可构成一个完整的 zk-SNARK 协议。

在对多项式整除性进行检查时，我们可以使用一个 $\delta$-偏移，来隐藏原本的多项式，并使偏移后的多项式依然满足整除性要求，即：

$$\begin{aligned}\delta P(s)&=t(s)\cdot \delta H(s)\\
\delta(V(s)\cdot W(s)-Y(s))&=\delta t(s)H(s)\\
\Rightarrow \delta V(s)\cdot \delta W(s)-\delta^2 Y(s)&=t(s) \cdot \delta^2 H(s)\\
e(g,g)^{\delta^2 V(s)W(s)}&=e(g,g)^{\delta^2(t(s)H(s)+Y(s))}
\end{aligned}$$

和前述章节一样，使用相同的的随机数进偏移，也会造成一些信息的泄露。所以可以对不同的多项式进行不同的偏移，即

$$\delta_v V(s)\cdot \delta_w W(s) - \delta_v\delta_w Y(s)=t(s)\cdot (\delta_v\delta_w H(s))$$

偏移后的整除性依然满足。但如果证明者这样添加了偏移，验证者的输入也需要进行对应的偏移，那么双方就需要有交互的过程。这不符合非交互性的定义。

在这里，我们考虑使用加法来做随机偏移：

$$(V(s)+\delta_v t(s))\cdot (W(s)+\delta_w t(s))-(Y(s)+\delta_y t(s))=t(s)\cdot (\Delta+h(s))$$

在这里，为了满足整除性，取 $\Delta=\delta_w V(s)+\delta_v W(s)+\delta_v \delta_w t(s)- \delta_y$。我们仍然可以计算其同态值 $g^{\Delta}=(g^{V(s)})^{\delta_w}\cdot (g^{W(s)})^{\delta_v}\cdot (g^{t(s)})^{\delta_v \delta_w} g^{-\delta_y}, g^{V(s)+\delta_v t(s)}=g^{V(s)}\cdot (g^{t(s)})^{\delta_v}$，以及同样地计算 $g^{W(s)+\delta_w t(s)},g^{Y(s)+\delta_y t(s)}$。那么整除性的检验就变成了 $$V\cdot W-Y+t(s)(\delta_wV+\delta_vW+\delta_v \delta_w t(s)-\delta_y)=t(s)H+t(s)(\delta_wV+\delta_vW+\delta_v \delta_w t(s)-\delta_y)$$

为了使得多项式限制检验和多项式一致性检验同样使用了 $\delta$-偏移。我们需要在证明密钥中添加如下值：$$g_v^{t(s)},g_w^{t(s)},g_y^{t(s)},g_v^{\alpha_v t(s)},g_w^{\alpha_w t(s)},g_y^{\alpha_y t(s)},,g_v^{\beta t(s)},g_w^{\beta t(s)},g_y^{\beta t(s)}$$

这样我们就可以构造一个零知识的 Pinocchio 协议，亦即一个完整的 zk-SNARK 协议。

- 设置（Settup）

    - 选取生成元 $g$ 和双线性映射 $e$
    - 选取随机数 $s, \alpha_v,\alpha_w,\alpha_y,r_v,r_w,\beta,\gamma$
    - 计算 $r_y=r_v\cdot r_w$ 和操作多项式生成元（operand generator）$g_v=g^{r_v},g_w=g^{r_w},g_y=g^{r_y}$  
    - 根据公布的函数 $F$，$F$ 有 $N$ 个输入输出变量，计算其 QAP 形式，包括一个目标多项式 $t(x)$ 和三个多项式集合 $\mathcal{V}=\{v_i(x)\},\mathcal{W}=w_i(x)\},\mathcal{Y}=\{y_i(x)\},i\in [m]$，其大小（size）为 $m$，次数为 $d$
    - 令 $I_{mid}=\{N+1,\cdots,m\}$

    - 证明密钥：$(\{g^{s^i}\}_{i\in[d]},\{g_v^{v_i(s)},g_w^{w_i(s)},g_y^{ y_i(s)},g_v^{\alpha_v v_i(s)},g_w^{\alpha_w w_i(s)},g_y^{\alpha_y y_i(s)},g_v^{\beta v_i(s)}\cdot g_w^{\beta w_i(s)}\cdot g_y^{\beta y_i(s)}\}_{i\in I_{mid}},g_v^{t(s)},g_w^{t(s)},g_y^{t(s)},g_v^{\alpha_v t(s)},g_w^{\alpha_w t(s)},g_y^{\alpha_y t(s)},,g_v^{\beta t(s)},g_w^{\beta t(s)},g_y^{\beta t(s)})$
    - 验证密钥：$(g^1,g^{\alpha_v},g^{\alpha_w},g^{\alpha_y},g_y^{t},g^{\beta \gamma},g^\gamma,\{g_v^{v_i(s)},g_w^{w_i(s)},g_y^{ y_i(s)}\}_{i\in[0,N]})$

- 证明（Proving）

    - 根据公布的函数 $F$，计算其 QAP 形式，包括一个目标多项式 $t(x)$ 和三个多项式集合 $\mathcal{V}=\{v_i(x)\},\mathcal{W}=w_i(x)\},\mathcal{Y}=\{y_i(x)\},i\in [m]$，其大小（size）为 $m$，次数为 $d$
    - 根据输入 $u$，计算 $F(u)$，以及满足 QAP 的合法赋值 $\{c_i\}_{i\in[m]}$
    - 计算多项式 $$V(x)=\sum_{i=1}^mc_i\cdot v_i(x),W(x)=\sum_{i=1}^mc_i\cdot w_i(x),Y(x)=\sum_{i=1}^mc_i\cdot y_i(x)$$
    - 选取随机数 $\delta_v,\delta_w,\delta_y$
    - 计算多项式 $H(x)=\frac{V(x)\cdot W(x)-Y(x)}{t(x)}+\delta_wV(x)+\delta_vW(x)+\delta_v \delta_w t(x)-\delta_y$
    - 使用 $\{g^{s^i}\}_{i\in[d]}$ 计算 $g^{H(s)}$
    - 计算 $g^{Z(s)}=(g_v^{\beta t(s)})^{\delta_v}(g_w^{\beta t(s)})^{\delta_w}(g_y^{\beta t(s)})^{\delta_y}\cdot \prod_{i\in I_{mid}} (g_v^{\beta v_i(s)}\cdot g_w^{\beta w_i(s)}\cdot g_y^{\beta y_i(s)})^{c_i}$

    - 计算操作数多项式（operand polynomials） $$g_v^{V_{mid}(s)}=(g_v^{\beta t(s)})^{\delta_v}\cdot \prod_{i\in I_{mid}}(g_v^{v_i(s)})^{c_i},
        g_w^{W_{mid}(s)}=(g_w^{\beta t(s)})^{\delta_w}\cdot \prod_{i\in I_{mid}}(g_w^{w_i(s)})^{c_i},
        g_y^{Y_{mid}(s)}=(g_y^{\beta t(s)})^{\delta_y}\cdot \prod_{i\in I_{mid}}(g_y^{y_i(s)})^{c_i}$$

    - 计算偏移多项式 $$g_v^{V_{mid}'(s)}=(g_v^{\alpha_v t(s)})^{\delta_v}\cdot \prod_{i\in I_{mid}}(g_v^{\alpha_v v_i(s)})^{c_i},
        g_w^{W_{mid}'(s)}=(g_w^{\alpha_w t(s)})^{\delta_w}\cdot \prod_{i\in I_{mid}}(g_w^{\alpha_w w_i(s)})^{c_i},
        g_y^{Y_{mid}'(s)}=(g_y^{\alpha_y t(s)})^{\delta_y}\cdot \prod_{i\in I_{mid}}(g_y^{\alpha_y y_i(s)})^{c_i}$$

    - 生成证明 $\pi_F=(g_v^{V_{mid}(s)},g_w^{W_{mid}(s)},g_y^{Y_{mid}(s)},g_v^{V'_{mid}(s)},g_w^{W'_{mid}(s)},g_y^{Y'_{mid}(s)},g^{H(s)},g^{Z(s)})$

- 验证（Verification）

    - 收取到证明 $\pi=(g_v^{V_{mid}},g_w^{W_{mid}},g_y^{Y_{mid}},g_v^{ V_{mid}'},g_w^{W_{mid}'},g_y^{Y_{mid}'},g^{H},g^Z )$
    - 计算 $g_v^{V}=g_v^{v_0(s)}\cdot \prod_{i\in [N]} (g_v^{v_{i}(s)})^{c_i} \cdot g_v^{V_{mid}}$，$g_w^{W}=g_w^{w_0(s)}\cdot \prod_{i\in [N]} (g_w^{w_{i}(s)})^{c_i} \cdot g_w^{W_{mid}} $，$g_y^{Y}=g_y^{y_0(s)}\cdot \prod_{i\in [N]} (g_y^{y_{i}(s)})^{c_i} \cdot g_y^{Y_{mid}} $
    - 检查多项式限制（polynomial restriction），即是否使用了给定的多项式计算：$$e(g_v^{V_{mid}'},g)=e(g_v^{V_{mid}},g^{\alpha_v}),e(g_w^{W'_{mid}},g)=e(g_w^{W_{mid}},g^{\alpha_w}),e(g_y^{Y'_{mid}},g)=e(g_y^{Y_{mid}},g^{\alpha_y})$$
    - 检查多项式系数，即证明者是否使用了正确的系数来计算 $F(u)$，亦即 $V\cdot W-Y$ 和 $t\cdot h$ 的关系，$$e(g_v^V,g_w^W)=e(g_y^{t},g^h)\cdot e(g_y^Y,g)$$
    - 检查操作多项式的一致性：$$e(g_v^{V_{mid}}\cdot g_w^{W_{mid}}\cdot g_y^{Y_{mid}},g^{\beta \gamma}) = e(g^Z,g^{\gamma})$$

## Conclusion

最终我们所构造的协议有如下的性质：

- 简洁性：所需证明 $\pi$ 的大小是固定的，且只有 8 个元素
- 非交互性：一个证明 $\pi$ 可以被任意方所验证，而不需要和证明者进行交互
- 正确性：一个陈述可以以极高的概率证明成功
- 零知识性：从证明中提取出有效的信息是不可行（infeasible）的

## Reference

1. http://eprint.iacr.org/2013/279
2. https://arxiv.org/pdf/1906.07221
3. https://drive.google.com/file/d/0B-WxC9ydKhlRZG92dnJ0RmdWRkZKUXR5Q3FTd0pZMl9Tdnln/view
4. https://www.zeroknowledgeblog.com/index.php/the-pinocchio-protocol
5. http://people.seas.harvard.edu/~salil/research/SZKargs-focs.pdf
