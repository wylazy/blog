---
id: 1527563667
title: Moore NIM 必胜策略
date: 2018-05-29 11:14:27
categories: 算法
tags:
  算法
mathjax2: true
---

### 问题描述
Moore Nim 也叫 index-k Nim，这个问题由 Moore 在 1910 年提出，是普通 Nim 问题的扩展，问题的描述如下。

总共有 $n$ 堆石子，每堆石子的个数分别为 $a_1, a_2, a_3 ... a_n$ ；有 2 个玩家轮流取石子，每次选择 $k$ 堆石子，从中取任意多个石子，而且至少要取一个石子。取走最后一颗石子的玩家获胜，问先手有没有必胜策略。

### 必胜状态
必胜状态显然和每个石子的个数有关，需要分析每堆石子的个数，将每堆石子的初始个数 $a_i$ 写成二进制展开形式：
$$
\begin{align}
a_0 &= c_{00} * 2^0 + c_{01} * 2^1 + ... + c_{0m} * 2^m \\
a_1 &= c_{10} * 2^0 + c_{11} * 2^1 + ... + c_{1m} * 2^m \\
    & ... \\
a_n &= c_{n0} * 2^0 + c_{n1} * 2^1 + ... + c_{nm} * 2^m
\end{align}
$$
将上述方程组竖着看，将每一列的系数相加：
$$
\begin{align}
c_0 &= c_{00} + c_{10} + ... + c_{n0} \pmod {k+1} \\
c_1 &= c_{01} + c_{11} + ... + c_{n1} \pmod {k+1}\\
    & ... \\
c_m &= c_{0m} + c_{1m} + ... + c_{nm} \pmod {k+1} \\
\end{align}
$$
如果每一项系数之和 $c_i$ 都等于 $0$，先手必败，否者先手必胜。
也就是说如果每堆石子的二进制展开系数和除以 $(k+1)$ 余数为$0$，先手必败，否则先手必胜。

证明过程将在文章的最后才会给出。
为了方便描述，将上述从 $A = \{a_0, a_1, ... a_n\}$ 到有序列表 $C=(c_m, c_{m-1}, ... c_1, c_0)$
的映射定义为函数 $f_{k+1}: f_{k+1}(A) \to C$ ,
即 $f_{k+1}$ 将 $A$ 中每个数写成二进制形式，将对应的位按模 $k+1$ 相加，并将它们排列成列表 $C$ 。

### 证明必胜状态
<b>定理一：</b>将二进制数最左边的 $1$ 置为 $0$ ，无论它右边的比特如何设置，其值都小于原来的数值。比如：
``` plain
a = 10001001
b = 0xxxxxxx
```
在上面的例子中 $x \in \{0, 1\}$ , $a$ 和 $b$ 一定满足关系 $b < a$。
这是显然的，因为 $a$ 比 $b$ 用二进制表示要多一位，一个 $8$ 位数肯定要比 $7$ 位数大。
反过来说，在取石子的游戏中，玩家可以通过降低某堆石子最高比特位的方式，让剩余的比特为任意 $\{0, 1\}$ 组合。

<b>证明必胜策略：</b>

首先将石子数在 $f_{k+1}$ 的映射下形成有序列表 $C$，证明过程需要对每个比特按位处理。

证明必败状态：如果 $C$ 中每个元素 $c_i = 0$ ，那么玩家一次操作之后，只能将游戏局面变成必胜状态。
因为玩家至少取一个石子，所以操作之后，至少会导致某个 $c_i \ne 0$ ，而且最多只能操作 $k$ 堆石子，所以
新的 $ci$ 满足关系 $0 < c_i \le k$，从而使对手到达必胜状态。

证明必胜状态：从左向右扫描 $C$ 中每个元素，找到第一个不为零的元素，比如 $c_i \neq 0$ ，
说明 $A$ 中总共有 $c_i$ 堆石子，它们的第 $i$ 个比特位是1，将它们的最高比特位置零。
记录已修改的石子堆数 $u = c_i$，以后的操作可以增加 $u$ ，但应该满足关系 $0 < u \le k$ 。
操作之后对未修改的石子堆集合做 $f_{k+1}$ 运算，映射到一个新的列表 $C$，
然后继续扫描列表 $C$ 中的下一个元素，比如下一个不为零的元素是 $c_j$。
说明有 $c_j$ 堆石子，它们的第 $j$ 个比特位是1。 这时分 2 种情况考虑。

情况一：如果 $u + c_j \le k$，直接将这 $c_j$ 堆石子的最高比特位置 $0$，更新 $u = u + c_j$。

情况二：如果 $u + c_j > k$，因为最多只能取 $k$ 堆石子，所以不能简单的将这 $c_j$ 堆石子的最高比特位置零。
由条件假设知 $u > k - c_j \Rightarrow u \ge k - c_j + 1$，
所以我们可以在 $u$ 中选择 $k - c_j + 1$ 堆石子，将它们的第 $j$ 比特位设置为 $1$。
根据定理一我们知道，这不会增加石子数，所以操作是可行的。
操作之后我们计算一下新的石子在第 $j$ 比特位之和为 $c_j + (k - c_j + 1) = k + 1$，满足模 $k + 1$为零。
然后我们不断的从左向右处理每一个比特位，最终所有比特和都满足模 $k+1$为零。

因此无论是这两种的哪种情况，我们都有办法使第 $j$ 个比特位和满足模 $k + 1$ 为零，
不断使用这个步骤，可以将所有比特位都满足模 $k+1$ 为零，从而使游戏局面变成必败状态。
