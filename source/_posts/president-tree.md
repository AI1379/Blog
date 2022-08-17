---
title: 算法学习笔记：主席树
date: 2022-08-17 14:28:01
mathjax: true
tags:
  - 数据结构
categories:
  - DataStructure
  - Persistent
---

前置知识：[线段树](https://oi-wiki.org/ds/seg/)

## 基本原理

主席树，又名**可持久化权值线段树**，是一种可持久化线段树。当然，很多地方通常也会用主席树来指代可持化线段树。它是最常见的可持久化数据结构之一。

要想实现线段树的可持久化，有一个很笨的方法：每次更改都复制一份进行修改。很显然，这在时空上效率都极其低下。我们要考虑如何节省时间和空间。此时我们很容易想到：只记录修改的节点。由于线段树的深度是$O(\log n)$的，所以每次单点修改最多修改$O(\log n)$的节点，因此也可以在$O(\log n)$的空间内实现历史记录。

这个时候可持久化线段树的结构就很容易想到了。由于我们增加了很多节点，因此我们不能用常规线段树用两倍表示孩子的做法，而应该使用类似动态开点线段树的策略，在节点里面保存左右儿子的编号。

![图源lyd蓝书](/img/president-tree/structure.png)

接下来我们考虑怎么实现单点修改和区间查询。在可持久化线段树上实现区间修改是比较困难的，之后再考虑。

### 建树 & 区间查询

~~其实建树和区间查询的代码和正常的线段树一模一样~~

从图上我们可以看出来，假如说我们从某一个根节点开始向下深搜，得到的树就是一棵正常的线段树，因此我们只要从某一个根节点开始，和普通线段树一样向下查询，即可得到正确的结果。

### 单点修改

单点修改有点麻烦。

进行一次修改的时候，首先需要从原来的版本里复制一个根节点出来，然后同时从新旧两个根开始向下遍历，每走到一个要更新的节点就创建这个节点的一个副本，然后继续向下遍历，直到叶节点。

于是我们就可以开始写代码了。[【模板】可持久化线段树 1（可持久化数组）](https://www.luogu.com.cn/problem/P3919)

代码如下（注：这里的查询是单点查询）

```cpp
#include <iostream>
using namespace std;
#define ll int
const ll MAXN = 1000010;
const ll MAXM = 1000010;
const ll MAXMLOGN = 30 * MAXM;
struct Node {
  ll lchild, rchild, val;
  ll l, r;
};
Node segt[MAXMLOGN];
ll n, m;
ll val[MAXN];
ll roots[MAXM];
ll tot;
ll build(ll l, ll r) {
  ll pos = ++tot;
  segt[pos].l = l;
  segt[pos].r = r;
  if (l == r) {
    segt[pos].val = val[l];
    return pos;
  }
  ll mid = (l + r) / 2;
  segt[pos].lchild = build(l, mid);
  segt[pos].rchild = build(mid + 1, r);
  return pos;
}
ll query(ll p, ll cur) {
  if (segt[cur].l == p && segt[cur].r == p) {
    return segt[cur].val;
  }
  ll mid = (segt[cur].l + segt[cur].r) / 2;
  if (p <= mid) {
    return query(p, segt[cur].lchild);
  } else {
    return query(p, segt[cur].rchild);
  }
}
void modify(ll p, ll val, ll pre, ll cur) {
  segt[cur].l = segt[pre].l;
  segt[cur].r = segt[pre].r;
  if (segt[cur].l == p && segt[cur].r == p) {
    segt[cur].val = val;
    return;
  }
  ll mid = (segt[cur].l + segt[cur].r) / 2;
  if (p <= mid) {
    segt[cur].lchild = ++tot;
    segt[cur].rchild = segt[pre].rchild;
    modify(p, val, segt[pre].lchild, segt[cur].lchild);
  } else {
    segt[cur].rchild = ++tot;
    segt[cur].lchild = segt[pre].lchild;
    modify(p, val, segt[pre].rchild, segt[cur].rchild);
  }
}
ll v, k, s, t, x;
int main() {
  ios::sync_with_stdio(false);
  cin.tie(nullptr);
  cout.tie(nullptr);
  cin >> n >> m;
  for (int i = 1; i <= n; i++) {
    cin >> val[i];
  }
  roots[0] = build(1, n);
  for (int i = 1; i <= m; i++) {
    cin >> v >> k;
    if (k == 1) {
      cin >> s >> t;
      roots[i] = ++tot;
      modify(s, t, roots[v], roots[i]);
    } else {
      cin >> x;
      roots[i] = ++tot;
      segt[roots[i]] = segt[roots[v]];
      cout << query(x, roots[i]) << endl;
    }
  }
  return 0;
}
```

## 例题：区间第k小问题

[题目链接](https://www.luogu.com.cn/problem/P3834)

这个问题的解法很多很多，包括主席树、树套树、CDQ分治等等。我们这里先讲主席树，树套树和CDQ则留到后面。

在序列上直接建立线段树看起来是不太可行的，因此我们考虑在值域上建树。

首先对数列进行离散化。然后我们建立线段树。设某一个节点维护的区间为$[l,r]$，那么它的值所表示的就是序列中值在$[l,r]$范围内的数的个数。然后我们从左往右扫过序列，每次都在这棵主席树上进行单点修改，我们就可以在$O(\log n)$的时间内得到这个序列的每一个前缀中在$[l,r]$范围内的数的个数。

接下来，询问$[s,t]$区间的第$k$大时，我们就从`root[s-1]`和`root[t]`同时向下查找，很显然，两棵树中相对应的节点上的值的差就表示了$[s,t]$中在$[l,r]$范围内的数出现的次数。那么，当我们查找到一个节点时，判断他们左子树的差`delta`是否大于等于`k`。如果成立，说明这个区间内的第$k$大在他们的左子树中，否则就往右子树遍历。这样就可以在$O(\log n)$的时间内处理每个查询了。

代码如下：

```cpp
// https://www.luogu.com.cn/problem/P3834
// static range kth element
#include <algorithm>
#include <iostream>

using namespace std;
#define ll long long
const ll MAXN = 200010;
const ll MAXMLOGN = 20 * MAXN;
struct Node {
  ll l, r;
  ll lchild, rchild;
  ll val;
};
static Node segt[MAXMLOGN];
static ll tot;
static ll n, m, num;
static ll val[MAXN], f[MAXN];
static ll roots[MAXN];
#define lc segt[cur].lchild
#define rc segt[cur].rchild
void insert(ll v, ll cur, ll pre) {
  segt[cur].val = segt[pre].val;
  segt[cur].lchild = segt[pre].lchild;
  segt[cur].rchild = segt[pre].rchild;
  if (segt[cur].l == v && segt[cur].r == v) {
    ++segt[cur].val;
    return;
  }
  ll mid = (segt[cur].l + segt[cur].r) / 2;
  if (v <= mid) {
    segt[cur].lchild = ++tot;
    segt[lc].l = segt[cur].l;
    segt[lc].r = mid;
    insert(v, lc, segt[pre].lchild);
  } else {
    segt[cur].rchild = ++tot;
    segt[rc].l = mid + 1;
    segt[rc].r = segt[cur].r;
    insert(v, rc, segt[pre].rchild);
  }
  segt[cur].val = segt[lc].val + segt[rc].val;
}
ll query(ll k, ll lcur, ll rcur) {
  if (segt[lcur].l == segt[lcur].r && segt[rcur].l == segt[rcur].r &&
      segt[rcur].val - segt[lcur].val >= k) {
    return segt[lcur].l == 0 ? segt[rcur].l : segt[lcur].l;
  }
  ll delta = segt[segt[rcur].lchild].val - segt[segt[lcur].lchild].val;
  if (delta >= k) {
    return query(k, segt[lcur].lchild, segt[rcur].lchild);
  } else {
    return query(k - delta, segt[lcur].rchild, segt[rcur].rchild);
  }
}
int main() {
  ll l, r, k;
  ios::sync_with_stdio(false);
  cin.tie(nullptr);
  cout.tie(nullptr);
  cin >> n >> m;
  for (int i = 1; i <= n; i++) {
    cin >> val[i];
    f[i] = val[i];
  }
  sort(f + 1, f + n + 1);
  num = unique(f + 1, f + n + 1) - f - 1;
  for (int i = 1; i <= n; i++) {
    val[i] = lower_bound(f + 1, f + num + 1, val[i]) - f;
  }
  roots[0] = ++tot;
  segt[roots[0]] = Node{1, num, 0, 0, 0};
  for (int i = 1; i <= n; i++) {
    roots[i] = ++tot;
    segt[roots[i]] = segt[roots[i - 1]];
    insert(val[i], roots[i], roots[i - 1]);
  }
  for (int i = 1; i <= m; i++) {
    cin >> l >> r >> k;
    cout << f[query(k, roots[l - 1], roots[r])] << endl;
  }
  return 0;
}
```
