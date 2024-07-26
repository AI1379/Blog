---
title: 树链剖分
date: 2023-5-30 21:13:00
mathjax: true
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

事实上，单单一个树剖能解决的问题不多，但是由于树剖的一些优秀性质，使得我们可以通过树剖+数据结构（例如线段树）的方式，实现：

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

那么我们就可以很容易地使用区间处理的数据结构（比如线段树）实现树上子树和链上处理。比如，在这张图中，我们想要修改`dfn`为4和19的节点之间路径上的信息，那么我们就可以直接在线段树上修改`1-4, 16-17, 19`这三个区间。又如，我们若要修改`dfn`为2的子树信息，只需在线段树上修改`2-10`这一区间即可。**注意，在树剖完后，我们所有的节点都是通过对应的`dfn`序来访问和修改的，而不是节点原来的编号，这个一定要记牢，否则很容易混淆导致出错**。


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

再讲一个例题 [「SDOI」2014 旅行](https://www.luogu.com.cn/problem/P3313)

这个题一眼看上去就是一个树剖的题，因为涉及了树上链上查询和修改。但是有一个小问题是，他查询并不是查询一个区间里所有的点，而是只取其中的一类（也就是信仰一个宗教的城市）。这种情况下，有一个很暴力的做法是：针对每一类节点（每一种信仰）都开一颗单独的线段树来维护。显然，这样会暴空间。

但是我们有一个想法。注意到，我们正常的线段树是按照完全二叉树的方式存储的，这其中有很多叶节点的空间是浪费掉的。同时，由于还有很多的节点其实是空的（因为不信仰这个宗教），所以这种做法的空间浪费是很厉害的。

于是我们考虑**动态开点**线段树。

什么意思呢？顾名思义，就是**用一个点开一个点**的线段树。这样可以极大地节省空间。这样的话，我们就要抛弃原来的完全二叉树的写法，转而将儿子存在树的节点里面，就像普通的二叉树一样。事实上，动态开点线段树在数据结构题里面是一个很常见的技巧，不一定非得要和树链剖分绑定，同时还可以延伸出另一种叫主席树的数据结构，但这超出了这里的讨论范围。

那么代码就好写了。虽然代码会有点长，但是逻辑还是很好理解的。

```cpp
#include <bits/stdc++.h>
using namespace std;
#define ll long long
#define pll pair<ll, ll>
const ll MAXN = 100010;
struct Node {
  ll l, r;
  ll lson, rson;
  ll s, m;
};
ll n, q;
ll tot, root[MAXN];
// 线段树的内存池
Node segt[MAXN * 16];
vector<ll> tree[MAXN];
ll tag[MAXN], val[MAXN];
ll idx, weight[MAXN], prefer[MAXN];
ll dfn[MAXN], f[MAXN], top[MAXN], dep[MAXN];
// 两次深搜树剖
void getson(ll u, ll fa) {
  weight[u] = 1;
  f[u] = fa;
  dep[u] = dep[fa] + 1;
  for (auto v : tree[u]) {
    if (v == fa)
      continue;
    getson(v, u);
    weight[u] += weight[v];
    if (weight[prefer[u]] < weight[v])
      prefer[u] = v;
  }
}
void decomposition(ll u, ll t) {
  dfn[u] = ++idx;
  top[u] = t;
  if (prefer[u])
    decomposition(prefer[u], t);
  for (auto v : tree[u]) {
    if (v == f[u] || v == prefer[u])
      continue;
    decomposition(v, v);
  }
}

// 动态开点
// 这里为了让逻辑清楚一点封装了很多函数 实际上没有他看起来那么复杂
ll newnode(ll l, ll r) {
  ++tot;
  segt[tot].l = l;
  segt[tot].r = r;
  return tot;
}
void pushup(ll cur) {
  segt[cur].m = max(segt[segt[cur].lson].m, segt[segt[cur].rson].m);
  segt[cur].s = segt[segt[cur].lson].s + segt[segt[cur].rson].s;
}
// 单点修改
// 写法和普通的线段树差不多
void modify(ll s, ll v, ll cur) {
  if (segt[cur].l == s && segt[cur].r == s) {
    segt[cur].m = segt[cur].s = v;
    return;
  }
  ll mid = (segt[cur].l + segt[cur].r) / 2;
  if (s <= mid) {
    // 看看存不存在左儿子
    // 如果不存在就说明当前想修改的节点以前没有出现过
    // 需要建立新的节点
    if (!segt[cur].lson)
      segt[cur].lson = newnode(segt[cur].l, mid);
    modify(s, v, segt[cur].lson);
  } else {
    if (!segt[cur].rson)
      segt[cur].rson = newnode(mid + 1, segt[cur].r);
    modify(s, v, segt[cur].rson);
  }
  pushup(cur);
}
// 区间查询
ll querymx(ll s, ll t, ll cur) {
  if (s > t)
    swap(s, t);
  if (s <= segt[cur].l && segt[cur].r <= t)
    return segt[cur].m;
  ll mid = (segt[cur].l + segt[cur].r) / 2, res = 0;
  if (s <= mid && segt[cur].lson)
    res = max(res, querymx(s, t, segt[cur].lson));
  if (mid + 1 <= t && segt[cur].rson)
    res = max(res, querymx(s, t, segt[cur].rson));
  return res;
}
ll querysum(ll s, ll t, ll cur) {
  if (s > t)
    swap(s, t);
  if (s <= segt[cur].l && segt[cur].r <= t)
    return segt[cur].s;
  ll mid = (segt[cur].l + segt[cur].r) / 2, res = 0;
  if (s <= mid && segt[cur].lson)
    res += querysum(s, t, segt[cur].lson);
  if (mid + 1 <= t && segt[cur].rson)
    res += querysum(s, t, segt[cur].rson);
  return res;
}

// 修改信仰
// 相当于把这个节点在原来的线段树上归零
// 然后在另一颗线段树上把它加回去
// 这里还有一个做法是真的把这个节点从原来的二叉树上删掉
// 然后用一个堆来维护线段树的内存池里面的空节点
// 每次开点就从堆里面拿出一个节点来用
// 这样可以进一步节省空间 但是代码很繁琐
// （其实就相当于手写了一遍操作系统对堆内存的管理算法）
void modifyc(ll p, ll c) {
  modify(dfn[p], 0, root[tag[p]]);
  tag[p] = c;
  modify(dfn[p], val[p], root[c]);
}
void modifyw(ll p, ll w) {
  modify(dfn[p], w, root[tag[p]]);
  val[p] = w;
}
// 跳链查询
ll querys(ll u, ll v) {
  ll res = 0, c = tag[u];
  while (top[u] != top[v]) {
    if (dep[top[u]] < dep[top[v]])
      swap(u, v);
    res += querysum(dfn[u], dfn[top[u]], root[c]);
    u = f[top[u]];
  }
  res += querysum(dfn[u], dfn[v], root[c]);
  return res;
}
ll querym(ll u, ll v) {
  ll res = 0, c = tag[u];
  while (top[u] != top[v]) {
    if (dep[top[u]] < dep[top[v]])
      swap(u, v);
    res = max(res, querymx(dfn[u], dfn[top[u]], root[c]));
    u = f[top[u]];
  }
  res = max(res, querymx(dfn[u], dfn[v], root[c]));
  return res;
}
int main() {
  ll u, v;
  string op;
  cin >> n >> q;
  for (int i = 1; i <= n; ++i) {
    cin >> val[i] >> tag[i];
  }
  for (int i = 1; i <= n - 1; ++i) {
    cin >> u >> v;
    tree[u].emplace_back(v);
    tree[v].emplace_back(u);
  }
  getson(1, 0);
  decomposition(1, 1);
  for (int i = 1; i <= n; ++i) {
    if (!root[tag[i]])
      root[tag[i]] = newnode(1, idx);
    modify(dfn[i], val[i], root[tag[i]]);
  }
  while (q--) {
    cin >> op >> u >> v;
    if (op == "CC") {
      modifyc(u, v);
    } else if (op == "CW") {
      modifyw(u, v);
    } else if (op == "QS") {
      cout << querys(u, v) << endl;
    } else if (op == "QM") {
      cout << querym(u, v) << endl;
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
    // 扫子树的长链 合并答案
    for (int i = 1; i <= len[v]; ++i) {
      dp[u][i] += dp[v][i - 1];
      // 更新答案
      // 这里注意 因为要求最小的i使对应的dp最大 所以这里打擂的时候要分类
      if ((i < ans[u] && dp[u][i] >= dp[u][ans[u]]) ||
          (i > ans[u] && dp[u][i] > dp[u][ans[u]]))
        ans[u] = i;
    }
  }
  // 如果答案是1 也就是u的子树是一条链
  // 这个时候显然答案应该是0 也就是只留自己
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
  // 初始化
  dp[1] = cur = val;
  cur += len[1];
  dfs(1, 0);
  for (int i = 1; i <= n; ++i)
    cout << ans[i] << endl;
  return 0;
}
```

可以看到，单论代码量，长链剖分的题会比重链剖分要小不少，但是其实长链剖分优化dp的题普遍来讲是难于重链剖分+数据结构的。

由于这类dp优化种类很多并且通常较难，这里不再细讲，有兴趣可以参考[这篇文章](https://www.cnblogs.com/zhoushuyu/p/9468669.html)。

## Link-Cut Tree

把LCT放到树剖里面是因为他确实和树剖密不可分。但是我们不细讲，因为码量不小（虽然功能不少）并且难度较大。

首先我们知道一个事情，线段树能做的事情，其实用平衡树（比如Splay和Treap）也可以实现，甚至可以节省很多空间~~但是多费很多脑子~~。也就是说，我们重链剖分剖完了之后的结果完全可以使用平衡树而不是线段树来维护。

接下来我们考虑扩展树剖，使他可以支持动态树上问题。这里的动态不是指节点上的数据是动态的，而是指**这棵树（其实是森林）的形状会变**。换句话说，存在加边和删边的操作（但是保证结果是一个森林）。

这种情况下，由于树的形状发生了变化，一次树剖跑完之后的结果在变形了之后就失效了，需要重新剖分。这很麻烦，复杂度也很高。

考虑优化。

注意到，当我们删边的时候，如果删的是连到轻儿子的边，那么我们其实什么也不用做。而如果是连到重儿子的边，那我只要换一个重儿子就好了。事实上，并不需要保证新的儿子一定是重量最大的儿子，就算是随机儿子也能保证均摊复杂度正确。这样的剖法有时也被称作实链剖分。

但是这样用线段树维护链上信息就会很麻烦，因为链改了`dfn`就不一定连续了。于是我们考虑用splay来维护。注意这里splay不能用treap代替，因为我们会需要利用`splay`操作把平衡树的某个节点提到这棵树的根上的性质来实现实链与虚链的切换。

于是我们就得到了大名鼎鼎的LCT。他可以实现包括动态LCA、动态树上路径维护、可删边的并查集等等功能，而这些功能几乎只有以LCT为代表的动态树结构才能实现。

当然，他也有缺点，比如不太擅长维护子树信息（想想为什么）。和他功能相似的用来处理动态树上问题的还有一个叫ETT的东西，这个和LCT恰好相反，他更擅长维护子树信息而不太好处理路径信息。

由于他们的代码很繁琐且常数较大，并且考场上几乎用不到，就不贴代码了，仅作了解即可。
