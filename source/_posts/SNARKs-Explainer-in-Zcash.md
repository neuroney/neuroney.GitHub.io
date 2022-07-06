---
title: SNARKs Explainer in Zcash
date: 2022-03-25 23:27:58
categories: Cryptography
mathjax: true
---

本篇文章是阅读 Zcash 关于 SNARK 的 [技术文档](https://z.cash/technology/zksnarks)做的一些记录。

Succinct Non-interactive Argument of Knowledge（SNARK）是一种对特定信息的证明结构。

## 一、同态隐藏

同态隐藏（Homomorphic Hiding，HH）是构造 zk-SNARKs 最重要的一环。对数据 $x$ 的 HH 函数 $E(x)$ 需要满足如下性质： 

- 对**大多数** $x$ 而言，知道 $E(x)$ 计算出 $x$ 是困难的。
- 不同的输入会对应不同的输出，即如果 $x\neq y$，那么 $E(x)\neq E(y)$.
- 如果知道 $E(x), E(y)$, 可以计算出 $x,y$ 算术表达式的 HH。在加法的情况下，我们可以从 $E(x), E(y)$ 计算出 $E(x+y)$ 而不需要知道 $x,y$ 的值。

<!-- more -->

HH 对于零知识证明（Zero-knowledge proofs）十分有效。简单的做法，可以选取一个有限群（finite groups）来构造。

对于一个素数 $p$，选取群 $\mathbb{Z}_p^*=\{1,2,\cdots,p-1\}$，该群有如下性质：

1. 该群是循环群（cyclic group），即存在生成元（generator）$g$，使得 $\mathbb{Z}_p^*=\{g^0,g^1,$
    $\cdots,g^{n-2}\}$。
2. 基于离散对数问题（discrete logarithm problem）保证即使知道 $h$，很难找到 $a$ 使得 $g^a=h(\mod p)$。
3. $\forall a,b\in\{0,\cdots,p-2\},g^a \cdot g^b =g^{a+b(\mod p-1)}$。

在该群中，就可以定义一个简单的同态隐藏函数 $E(X)=g^x$。

HH 并不是密码学中正式定义的术语，在这里作为例子使用。它是相似概念 computationally hiding commitment 的弱化版本。HH 是一个确定性函数，但 commitment 是一个概率性算法。HH 只要求隐藏**大多数**输入，而 commitment 要求隐藏**全部**的输入。

## 二、多项式的盲评估

记 $\mathbb{F}_{p}=\{0,\cdots,p-1\}$，其中加法和乘法分别模 $p$。

### 多项式和线性组合

$\mathbb{F}_{p}$ 上的多项式 $P: P(X)=a_0+a_1\cdot X+a_2\cdot X^2+\cdots+a_d\cdot X^d$ 其中 $a_0,\cdots,a_d\in \mathbb{F}_p$。

在 $s, s\in \mathbb{F}_p$ 上对 $P$ 的评估：计算 $P(s)=a_0+a_1\cdot s+a_2\cdot s^2+\cdots+a_d\cdot s^d$ 。

如果我们知道 $P$ 的话，那么 $P(s)$ 其实是 $1,s,\cdots,s^d$ 的线性组合。在上一小节中，定义 $E(x)=g^x$，那么我们可以推导出 HH 的线性关系： $E(ax+by)=E(x)^a+E(y)^b$。

### 多项式的盲评估

假设 Alice 拥有多项式 $P$，度为 $d$。Bob 随机选择 $s, s\in \mathbb{F}_p$。

多项式的盲评估（Blind Evaluation of Polynomials）要求在通信过程中，Bob 知道 $E(P(s))$ 但不知道 $P$。Alice 计算 $E(P(s))$ 并发送给 Bob  但不知道 $s$。

通过 HH，构建如下的通信过程：

1. Bob 发送给 Alice，$E(1),E(s),\cdots,E(s^d)$。
2. Alice 通过接受到的元素，通过线性组合计算出 $E(P(s))$ 发送给 Bob。

这样，多项式的盲评估就完成了，既没有泄露 $s$ 也没有泄露 $P$。

在 Zcash 中，所构造的多项式太长，以及不同的信息所产生的多项式各不相同，所以传递多项式并不是 succinct 的做法。事实上隐藏性只保证了从 $E(s)$ 无法恢复出 $s$，在这里我们需要申明从 $E(1),E(s),\cdots,E(s^d)$ 中也无法恢复出 $s$，这里由 d-power Diffie-Hellman assumption 保证。

下面的章节会介绍盲评估是如何在 SNARKs 中使用的。粗略地讲，验证者（verifier）需要检查自己的多项式是否被证明者（prover）知道，基于 Schwartz-Zippel 定理 “不同多项式在大多数点的值都不同”。

## 三、知识系数测试和假设

上一节的内容只保证 Alice 能够计算 $E(P(s))$ 但无法强制要求 Alice 向 Bob 发送 $E(P(s))$。本节介绍的知识系数测试（Knowledge of Coefficient (KC) Test）是满足“强制性”方法的一个基本工具。

基于离散对数问题，构建一个阶（order）为 $p$，生成元（generator）为 $g$ 的群 $G$。对 $\alpha \in \mathbb{Z}_p^*$，定义 $\alpha$-pair 为 $(a,b)$ ，其中 $a,b\neq0, b=\alpha\cdot a,a,b\in G$。

*KC Test* 的流程如下： 

1. Bob 随机选择 $\alpha \in F_p^*$ 和 $a \in G$ 并计算 $b=\alpha\cdot a$。
2. Bob 发送 $(a,b)$ 给 Alice。 
3. Alice 必须回复 $(a',b')$ 同样也是 $\alpha-$pair 给 Bob。
4. Bob 检查 $(a',b')$ 是否是 $\alpha-$pair（$b'=\alpha\cdot a'$）。

在不知道 $\alpha$ 的情况下，Alice可以选择一些 $\gamma \in \mathbb{F}_p^*$ 构造 $(a',b')=(\gamma\cdot a,\gamma\cdot b)$，这样的 $(a',b')$ 是满足 $\alpha-$pair 的。

知识系数假设（ Knowledge of Coefficient Assumption，KCA）：如果 Alice 对 Bob 的挑战 $(a,b)$ 回复 $(a',b')$ 是 $\alpha$-pair 的概率是不可忽略的，那么 Alice 知道满足 $a'=\gamma \cdot a$ 的 $\gamma$。

## 四、可验证的多项式盲评估

同第二节，假设 Alice 拥有多项式 $P$，度为 $d$。Bob 随机选择 $s, s\in \mathbb{F}_p$。在这里，额外要求如果 Alice 发送了不是正确的 $E(P(s))$ 的值给 Bob，Bob 仍然接受的概率是可以忽略的。即可验证的多项式盲评估要求：

1. 盲性（Blindness）：Alice 不知道 $s$ 且 Bob 不知道 $P$。
2. 可验证性（Verifiability）：Alice 发送了不是正确的 $E(P(s))$ 的值给 Bob，Bob 仍然接受的概率是可以忽略的。

### 扩展知识系数假设

扩展知识系数假设（extended KCA）可以如下构造：Bob 发送给 Alice 多组 $\alpha$-pair $(a_1,b_1),\cdots,(a_d,b_d)$，Alice 仍需要在不知道 $\alpha$ 的情况下回复某个 $\alpha$-pair $(a',b')$。显然 Alice 可以用上一节的方法，选择一些 $\gamma \in \mathbb{F}_p^*$ 构造 $(a',b')=(\gamma\cdot a_i,\gamma\cdot b_i),i\in [1,d]$。Alice 也可以选择使用其他的方法，比如同时选择几个 Bob 发送的 $\alpha$-pair，用它们的线性组合来构造新的 $\alpha$-pair。比如：Alice 选择 $c_1,c_2\in \mathbb{F}_p$，并计算 $(a',b')=(c_1\cdot a_1+$
$c_2\cdot a_2,c_1 \cdot b_1 + c_2 \cdot b_2)$。可以验证 $b'=(c_1\cdot b_1+c_2\cdot b_2)=\alpha (c_1\cdot a_1+c_2\cdot a_2)=$
$\alpha \cdot a'$，从而 $(a',b')$ 是 $\alpha$-pair。

一般来说，Alice 可以选择 $c_1,\cdots,c_d\in \mathbb{F}_p$，并计算 $(a',b')$
$=(\sum_{i=1}^d c_i\cdot a_i,\sum_{i=1}^d c_i \cdot b_i)$，可以验证 $(a',b')$ 是 $\alpha$-pair。

Extended KCA 表示，这是 Alice 唯一可以生成 $\alpha$-pair 的方法。正式地讲，假设群 $G$ 的阶为 $p$，生成元为 $g$。$d$ 阶知识系数假设（d-power Knowledge of Coefficient Assumption，d-KCA）表述如下：

**d-KCA**：假设 Bob 随机选取 $\alpha\in \mathbb{F}_p^*,s\in \mathbb{F}_p$，并发送 $\alpha-$pairs $(g,\alpha\cdot g),(s\cdot g,\alpha s\cdot g),$
$\cdots,(s^d\cdot g,\alpha s^d\cdot g)$ 给 Alice。如果 Alice 回复 $\alpha-$pair $(a',b')$，那么 Alice 以不可忽略的概率知道 $c_0,\cdots,c_d\in\mathbb{F}_p$ 使得 $\sum_{i=0}^d c_i s^i \cdot g=a'$。

### 可验证的多项式盲评估协议

假设 HH 函数 $E(x)=x\cdot g$，$g$ 是上述群 $G$ 的生成元。 简单地说，对这样的 $E$ 的可验证多项式盲评估协议可构造为：
1. Bob 随机选取 $\alpha\in \mathbb{F}_p^*$，并发送 $\alpha-$pairs $(g,\alpha\cdot g),(s\cdot g,\alpha s\cdot g),$
$\cdots,(s^d\cdot g,\alpha s^d\cdot g)$ 给 Alice。
2. Alice 用接收到的值计算 $a=P(s)\cdot g, b=\alpha P(s)\cdot g$ 并发送给 Bob。
3. Bob 检查 $(a,b)$ 是否是 $\alpha$-pair，若是则接受。

首先，因为 $P(s)\cdot g$ 是一种 $g,s\cdot g, \cdots, s^d\cdot g$ 的线性组合关系，$\alpha P(s)\cdot g$ 是 $\alpha \cdot g,\alpha  s\cdot g, \cdots, \alpha s^d\cdot g$ 的线性组合关系，所以如果 Alice 知道多项式 $P$，那么她一定可以借由 HH 的性质计算出多项式 $P$ 的结果。

其次，根据 d-KCA，如果 Alice 返回了正确的 $\alpha$-pair，那么几乎可以确信 Alice 知道 $c_0,\cdots,c_d\in\mathbb{F}_p$ 使得 $\sum_{i=0}^d c_i s^i \cdot g=a'$。此时，$a=P(s)\cdot g$，Alice 知道多项式 $P(X)=\sum_{i=0}^d c_i X^i $。

##  五、从计算到多项式

本节将介绍 *Quadratic Arithmetic Program* (QAP)，当前最实用的 zk-SNARK 构造基础。

假设 Alice 想要向 Bob 证明她知道这样的 $c_1,c_2,c_3\in \mathbb{F}_p$ 使得 $(c_1\cdot c_2)\cdot (c_1+c_3)=7$。那么，第一步，就是将 $c_1,c_2,c_3$ 的计算表达式转化成等价的算数电路。

### 算数电路

一个算数电路包括加法门和乘法门，以及连接线组成。本节所用的例子如下：

<img src="https://pico-1258741719.cos.ap-shanghai.myqcloud.com/blog/SNARKs%20Explainer%20in%20Zcash/CircuitDrawing.png" alt="img" style="zoom:60%;" />

最底层的连接线（$w_1,w_2,w_3$）就是输入线（input wires），最顶层的线（$w_5$）就是输出线（output wire），转化成 QAP 的流程如下：

- 当相同的输出线，输入不同的门，它们记为同样的输入线，如 $w_1$。
- 乘法门有两个输入线，左输入和右输入。
- 加法门并不计入输入线，只计入乘法门的输入，如 $w_1, w_3$ 都是 $g_2$ 的右输入。

电路的一个合法赋值（legal assignment）就是电路中线和门的关系是相对应的。对上图的电路而言，一个合法赋值就是 $(c_1,\cdots,c_5)$ 其中 $c_4=c_1\cdot c_2, c_5=c_4 \cdot (c_1+c_3)$。Alice 想要证明的是，她拥有这样的一组合法赋值 $(c_1,\cdots,c_5)$ 且 $c_5=7$。

### 到 QAP 的规约

将每个乘法门与域中元素对应，$g_1$ 对应 $1\in \mathbb{F}_p$，$g_2$ 对应 $2\in \mathbb{F}_p$。将 $\{1,2\}$ 叫做目标点（target points），同时记左线多项式（left wire polynomials）为 $L_1,\cdots,L_5$，右线多项式（right wire polynomials）为 $R_1,\cdots,R_5$，输出线多项式（output wire polynomials）为 $O_1,\cdots,O_5$。

$w_1,w_2,w_4$ 分别是 $g_1$ 的左输入线、右输入线和输出线。记 $L_1=R_2=O_4=2-X$，多项式 $2-X$ 在点 $1$ 处取值为 $1$ 和 $g_1$ 相关联，在点 $2$ 处取值为 $0$ 和 $g_2$ 相关联。$w_1,w_3$ 是 $g_2$ 的右输入线，$w_4$ 是 $g_2$ 的左输入线，$w_5$ 是 $g_2$ 的输出线。记 $L_4=R_1=R_3=O_5=X-1$，多项式 $X-1$ 在点 $2$ 处取值为 $1$ 和 $g_2$ 相关联，在点 $1$ 处取值为 $0$ 和 $g_1$ 相关联。其余的多项式全部为零多项式。

定义**和多项式**（sum polynomials）为
$$L:=\sum_{i=1}^5 c_i \cdot L_i;R:=\sum_{i=1}^5 c_i \cdot R_i;O:=\sum_{i=1}^5 c_i \cdot O_i.$$

同时 $P:=L\cdot R - O$。 $(c_1,\cdots,c_5)$ 是一组合法赋值， 当且仅当 $P$ 在所有目标点的值都为零。例如，$P(1)=L(1)\cdot R(1)-O(1)=c_1\cdot c_2 - c_4; P(2)=c_4\cdot (c_1+c_3) - c_5$。

基于代数事实（algebraic fact）：对给定的多项式 $P$ 和 $a\in \mathbb{F}_p$，$P(a)=0 $
$\Leftrightarrow P=H\cdot (X-a)$，$H$ 也是一个多项式。

记目标多项式（target polynomial） $T(X):=(X-1)\cdot (X-2)$，$T$ 整除 $P$ 当且仅当 $(c_1,\cdots,c_5)$ 是一组合法赋值。

现在，我们可以给出 QAP 的定义如下：

一个度数为 $d$，大小为 $m$ 的QAP $Q$，包括多项式 $L_1,\cdots,L_m,R_1,\cdots,R_m,O_1,\cdots,O_m$ 和一个度数为 $d$ 的目标多项式 $T$。一组满足 $Q$ 的合法赋值 $(c_1,\cdots,c_5)$ 当且仅当定义 $L:=\sum_{i=1}^5 c_i \cdot L_i;R:=\sum_{i=1}^5 c_i \cdot R_i;O:=\sum_{i=1}^5 c_i \cdot O_i;P:=L\cdot R - O$, 且 $T$ 整除 $P$。

### Pinocchio 协议

这一节介绍 Pinocchio 协议， Alice 如何通过更简短的证明向 Bob 证实自己拥有满足 QAP 的一个合法赋值。

从上一节可知，Alice 拥有一个合法赋值，就意味着对于多项式 $L,R,O,P$，存在着一个多项式 $H$，使得 $P=H\cdot T$。即 $\forall s\in \mathbb{F}_p, P(s)=H(s)\cdot T(s)$。由 Schwartz-Zippel 定理可知，如果 Alice 没有这样的赋值，那么找到 $H$ 使得  $P=H\cdot T$ 的概率是可以忽略的。

可以构造如下的协议去测试 Alice 是否拥有一个合法的赋值：

1. Alice 选择度数最高为 $d$ 的多项式 $L,R,O,H$。
2. Bob 随机选择点 $s\in \mathbb{F}_p$ 并计算 $E(T(s))$。
3. Alice 发送给 Bob 这些多项式在 $s$ 点的评估值的 HH 值，$E(L(s)),E(R(s)),$
$E(O(s)),E(H(s))$。
4. Bob 检查 $E(L(s)\cdot R(s)- O(s))$ 是否等于 $E(T(s)\cdot H(s))$。

如果 Alice 不知道一个合法的赋值，那么协议运行多次，Bob 可以以极高的概率拒绝 Alice 的回复。关于 Alice 如何在不知道 $s$ 的情况下，计算这些多项式的值，在可验证的盲评估协议中已经解决。

## 六、确保 Alice 使用了正确的多项式

如果 Alice 没有一组合法的赋值，并不代表她不能找到满足 $L\cdot R - O=H\cdot T$ 的度最高为 $d$ 的多项式 $L,R,O,H$。这只是说明，Alice 不能找到从一组赋值中产生的对应的多项式。上一节保证了 Alice 使用的多项式 $L,R,O$ 的度数是正确的，但不能说明它们式由合法赋值产生的。

这里用一个不准确的方法来结局，定义多项式 $F:=L+X^{d+1}\cdot R+X^{2(d+1)}\cdot O$，其中 $X^{d+1},X^{2(d+1)}$ 是保证  $L,R,O$ 的系数不会混淆，$1,X,\cdots,X^d$ 是 $L$ 的系数，$X^{d+1},\cdots,X^{2d+1}$ 是 $R$ 的系数，$X^{2(d+1)},\cdots,X^{3d+2}$ 是 $O$ 的系数。

同样地，可以将 QAP 中的多项式合并起来，为每一个 $i\in\{1,\cdots,m\}$ 定义一个多项式 $F_i=L_i+X^{d+1}\cdot R_i+X^{2(d+1)}\cdot O_i$。同样地，对一组赋值 $(c_1,\cdots,c_m)$ 而言，$F=\sum_{i=1}^m c_i\cdot F_i$，我们也会有 $L=\sum_{i=1}^5 c_i \cdot L_i;R=\sum_{i=1}^5 c_i \cdot R_i;O=\sum_{i=1}^5 c_i \cdot O_i$。也就是说如果 $F$ 是 $F_i$ 的线性组合，那么 $L,R,O$ 是由一组合法赋值产生的。

Bob 将需要 Alice 对其证明 $F$ 是 $F_i$ 的线性组合。Bob 随机选择 $\beta \in \mathbb{F}_p^*$，并发送 $E(\beta \cdot F_1(s)),\cdots, E(\beta \cdot F_m(s))$ 给 Alice。Alice 需要回复 $E(\beta \cdot F(s))$。 如果 Alice 回复了正确的值，那么 d-KCA 保证了 Alice 知道如何将 $F_i$ 线性组合成 $F$。

### 零知识部分——隐藏赋值

在 zk-SNARK 中，Alice 想要隐藏关于自己拥有的合法赋值的信息，但 $E(L(s)),E(R(s)),$
$E(O(s)),E(H(s))$ 仍会泄漏一些关于赋值信息。比如，对给定的赋值 $(c_1',\cdots,c_m')$，Bob 可以计算出相关的 $L',R',O',H'$ 和 $E(L'(s)),E(R'(s)),E(O'(s)),E(H'(s))$，如果这些不同于 Alice 的回复，那么 Bob 知道 $(c_1',\cdots,c_m')$ 不是 Alice 所拥有的赋值。

因此，Alice 通过为每个多项式增加一个随机 T 偏移（random T shift）来隐藏真实的多项式，即 Alice 随机选择 $\delta_1,\delta_2,\delta_3 \in \mathbb{F}_p^*$，记 $L_z:=L+\delta_1 \cdot T,R_z:=R+\delta_2 \cdot T,$
$O_z:=O+\delta_3 \cdot T, H_z=H+L\cdot \delta_2 + R \cdot \delta_1+T\cdot \delta_1 \delta_2 - \delta_3$，可以验证 $L_z \cdot R_z - O_z$
$ = T\cdot H_z$。

## 七、椭圆曲线的配对

本节将介绍椭圆曲线，一个支持乘法的 HH，也同样足够将交互式的系统转化为非交互式的系统。

### 椭圆曲线和它们的配对

假定 $p$ 是一个大于 3 的素数，找一些 $u,v\in \mathbb{F}_p$ 使得 $4u^3+27u^2\neq 0$。然后考察如下方程 $$Y^2=X^3+u\cdot X + v$$

一个椭圆曲线 $\mathcal{C}$，就是满足上述方程的点 $(x,y)$ 的集合。用满足这个方程的点的集合，和一个特殊点 $\mathcal{O}$ 构造一个新的群，点 $\mathcal{O}$ 也叫做无限点（point at infinity）、零元（zero）、单位元（identity element）。

群中的加法由除子类群（divisor class group）派生，在除子类群中，在任意线上，点的加和必须为零，即 $\mathcal{O}$。

在椭圆曲线中，由 $X=c$ 所确定的竖直线与椭圆曲线在点 $P=(x_1,y_1)$ 处相交，由于椭圆曲线关于横轴的对成型，那么点 $Q=(x_1,-y_1)$ 也一定是交点。因为， $Y$ 的阶数为 2，所以这也是仅有的两个交点。于是 $P+Q=\mathcal{O}$，$Q$ 是 $P$ 在群中的相反数。

如果 $P=(x_1,y_1),Q=(x_2,y_2),x_1\neq x_2$，那么可以做一条直线 PQ 与椭圆曲线相交，因此 $x$ 的阶数为 3，因此必然还有一个交点 $R$，且 $P+Q+R=\mathcal{O}$，即 $-R=P+Q$，$R$ 可以用翻转第二个左边的方式由 $-R$ 得到。

因此，我们得到了群加法的运算规则，这个群通常被记为 $\mathcal{C}(\mathbb{F}_p)$，下文中，我们将用 $G_1$ 来表示。$G_1$ 中一共有素数 $r,r\neq p$ 个元素，生成元为 $g\neq \mathcal{O}$。

使得 $r$ 整除 $p^k -1$ 的最小整数 $k$ 成为曲线的嵌入阶（embedding degree）。据推测，当 $k$ 不太小，比如至少是 6 的时候，那么 $G_1$ 中的离散对数问题是非常困难的。

乘法群 $\mathbb{F}_p^k$ 包含一个 $r$ 阶子群 $G_T$，坐标不仅在 $\mathbb{F}_p$ 上，还能可能在 $\mathbb{F}_p^k$ 上的点的集合，和 $\mathcal{O}$ 也形成一个群。这样，除开 $G_1$， $\mathcal{C}(\mathbb{F}^k_p)$ 还包含一个（实际上 $r-1$ 个）额外的 $r$ 阶子群 $G_2$。

固定生成元 $g\in G_1, h\in G_2$，我们可以获得一个高效的映射——泰特缩减配对（Tate reduced pairing），它将一对 $G_1$ 和 $G_2$ 中的元素映射到 $G_T$ 中的一个元素，使得：

1. $Tate(g,h)=\mathbf{g}$，$\mathbf{g}$ 是 $G_T$ 的生成元。
1. 对给定的 $a,b\in \mathbb{F}_p^k, Tate(a\cdot g,b\cdot h)=\mathbf{g}^{ab}$。

Tate 的定义不在本文的考虑范围内，下面给出一个 Tate 的定义概要：

对于给定的 $a\in \mathbb{F}_p$，多项式 $(X-a)^r$ 在 $a$ 处有一个重数为 $r$ 的零点，并且再没有其他零点。对于点 $P\in G_1$，除数使我们证明存在一个从曲线到 $\mathbb{F}_p$ 的映射 $f_P$。从某种意义上，除了在 $P$ 处的 $r$ 重零点，没有其他零点，$Tate(P,Q):= f_P(Q)^{(p^k-1)/r}$。

定义 $E_1(x):=x \cdot g,E_2(x):=x \cdot h,E_3(x):=x \cdot \mathbb{g}$，我们得到一个弱化版的支持加法和乘法的 HH。

### CRS 模型中的非交互式证明

从直觉上讲，典型的非交互式证明应该是这样的。为了证明一个声明（claim），证明这广播简单的信息给所有人，使得任何收到这条信息的人都可以确信证明者的声明。

一个稍微宽松点的非交互式证明允许引入一个公共字符串（common reference string， CRS）。在 CRS 模型中，在证明过程开始之前，有一个设置阶段（setup phase），设置阶段中随机产生一个字符串，并广播给所有人。这个字符串就称为 CRS。由于 CRS 是随机产生的，当事人无法用它来构造假的证明。

下面将介绍如何在 CRS 模型下，将可验证的盲评估协议转化为非交互式证明系统。

### 一个非交互式评估协议

非交互式的盲评估协议，很显然将 Bob 发送的第一条信息当作 CRS。

**设置阶段（Setup）**：随机选择 $\alpha \in \mathbf{F}_r^*,s\in  \mathbf{F}_r$，并公开 CRS
$(E_1(1),E_1(s)\cdots,E_1(s^d),$
$E_2(\alpha),E_2(\alpha s),\cdots,E_2(\alpha s^d),)$。

**证明阶段（Proof）**：Alice 使用 CRS 的元素和 HH 的线性组合性质计算 $a=E_1(P(s)), $
$b=E_2(\alpha P(S))$。

**验证阶段（Verification）**：固定 $x,y\in \mathbf{F}_r$ 使得 $a=E_1(x),b=E_2(y)$。Bob 计算 $E(\alpha x)=Tate(E_1(x),E_2(\alpha)),E(y)=Tate(E_1(1),E_2(y))$ 并检查二者是否相等。若相等，则 $y=\alpha x$。

如第四节所述，Alice 若想构造通过验证的 $(a,b)$，则必须知道 $d$ 阶多项式 $P$。和交互式证明最主要的不同就是，Bob 不再需要 $\alpha$ 去验证，Bob 可以使用 CRS 中的元素 $E_1(x),E_2(\alpha)$ 来计算 $E(\alpha x)$。

## Reference

1. https://z.cash/technology/zksnarks
