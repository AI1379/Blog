---
title: 连续正整数的因子和之和问题
date: 2022-08-23 23:10:37
mathjax: true
tags:
  - 数学
categories:
  - Math
---

## 题目大意

已知正整数$x<y$, 求:
$$
\sum^{y}_{i=x} \sum _{j|i} j
$$

数据范围：

对于$50\%$的数据，有：$0<x,y<=1\times10^6$

对于$100\%$的数据，有：$0<x,y<=2\times10^9$

保证结果在`long long`范围内

## 题目分析

题面非常的直白。令$\sigma(x)=\sum _{i|x}x$，$D(x)=\sum^x_{i=1}\sigma(i)$，那么显然，我们所求的东西就是$D(y)-D(x-1)$。

接下来的问题就是，如何快速的求$D(x)$。

显然我们有一个$O(n)$的解法，就是枚举每一个数，求区间内他的倍数出现次数，得到$D(x)=\sum^x_{i=1}i\lfloor\frac{x}{i}\rfloor$。但这显然无法满足全部数据的要求。我们需要更快的解法。在此我们给出一个$O(\sqrt{n})$的解法。

先给出一个结论：

$$
\begin{align*}
D(x)&=\sum^{x}_{i=1}\sigma(i)\\
    &=x^2-\sum^{x}_{i=1}(x\ mod\ i)
\end{align*}
$$

------

对于这个结论的证明（可以先跳过）：

$$
\begin{align*}
D(x)&=x^2-\sum^{x}_{i=1}(x\ mod\ i)\\
    &=\sum^x_{i=1}x-\sum^x_{i=1}(x\ mod\ i)\\
    &=\sum^x_{i=1}(x-(x\ mod\ i))\\
    &=\sum^x_{i=1}i\lfloor\frac{x}{i}\rfloor\\
    &=\sum^x_{i=1}\sigma(i)
\end{align*}
$$

------

那么我们的问题就是如何快速的求出这个$\sum^{x}_{i=1}(x\ mod\ i)$。

显然我们不可能直接枚举求它，这就不算优化了。这个东西乍一看比较困难。我们尝试求一下$x=100$时每一项的值：

```plaintext
0, 0, 1, 0, 0, 4, 2, 4, 1, 0, 1, 4, 9, 2, 10, 4, 15, 10, 5, 0,
16, 12, 8, 4, 0, 22, 19, 16, 13, 10, 7, 4, 1,
32, 30, 28, 26, 24, 22, 20, 18, 16, 14, 12, 10, 8, 6, 4, 2, 0,
49, 48, 47, 46, 45, 44, 43, 42, 41, 40, 39, 38, 37, 36, 35, 34, 33, 32, 31, 30, 29, 28, 27, 26, 25, 
24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 
6, 5, 4, 3, 2, 1, 0
```

我们可以发现：从某一项开始，他的值是由几段连续的等差数列拼成的，并且这几个等差数列的公差还是单调下降的一段连续自然数。这启发我们把它分解成几段连续的等差数列计算。

接下来就要考虑每段等差数列的起点问题。观察100的数据，我们发现，等差数列的起点分别是`x/2+1`, `x/3+1`, `x/4+1`, `x/5+1`等。那么我们只要枚举这个分母，然后利用等差数列求和即可。

------

关于这个等差数列的感性证明：

首先我们从大到小考虑。对于一个$i<x$，假如$i\in (\lfloor\frac2{x}\rfloor,x]$，那么很显然$x\ mod\ i =x-i$。那么当$i$从$\lfloor\frac2{x}\rfloor$开始增大到$x$时，这显然就是一段等差数列。同样这也可以推广到分母为别的数的时候。

------

但是我们仍然没有解决实际问题。枚举分母这个过程仍然是线性时间的。但是我们可以只枚举分母到平方根，求出$\sqrt x\sim x$这几项的和。而对于小于$\sqrt x$的部分我们可以直接求对应项的和，这样就可以实现在$O(\sqrt n)$时间内求解这个问题。

代码如下：

```cpp
#include <bits/stdc++.h>
using namespace std;
#define ll long long
static ll x, y;
static ll ans;
static ll num, s, t, curr, res, ori;
ll calc(ll x) {
  num = 1, curr = 1, res = 0, ori = x;
  while (x * x > ori) {
    s = ori % x;
    t = ori % (ori / curr + 1);
    num = x - ori / curr;
    res += (s + t) * num / 2;
    x = ori / curr;
    ++curr;
  }
  while (x) {
    res += ori % x;
    --x;
  }
  return ori * ori - res;
}
int main() {
  cin >> x >> y;
  ans = calc(y) - calc(x - 1);
  cout << ans << endl;
  return 0;
}
```
