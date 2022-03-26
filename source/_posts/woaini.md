---
title: woaini
date: 2022-03-25 23:27:58
categories: 我爱你
mathjax: true
---

### Homomorphic Hiding (HH) 

A homomorphic hiding function $E(x)$ has the following properties:

- For most $x$, given $E(x)$ is hard to find $x$
- Different inputs lead to different outputs – so if $x\neq y$, then $E(x)\neq E(y)$.
- If someone knows $E(x)$ and $E(y)$, they can generate the HH of arithmetic expressions in $x$ and $y$. For example, they can compute $E(x+y)$ from $E(x)$ and $E(y)$.

HH is useful for Zero-knowledge proofs.

Construction for HH: finite groups $\mathbb{Z}_p^*$, discrete logarithm problem 

$\mathbb{F}_p$, the set $\{0,1,\cdots,p-1\}$ and $+,\times$ are done $\mod p$.

HH is used in snarks to hide verifier challenges rather than prover secrets.

<!-- more -->

### Blind Evaluation of Polynomials

The polynomial $P: P(X)=a_0+a_1\cdot X+a_2\cdot X^2+\cdots+a_d\cdot X^d$ for some $a_0,\cdots,a_d\in \mathbb{F}_p$.

Evaluate $P$ at a point $s\in \mathbb{F}_p$: computing the resultant sum $P(s)=a_0+a_1\cdot s+a_2\cdot s^2+\cdots+a_d\cdot s^d$ .

HH supports this linear combinations by $E(ax+by)=E(x)^a+E(y)^b$.

Alice wants Bob to learn $E(P(s))$ without learning $P$. And Bob doesn't want Alice  to learn $s$. 

Thus, Bob send to Alice the hidings $E(1),E(s),\cdots,E(s^d)$. Alice computes $E(P(s))$ by linear combination and sends it to Bob. (d-power Diffie-Hellman assumption)

### The Knowledge of Coefficient Test and Assumption

For $\alpha \in \mathbb{F}_p^*$, let us call a pair $(a,b)$ in $G$ an $\alpha-$pair if $a,b\neq0$ and $b=\alpha\cdot a$. 

The *KC Test*  proceeds as follows: 

1. Bob chooses random $\alpha \in F_p^*$ and $a \in G$. He computes $b=\alpha\cdot a$.
2. He sends to Alice the "challenge" pair $(a,b)$. ($\alpha-$pair) 
3. Alice must now respond with a different pair $(a',b')$ that is also an $\alpha-$pair.
4. Bob accepts Alice's response only if $(a',b')$ is indeed an $\alpha-$pair. (As he knows $\alpha$ he can check if $b'=\alpha\cdot a'$.)

Without knowing $\alpha$, Alice can send $(a',b')=(\gamma\cdot a,\gamma\cdot b)$ for some $\gamma \in \mathbb{F}_p^*$, which is the coefficient.

The Knowledge of Coefficient Assumption (KCA):  If Alice returns a valid response $(a',b')$  to Bob's challenge $(a,b)$ with non-negligible probability over Bob's choices of $a,\alpha$ then she knows $\gamma$ such that $a'=\gamma \cdot a$.

### Verifiable Blind Evaluation of Polynomials 

Blindness: Alice will not learn $s$ and Bob will not learn $P$. 

Verifiability: The probability that Alice sends a value not $E(P(s))$  for $P$ of degree $d$ but Bob still accepts – is negligible.

The foregoing KCA achieves the blindness, now we need an extended version of KCA .

**d-KCA**: Suppose Bob chooses random $\alpha \in \mathbb{F}_p^*$ and $s\in \mathbb{F}_p$, and sends to Alice the $\alpha-$pairs $(g,\alpha\cdot g),(s\cdot g,\alpha s\cdot g),\cdots,(s^d\cdot g,\alpha s^d\cdot g)$. Suppose that Alice then outputs another $\alpha-$pair $(a',b')$. Then, except with negligible probability, Alice knows $c_0,\cdots,c_d\in\mathbb{F}_p$ such that $\sum_{i=0}^d c_i s^i \cdot g=a'$.

