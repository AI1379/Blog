---
title: 树链剖分
date: 2023-5-30 21:13:00
tags:
  - 数据结构
  - 树形结构
categories:
  - DataStructure
---

## 关于树链剖分

首先，需要知道一个事情：**树链剖分其实很简单**。

其次，需要知道另一个事情：**树链剖分能解决的只有静态树上问题**。如果你需要解决动态树（指的是树的结构发生变化）的问题，那么你需要的应该是[Link-Cut Tree](https://oi-wiki.org//ds/lct/)

那么树剖可以解决什么问题呢？

事实上，单单一个树剖能解决的问题不多，但是由于树剖的一些优秀性质，使得我们可以通过树剖+线段树的方式，实现：

1. 子树查/改
2. 路径查/改

这无疑是非常有用的。

另一方面，对于某些类型的树形dp，我们也可以使用树剖优化。

需要注意的是，通常来讲认为树剖分为两类，第一类是**重链剖分**，通常与线段树结合，另一类是**长链剖分**，通常用于优化dp。还有一类特殊的，有时被称作**实链剖分**，只用于LCT当中。

那么我们开始。

## 重链剖分

[例题：P3384](https://www.luogu.com.cn/problem/P3384)

首先我们要知道重链剖分是什么。在此之前，有一个概念叫重儿子。~~故名思义，就是最重的儿子~~。对于每一个节点，他的所有儿子中，对应的子树中节点个数最多的儿子，被称作一个节点的重儿子。（其实就是最重的儿子）

那么显然，除了叶子节点以外，每个节点有且只有一个重儿子。我们把所有的节点与他们的重儿子连起来，就可以把整棵树分成一条条链，这个链的个数是$O(\log n)$的。容易发现，我们只要一次深搜就可以完成查找所有重儿子的任务。

接下来，我们就要找重链。这个问题也很好解决，我们只要再进行一次深搜，每次搜索的时候优先搜索重儿子即可。

> （大概就是长这样）
>
> ![](https://oi-wiki.org//graph/images/hld.png)
>
> 图源oi-wiki

但是这样还没用。要利用树剖的性质，我们需要记录在第二次深搜中每个节点的`dfn`序。容易发现，按照第二次深搜的流程， **同一条重链上的节点`dfn`必然连续，同一颗子树的节点`dfn`也连续。** 也就是说，我们完全可以通过树链剖分，把**子树和链上处理转化为`dfn`上的区间处理**。

那么我们就可以很容易地使用区间处理的数据结构（比如线段树）实现树上子树和链上处理。

但是我们还有一些细节需要处理。子树问题很好解决，但是链上处理比较复杂。虽然重链保证连续，但是我们修改的时候不一定保证在同一条重链上。

首先我们要知道如何判定两个点在不在一条链上。这很简单，只要判断一下链顶是否一样即可。

然后处理不在一条链的情况。类似倍增LCA的思想，我们考虑一条一条往上跳，直到在同一条链上。具体见代码。

下面是例题代码：

```cpp
// https://www.luogu.com.cn/problem/P3384
#include <bits/stdc++.h>
using namespace std;
#define ll long long
const ll MAXN = 100010;
struct Node {
  ll l, r, val, tag;
};
ll n, m, r, p;
// orig 存的是输入数据
// val 表示每个 dfn 对应节点的值
ll orig[MAXN], val[MAXN];
vector<ll> tree[MAXN];
// parent 是当前节点的父节点
// prefer 是倾向的儿子也就是重儿子
// weight 是每个儿子的重量
ll parent[MAXN], prefer[MAXN], weight[MAXN];
ll dfn[MAXN], dep[MAXN];
// top 表示每个节点所在的链顶和链底
// bottom 表示每个节点对应的子树的最后一个节点
ll top[MAXN], bottom[MAXN];
ll idx = 0;
Node segt[MAXN * 4];
#define lc(x) segt[(x)*2]
#define rc(x) segt[(x)*2 + 1]

// 两次深搜树剖
// 第一次深搜记录儿子们的重量并且找出重儿子
void buildTree(ll u, ll p) {
  parent[u] = p;
  dep[u] = dep[p] + 1;
  for (auto v : tree[u]) {
    if (v != p) {
      buildTree(v, u);
      weight[u] += weight[v];
      if (weight[v] > weight[prefer[u]])
        prefer[u] = v;
    }
  }
  ++weight[u];
}
// 第二次深搜找链
// t 记录当前节点所在链的链顶
void treeDecomposition(ll u, ll t) {
  dfn[u] = ++idx;
  top[u] = t;
  bottom[u] = dfn[u];
  if (prefer[u])
    // 重儿子的链顶继承父亲的链顶
    treeDecomposition(prefer[u], t);
  for (auto v : tree[u]) {
    if (v == parent[u])
      continue;
    if (v != prefer[u])
      // 不是重儿子的链顶就是自己
      treeDecomposition(v, v);
    bottom[u] = max(bottom[u], bottom[v]);
  }
}

// 线段树板子
void buildSegt(ll l, ll r, ll cur) {
  segt[cur].l = l;
  segt[cur].r = r;
  if (l == r) {
    segt[cur].val = val[l];
    return;
  }
  ll mid = (l + r) / 2;
  buildSegt(l, mid, cur * 2);
  buildSegt(mid + 1, r, cur * 2 + 1);
  segt[cur].val = (segt[cur * 2].val + segt[cur * 2 + 1].val) % p;
}
void pushdown(ll cur) {
  if (segt[cur].tag) {
    lc(cur).val =
        (lc(cur).val + segt[cur].tag * (lc(cur).r - lc(cur).l + 1) % p) % p;
    rc(cur).val =
        (rc(cur).val + segt[cur].tag * (rc(cur).r - rc(cur).l + 1) % p) % p;
    lc(cur).tag = (lc(cur).tag + segt[cur].tag) % p;
    rc(cur).tag = (rc(cur).tag + segt[cur].tag) % p;
    segt[cur].tag = 0;
  }
}
void modifySegt(ll s, ll t, ll delta, ll cur) {
  if (s > t)
    swap(s, t);
  if (s <= segt[cur].l && segt[cur].r <= t) {
    segt[cur].val =
        (segt[cur].val + delta * (segt[cur].r - segt[cur].l + 1) % p) % p;
    segt[cur].tag = (segt[cur].tag + delta) % p;
    return;
  }
  ll mid = (segt[cur].l + segt[cur].r) / 2;
  pushdown(cur);
  if (s <= mid)
    modifySegt(s, t, delta, cur * 2);
  if (mid + 1 <= t)
    modifySegt(s, t, delta, cur * 2 + 1);
  segt[cur].val = (lc(cur).val + rc(cur).val) % p;
}
ll querySegt(ll s, ll t, ll cur) {
  if (s > t)
    swap(s, t);
  if (s <= segt[cur].l && segt[cur].r <= t) {
    return segt[cur].val;
  }
  ll res = 0, mid = (segt[cur].l + segt[cur].r) / 2;
  pushdown(cur);
  if (s <= mid)
    res = (res + querySegt(s, t, cur * 2)) % p;
  if (mid + 1 <= t)
    res = (res + querySegt(s, t, cur * 2 + 1)) % p;
  return res;
}

// 链上修改
void modifyPath(ll u, ll v, ll delta) {
  // 当不在同一条链上时
  while (top[u] != top[v]) {
    // 跳的链应该是链顶深度较大的点
    // 注意看的是所在链的链顶的深度而不是自己的深度
    // （可以自己画个图理解一下）
    if (dep[top[u]] < dep[top[v]])
      swap(u, v);
    // 一层一层向上跳链
    // 在线段树上修改当前节点到当前所在链的链顶
    // 注意修改区间的边界是对应的dfn
    modifySegt(dfn[u], dfn[top[u]], delta, 1);
    // 跳到链顶的父亲
    u = parent[top[u]];
  }
  // 在同一条重链上
  // 可以直接修改线段树
  modifySegt(dfn[u], dfn[v], delta, 1);
}
ll queryPath(ll u, ll v) {
  ll res = 0;
  while (top[u] != top[v]) {
    if (dep[top[u]] < dep[top[v]])
      swap(u, v);
    res = (res + querySegt(dfn[u], dfn[top[u]], 1)) % p;
    u = parent[top[u]];
  }
  res = (res + querySegt(dfn[u], dfn[v], 1)) % p;
  return res;
}
// 子树上修改
// 直接修改线段树即可
void modifySubtree(ll u, ll delta) { modifySegt(dfn[u], bottom[u], delta, 1); }
ll querySubtree(ll u) { return querySegt(dfn[u], bottom[u], 1); }

int main() {
  ll opt, x, y, z;
  cin >> n >> m >> r >> p;
  for (int i = 1; i <= n; i++) {
    cin >> orig[i];
  }
  for (int i = 1; i <= n - 1; i++) {
    cin >> x >> y;
    tree[x].push_back(y);
    tree[y].push_back(x);
  }
  buildTree(r, 0);
  treeDecomposition(r, r);
  // 记录每个dfn对应的权值
  for (int i = 1; i <= n; i++) {
    val[dfn[i]] = orig[i] % p;
  }
  buildSegt(1, n, 1);
  while (m--) {
    cin >> opt;
    if (opt == 1) {
      cin >> x >> y >> z;
      modifyPath(x, y, z);
    } else if (opt == 2) {
      cin >> x >> y;
      cout << queryPath(x, y) << endl;
    } else if (opt == 3) {
      cin >> x >> y;
      modifySubtree(x, y);
    } else {
      cin >> x;
      cout << querySubtree(x) << endl;
    }
  }
  return 0;
}
```

## 长链剖分

原理和重链剖差不多，不过这里的重子定义为子树深度最大的儿子。这样的话链的个数就不再是 $O(\log n)$ 而是 $O(\sqrt n)$

这个时候由于链的个数的复杂度问题，我们通常不会使用长链剖分来解决子树和链上的查改问题，而是用它来优化树形dp。这类dp有一个很显著的特点：状态转移方程通常会和深度有关。

[例题：CF1009F](https://www.luogu.com.cn/problem/CF1009F)

这个题显然可以dp。我们定义`dp[i][j]`表示`i`为根的子树距离`j`的节点个数。显然，可以 $O(n^2)$ 复杂度进行转移。

但是显然这是过不去的。我们考虑优化。

长链剖分之后，考虑每个节点直接继承重儿子的dp数组和答案。对于轻儿子，我们考虑直接暴力合并。由于总的点数是 $O(n)$ 的，所以最后总的时空复杂度也是 $O(n)$ 的。

代码：

```cpp
#include <bits/stdc++.h>
using namespace std;
#define ll long long
const ll MAXN = 1000010;
ll n;
vector<ll> tree[MAXN];
// val是内存池，存放所有的dp数据
// dp指向内存池中的某个位置 方便合并
// ans记录答案
ll val[MAXN], *dp[MAXN], *cur, ans[MAXN];
// len记录重链长度
ll len[MAXN], prefer[MAXN];
// 树剖
void decomposition(ll u, ll f) {
  for (auto v : tree[u]) {
    if (v == f)
      continue;
    decomposition(v, u);
    if (len[v] > len[prefer[u]])
      prefer[u] = v;
  }
  len[u] = len[prefer[u]] + 1;
}
// dp
void dfs(ll u, ll f) {
  dp[u][0] = 1;
  if (prefer[u]) {
    dp[prefer[u]] = dp[u] + 1;
    dfs(prefer[u], u);
    ans[u] = ans[prefer[u]] + 1;
  }
  for (auto v : tree[u]) {
    if (v == f || v == prefer[u])
      continue;
    dp[v] = cur;
    cur += len[v];
    dfs(v, u);
    for (int i = 1; i <= len[v]; ++i) {
      dp[u][i] += dp[v][i - 1];
      if ((i < ans[u] && dp[u][i] >= dp[u][ans[u]]) ||
          (i > ans[u] && dp[u][i] > dp[u][ans[u]]))
        ans[u] = i;
    }
  }
  if (dp[u][ans[u]] == 1)
    ans[u] = 0;
}
int main() {
  ll u, v;
  cin >> n;
  for (int i = 1; i <= n - 1; ++i) {
    cin >> u >> v;
    tree[u].emplace_back(v);
    tree[v].emplace_back(u);
  }
  decomposition(1, 0);
  dp[1] = cur = val;
  cur += len[1];
  dfs(1, 0);
  for (int i = 1; i <= n; ++i)
    cout << ans[i] << endl;
  return 0;
}
```