###  From Computations to Polynomials

Suppose Alice wants to prove to Bob she knows $c_1,c_2,c_3\in \mathbb{F}_p$ such that $(c_1\cdot c_2)\cdot (c_1+c_3)=7$. The first step is to present the expression computed from $c_1,c_2,c_3$ as an arithmetic circuit.

<img src="https://pico-1258741719.cos.ap-shanghai.myqcloud.com/CircuitDrawing-1.png" alt="img" style="zoom:60%;" />

- When the same outgoing wire goes into more than one gate, we still think of it as one wire – like $w_1$ in the example.
- We assume multiplication gates have exactly two input wires, which we call the left wire and right wire.
- We don't label the wires going from an addition to multiplication gate, nor the addition gate; we think of the inputs of the addition gate as going directly into the multiplication gate. So in the example we think of $w_1$ and $w_3$ as both being right inputs of $g_2$.

A legal assignment for the circuit is an assignment of values to the labeled wires where the output value of each multiplication gate.

For example, a legal assignment is of the form: $(c_1,\cdots,c_5)$ where $c_4=c_1\cdot c_2$ and $c_5=c_4\cdot (c_1+c_3)$.

In this example, Alice wants to prove that she knows a legal assignment $(c_1,\cdots,c_5)$ such that $c_5=7$.

We associate each multiplication gate with a field element: $g_1$ will be associated with $1\in \mathbb{F}_p$ and $g_2$ with $2\in \mathbb{F}_p$. We call the points $\{1,2\}$ our target points. Now we need to define a set of "left wire polynomials" $L_1,\cdots,L_5$, "right wire polynomials" $R_1,\cdots,R_5$ and "output wire polynomials" $O_1,\cdots,O_5$. 

$w_1,w_2,w_4$ are respectively the left, right and output wire of $g_1$; define $L_1=R_2=O_4=2-X$, as the polynomial $2-X$ is one on the point 1 corresponding to $g_1$ and zero on the point 2 corresponding to $g_2$. 

$w_1, w_3$ are both right inputs of $g_2$; define $R_1 = R_3 = L_4 = O_5=X-1$, as $X-1$ is one on the point 1 corresponding to $g_2$ and zero on the point 1 corresponding to $g_1$. 

Set the rest of the polynomials to be the zero polynomial.

Define the "sum" polynomials. 

$$L:=\sum_{i=1}^5 c_i \cdot L_i;R:=\sum_{i=1}^5 c_i \cdot R_i;O:=\sum_{i=1}^5 c_i \cdot O_i;P:=L\cdot R - O$$

$P(1)=L(1)\cdot R(1)-O(1)=c_1\cdot c_2 - c_4; P(2)=c_4\cdot (c_1+c_3) - c_5$

Define the target polynomial $T(X):=(X-1)\cdot (X-2)$

Fact: For a Polynomial $P$ and a point $a\in \mathbb{F}_p$, $P(a) = 0 \Leftrightarrow P =(X-a)\cdot H$ for some $H$.

Thus $(c_1,\cdots,c_5)$ is a legal assignment if and only if $T$ divides $P$.

Now we can define a QAP as follows:

A Quadratic Arithmetic Program $Q$ of degree $d$ and size $m$ consists of polynomials $L_1,\cdots,L_m,R_1,\cdots,R_m,O_1,\cdots,O_m$ and a target polynomial $T$ of degree $d$.

An assignment $(c_1,\cdots,c_5)$ satisfies $Q$ if $L:=\sum_{i=1}^5 c_i \cdot L_i;R:=\sum_{i=1}^5 c_i \cdot R_i;O:=\sum_{i=1}^5 c_i \cdot O_i;P:=L\cdot R - O$, we have $T$ divides $P$.

### The Pinocchio Protocol

This part shows how Alice can send a very short proof to Bob on an assignment to a QAP.

1. Alice chooses polynomial $L,R,O,P$ of degree at most $d$899 9999999
