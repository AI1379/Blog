---
title: 关于BST：你所需要知道的一切
date: 2022-09-17 16:43:27
mathjax: true
tags:
 - 数据结构
 - 树形结构
categories:
 - DataStructure
---

## 什么是二叉搜索树？

讲二叉搜索树之前，首先得知道他解决了什么问题：

> 1. 查找数列第k大
> 2. 查询一个数的排名rank
> 3. 插入一个数
> 4. 删除一个数

显然，一个每次操作后都`sort`一遍的数组就可以解决这个问题。但是这显然不是我们想要的答案：他太慢了。

那么有什么快一些的方案呢？有，就是**二叉搜索树 Binary Search Tree**。

首先，顾名思义，二叉搜索树首先是一棵二叉树。

那么二叉搜索树BST又有什么和普通二叉树不一样的性质，使得他可以“搜索”的呢？

我们可以先回顾一个有点相似的东西叫做堆。我们知道，堆是一棵完全二叉树，他满足每一个节点都比他的所有儿子要大（大根堆）。依靠这个性质，我们可以很轻松地实现求一个序列中最大的数，并且实现快速的增和删操作。

但是，很显然，仅仅依靠堆我们是没办法实现上面的操作的。这是由于堆的性质不够强：他只要求父亲比儿子大，却并没有规定左右子树之间的大小关系。但是我们可以修改一下这颗树的性质，改成这样：每一个节点，左子树都小于自己，右子树都大于自己。——这就是BST了。

容易发现，BST的中序遍历，就是排序后的原序列。我们也可以从根节点开始，递归的向左右子树搜索，实现查询。

## 平衡二叉树

朴素的二叉搜索树是有很大的缺点的：他有可能退化成一条链。这个时候，他的时间复杂度就不再优美了。

怎么解决这个问题呢？观察可以发现，当一棵BST左右较为平衡时，他的性能更好。这个时候我们可以保证$O(\log n)$的复杂度。平衡树所做的事情，就是把不那么平衡的BST，变成一棵平衡的BST。下面就是几种常见的平衡树的实现。

### Rotate 旋转操作

在讲平衡树之前，我们需要先讲一个大部分平衡树都需要的功能：旋转。旋转包括了左旋和右旋。



### Splay

其实我个人不认为Splay算是真的平衡树，因为他其实并不是很平衡。但是他通过一个叫做`splay`的操作，巧妙的保证了均摊复杂度，因此一般也把它算作平衡树。

`splay`操作做了一个事情：把一个节点旋转到了根。在OI中，我们还经常扩展这个操作，让他实现把一个节点旋转到作为另一个节点的儿子。

我们首先定义一个函数`which(x)`，表示一个点是自己父亲的左子还是右子：

```cpp
bool which(ll x) { return x == splay[splay[x].par].son[1]; }
```

`splay`的旋转一共有六种（其实是三种）情况：父节点是根，自己、自己的父亲、自己的祖父三点共线和三点不共线。这三类各包含两种情况：自己是左子和自己是右子。

用Splay实现的线段树板子代码：

```cpp
#include <bits/stdc++.h>
using namespace std;
#define ll long long
#define pll pair<ll, ll>
const ll MAXN = 100010;
class Splay {
  struct Node {
    ll son[2], par, val, sum, size, tag;
  };
  array<Node, MAXN> splay;
  ll rt, tot = 0, size;
  bool which(ll x) { return x == splay[splay[x].par].son[1]; }
#define lson splay[p].son[0]
#define rson splay[p].son[1]
  void pushup(ll p) {
    splay[p].size = splay[lson].size + splay[rson].size + 1;
    splay[p].sum = splay[lson].sum + splay[rson].sum + splay[p].val;
  }
  void pushdown(ll p) {
    if (splay[p].tag) {
      splay[lson].sum += splay[lson].size * splay[p].tag;
      splay[rson].sum += splay[rson].size * splay[p].tag;
      splay[lson].val += splay[p].tag;
      splay[rson].val += splay[p].tag;
      splay[lson].tag += splay[p].tag;
      splay[rson].tag += splay[p].tag;
      splay[p].tag = 0;
    }
  }
  void rotate(ll cur) {
    auto p = splay[cur].par, gp = splay[p].par;
    bool chkc = which(cur), chkp = which(p);
    pushdown(gp);
    pushdown(p);
    splay[p].son[chkc] = splay[cur].son[chkc ^ 1];
    if (splay[cur].son[chkc ^ 1])
      splay[splay[cur].son[chkc ^ 1]].par = p;
    splay[cur].son[chkc ^ 1] = p;
    splay[p].par = cur;
    splay[cur].par = gp;
    if (gp)
      splay[gp].son[chkp] = cur;
    pushup(cur);
    pushup(p);
  }
  void spl(ll cur, ll fa) {
    for (auto p = splay[cur].par; p && p != fa; p = splay[cur].par) {
      if (splay[p].par != fa)
        rotate(which(cur) ^ which(p) ? cur : p);
      rotate(cur);
    }
    if (fa == 0)
      rt = cur;
  }
  ll getbyrank(ll rk) {
    ll p = rt;
    while (true) {
      pushdown(p);
      if (rk <= splay[lson].size) {
        p = lson;
      } else {
        rk -= (splay[lson].size + 1);
        if (rk == 0)
          return p;
        p = rson;
      }
    }
  }
  ll buildimpl(const array<ll, MAXN> &orig, ll l, ll r, ll pa) {
    if (l > r)
      return 0;
    ll mid = (l + r) / 2, p = ++tot;
    splay[p].par = pa;
    splay[p].val = orig[mid];
    lson = buildimpl(orig, l, mid - 1, p);
    rson = buildimpl(orig, mid + 1, r, p);
    pushup(p);
    return p;
  }
  void modifynode(ll p, ll d) {
    splay[p].sum += splay[p].size * d;
    splay[p].val += d;
    splay[p].tag += d;
  }

public:
  void build(ll x, const array<ll, MAXN> &orig) {
    size = x;
    rt = buildimpl(orig, 1, x, 0);
  }
  void modify(ll l, ll r, ll diff) {
    if (l != 1 && r != size) {
      auto lptr = getbyrank(l - 1), rptr = getbyrank(r + 1);
      spl(lptr, 0);
      spl(rptr, lptr);
      modifynode(splay[splay[rt].son[1]].son[0], diff);
    } else if (l == 1 && r == size) {
      modifynode(rt, diff);
    } else if (l == 1) {
      auto ptr = getbyrank(r + 1);
      spl(ptr, 0);
      modifynode(splay[rt].son[0], diff);
    } else {
      auto ptr = getbyrank(l - 1);
      spl(ptr, 0);
      modifynode(splay[rt].son[1], diff);
    }
  }
  ll query(ll l, ll r) {
    if (l != 1 && r != size) {
      auto lptr = getbyrank(l - 1), rptr = getbyrank(r + 1);
      spl(lptr, 0);
      spl(rptr, lptr);
      return splay[splay[splay[rt].son[1]].son[0]].sum;
    } else if (l == 1 && r == size) {
      return splay[rt].sum;
    } else if (l == 1) {
      auto ptr = getbyrank(r + 1);
      spl(ptr, 0);
      return splay[splay[rt].son[0]].sum;
    } else {
      auto ptr = getbyrank(l - 1);
      spl(ptr, 0);
      return splay[splay[rt].son[1]].sum;
    }
  }
#undef lson
#undef rson
};
ll n, q;
array<ll, MAXN> orig;
Splay splay;
int main() {
  ll opt, u, v, w;
  cin >> n >> q;
  for (int i = 1; i <= n; ++i) {
    cin >> orig[i];
  }
  splay.build(n, orig);
  while (q--) {
    cin >> opt;
    if (opt == 1) {
      cin >> u >> v >> w;
      splay.modify(u, v, w);
    } else {
      cin >> u >> v;
      cout << splay.query(u, v) << endl;
    }
  }
  return 0;
}
```

### Treap

#### 旋转Treap

我们知道一个事情：如果一棵普通的BST插入随机数据，他是期望平衡的。但是很悲伤的事情是，我们不能确保插入的一定是随机顺序。这个时候就可以有一个很简单的想法：人工添加随机化。但是由于平衡树要求在线操作，因此我们显然不可能把所有插入数据读入之后`shuffle`插入。Treap的思想，则是给每个节点附上一个随机权值，使得键值满足BST性质而权值满足堆性质。



#### 无旋Treap (aka. FHQTreap)

无旋Treap，顾名思义，不需要旋转操作。这是极少数不用旋转的平衡树之一。

正如普通的平衡树中的`rotate`，无旋Treap中也有他的基本操作`split`和`merge`。通过这两种操作，可以十分轻松地实现平衡树的所有操作。

分裂有两种：`split_by_rank`和`split_by_value`。顾名思义，一个是按排名分裂，一个是按值分裂。他们的写法十分相似。这里以按值分裂为例。

`split`操作，接受两个参数`cur`和`val`，返回两颗treap的根，表示将以`cur`为根的treap分成两颗新的treap，其中一部分的值都满足`v<=val`而另一部分的值都满足`val<v`。也就是，以`val`为界将整棵树分成两半。

考虑递归。由于BST的性质，`cur`左子的大小一定小于`cur->val`而右子一定大于`cur->val`。于是，我们只需考虑`cur->val`与`val`的大小即可。

如果`cur->val <= val`，那么`cur`及其左子树一定属于分裂后的左子树，于是我们递归的处理右子树，对右子树进行`split`操作，得到两棵树`lpart`和`rpart`，那么根据`split`的定义，`lpart`中的一定`<=val`而`rpart`中的一定`>val`。而由于BST的性质，`lpart`中的节点有一定会大于`cur`。于是，只需将`cur`的右儿子变成`lpart`，那么`cur`和`rpart`便是分裂得到的两颗新树的根。`cur->val > val`的操作同理。

<img alt="" src="https://static.cdn.menci.xyz/oi-wiki/ds/images/treap-none-rot-split-by-val.svg?h=Z4dwNw">

`merge`操作，接受两个treap `u`和`v`，且 **`v`中所有节点都比`u`中所有节点大** ，返回一颗新的treap。

思路与`split`类似。不妨假设`u`的权值比`v`小，也就是说合并后`v`是`u`的子节点（如果是按照小根堆的话）。由于`v`中所有节点都比`u`中所有节点大，所以只需将`u`的右儿子与`v`合并，然后`u`节点新的右儿子即可。

有了`split`和`merge`，那么平衡树的其他操作都很好实现。

插入一个新节点，只需通过两次`split`操作，将整棵树分成三部分：小于`val`，等于`val`和大于`val`（当然等于的那部分可能是空的）。如果等于的部分是空的，那么新建节点，否则`++count`即可。修改完再`merge`成一棵树，就完成了整个操作。删除操作同理。

`getKth`和`getRank`也相当简单。只需`split_by_rank`和`split_by_value`即可。`getPrev`和`getNext`也同理。

当然，无旋treap也是一颗bst，因此同样可以使用bst上的方法进行这些操作。这里的写法是按普通bst写法写的。

代码：

```cpp
#include <bits/stdc++.h>
using namespace std;
#define ll long long
#define pll pair<ll, ll>
static constexpr const ll MAXN = 100010;
static constexpr const ll INF = 1ll << (sizeof(ll) * 8 - 2);
struct Node {
  ll lson, rson;
  ll size, data, pri, count;
};
static ll n;
static ll root, endt;
static Node treap[MAXN];
void pushup(ll p) {
  treap[p].size =
      treap[treap[p].lson].size + treap[treap[p].rson].size + treap[p].count;
}
pll split(ll p, ll val) {
  pll tmp;
  if (p == 0)
    return {0, 0};
  if (treap[p].data <= val) {
    tmp = split(treap[p].rson, val);
    treap[p].rson = tmp.first;
    pushup(p);
    return {p, tmp.second};
  } else {
    tmp = split(treap[p].lson, val);
    treap[p].lson = tmp.second;
    pushup(p);
    return {tmp.first, p};
  }
}
ll merge(ll u, ll v) {
  if (u == 0 || v == 0)
    return u ^ v;
  if (treap[u].pri <= treap[v].pri) {
    treap[u].rson = merge(treap[u].rson, v);
    pushup(u);
    return u;
  } else {
    treap[v].lson = merge(u, treap[v].lson);
    pushup(v);
    return v;
  }
}
void ins(ll val) {
  static random_device rd;
  static mt19937_64 mt(rd());
  static uniform_int_distribution<ll> dist(1ll, MAXN);
  static pll tmp, tmpl;
  tmp = split(root, val);
  tmpl = split(tmp.first, val - 1);
  if (tmpl.second == 0) {
    treap[++endt].data = val;
    treap[endt].count = 1;
    treap[endt].size = 1;
    treap[endt].pri = dist(mt);
    tmpl.second = endt;
  } else {
    ++treap[tmpl.second].size;
    ++treap[tmpl.second].count;
  }
  root = merge(merge(tmpl.first, tmpl.second), tmp.second);
}
void del(ll val) {
  static pll tmp, tmpl;
  tmp = split(root, val);
  tmpl = split(tmp.first, val - 1);
  if (treap[tmpl.second].count > 1) {
    --treap[tmpl.second].count;
    --treap[tmpl.second].size;
    tmpl.first = merge(tmpl.first, tmpl.second);
  }
  root = merge(tmpl.first, tmp.second);
}
ll getRank(ll p, ll val) {
  if (treap[p].data == val)
    return treap[treap[p].lson].size + 1;
  return treap[p].data > val ? getRank(treap[p].lson, val)
                             : getRank(treap[p].rson, val) +
                                   treap[treap[p].lson].size + treap[p].count;
}
ll getKth(ll p, ll rk) {
  if (treap[treap[p].lson].size >= rk)
    return getKth(treap[p].lson, rk);
  if (treap[treap[p].lson].size + treap[p].count >= rk)
    return treap[p].data;
  return getKth(treap[p].rson, rk - treap[treap[p].lson].size - treap[p].count);
}
ll getPrev(ll val) {
  static ll p, res;
  p = root;
  res = -INF;
  while (p) {
    if (treap[p].data == val) {
      p = treap[p].lson;
      if (p) {
        while (treap[p].rson)
          p = treap[p].rson;
        res = treap[p].data;
      }
      break;
    }
    if (treap[p].data < val)
      res = max(res, treap[p].data);
    p = treap[p].data < val ? treap[p].rson : treap[p].lson;
  }
  return res;
}
ll getNext(ll val) {
  static ll p, res;
  p = root;
  res = INF;
  while (p) {
    if (treap[p].data == val) {
      p = treap[p].rson;
      if (p) {
        while (treap[p].lson)
          p = treap[p].lson;
        res = treap[p].data;
      }
      break;
    }
    if (treap[p].data > val)
      res = min(res, treap[p].data);
    p = treap[p].data < val ? treap[p].rson : treap[p].lson;
  }
  return res;
}
static ll opt, x;
#define pushcase(x, y)                                                         \
  case x: {                                                                    \
    y;                                                                         \
    break;                                                                     \
  }
int main() {
  cin >> n;
  while (n--) {
    cin >> opt >> x;
    switch (opt) {
      pushcase(1, ins(x));
      pushcase(2, del(x));
      pushcase(3, cout << getRank(root, x) << endl);
      pushcase(4, cout << getKth(root, x) << endl);
      pushcase(5, cout << getPrev(x) << endl);
      pushcase(6, cout << getNext(x) << endl);
    }
  }
  return 0;
}

```

区间修改求和：

```cpp
#include <bits/stdc++.h>
#include <cstddef>
#include <random>
using namespace std;
#define ll long long
#define pll pair<ll, ll>
const ll MAXN = 100010;
class Treap {
  struct Node {
    ll lson, rson, val, tag, sum, size, pri;
  };
  array<Node, MAXN> treap;
  ll tot = 0, rt, size;
  void pushup(ll p) {
    treap[p].size = treap[treap[p].lson].size + treap[treap[p].rson].size + 1;
    treap[p].sum =
        treap[treap[p].lson].sum + treap[treap[p].rson].sum + treap[p].val;
  }
  void pushdown(ll p) {
    if (treap[p].tag) {
      treap[treap[p].lson].sum += treap[treap[p].lson].size * treap[p].tag;
      treap[treap[p].rson].sum += treap[treap[p].rson].size * treap[p].tag;
      treap[treap[p].lson].val += treap[p].tag;
      treap[treap[p].rson].val += treap[p].tag;
      treap[treap[p].lson].tag += treap[p].tag;
      treap[treap[p].rson].tag += treap[p].tag;
      treap[p].tag = 0;
    }
  }
  pll split(ll cur, ll rk) {
    if (cur == 0)
      return {0, 0};
    if (treap[treap[cur].lson].size + 1 <= rk) {
      pushdown(cur);
      auto tmp = split(treap[cur].rson, rk - treap[treap[cur].lson].size - 1);
      treap[cur].rson = tmp.first;
      pushup(cur);
      return {cur, tmp.second};
    } else {
      pushdown(cur);
      auto tmp = split(treap[cur].lson, rk);
      treap[cur].lson = tmp.second;
      pushup(cur);
      return {tmp.first, cur};
    }
  }
  ll merge(ll u, ll v) {
    if (u == 0 || v == 0)
      return u ^ v;
    if (treap[u].pri < treap[v].pri) {
      pushdown(v);
      treap[v].lson = merge(u, treap[v].lson);
      pushup(v);
      return v;
    } else {
      pushdown(u);
      treap[u].rson = merge(treap[u].rson, v);
      pushup(u);
      return u;
    }
  }
  ll build(const array<ll, MAXN> &orig, ll p) {
    static std::random_device rd;
    static std::mt19937 rng(rd());
    static std::uniform_int_distribution<ll> dist(1, MAXN);
    if (p > size)
      return 0;
    ll cur = ++tot;
    treap[cur].pri = dist(rng);
    treap[cur].val = orig[p];
    pushup(cur);
    return merge(cur, build(orig, p + 1));
  }

public:
  void build(ll x, const array<ll, MAXN> &orig) {
    size = x;
    rt = build(orig, 1);
  }
  void modify(ll l, ll r, ll diff) {
    auto tmp = split(rt, r);
    auto tmpl = split(tmp.first, l - 1);
    treap[tmpl.second].sum += treap[tmpl.second].size * diff;
    treap[tmpl.second].val += diff;
    treap[tmpl.second].tag += diff;
    rt = merge(merge(tmpl.first, tmpl.second), tmp.second);
  }
  ll query(ll l, ll r) {
    auto tmp = split(rt, r);
    auto tmpl = split(tmp.first, l - 1);
    auto res = treap[tmpl.second].sum;
    rt = merge(merge(tmpl.first, tmpl.second), tmp.second);
    return res;
  }
};
ll n, q;
array<ll, MAXN> orig;
Treap splay;
int main() {
  ll opt, u, v, w;
  cin >> n >> q;
  for (int i = 1; i <= n; ++i) {
    cin >> orig[i];
  }
  splay.build(n, orig);
  while (q--) {
    cin >> opt;
    if (opt == 1) {
      cin >> u >> v >> w;
      splay.modify(u, v, w);
    } else {
      cin >> u >> v;
      cout << splay.query(u, v) << endl;
    }
  }
  return 0;
}
```

#### 可持久化Treap / 可持久化平衡树

既然有了好写跑得也快的有旋Treap，为什么还要FHQTreap呢？

因为FHQTreap**可以持久化**。

我们先来思考一下为什么普通的treap（以及其他所有基于旋转的BST和替罪羊树）不能实现持久化。先来回顾一下主席树的写法：每次修改操作，对修改路径上的每一个点增加一个副本，然后修改副本。如果使用基于旋转的平衡树，有一个很大的问题在于父子关系会发生交换，这让他们的可持久化变得困难。而FHQTreap没有旋转操作，父子关系是不会发生交换的，因此FHQTreap可以实现持久化。至于替罪羊树，重构的机制使得持久化彻底不可能。

那么如何实现呢？我们还是参照主席树的写法：在每一次修改操作中把修改的节点拷贝一份，然后在拷贝出来的节点里面进行修改。但是不像主席树，FHQTreap有两个基本操作：`merge`和`split`。事实上，如果相同的键值存在同一个节点里面，那么只要在`split`中复制即可，因为`split`后必然跟着一个`merge`。

### AVL

AVL是一种强平衡的平衡树。他也是最早被发明的平衡树之一。

我们回顾一下为什么普通BST会出现$O(n)$的最坏复杂度：因为不平衡。那怎样才能平衡呢？有一个简单且暴力的做法：记录每一个节点子树深度，然后通过旋转使得任意一个节点的左右儿子深度差的绝对值至多为1。这就是AVL的基本思想。

### RBTree 红黑树

红黑树同样是一棵真正的平衡树，只是它的平衡并不是真的平衡，而是黑点平衡。具体而言就是，对于每一个叶子节点，从根到他的简单路径上经过的黑点数都相同。事实上，从某些角度来说红黑树是2-3树的一个变体。如果我把每一个黑节点和他儿子里的红节点合并，他就能成为一棵2-3树。

但是他又什么用呢？快。红黑树插入删除的常数是极小的，因为可以证明红黑树每一次插入删除所需要的旋转次数的上限是一个很小的常量。这也是为什么红黑树虽然不那么的平衡，但是却是最快的平衡树之一，尤其是在数据规模极大的情况下。这也是为什么大多数语言标准库中的基于平衡树的`map`都是用红黑树实现的。这里唯一的例外是Rust，他的`map`默认是基于B树的。具体原因可以在Rust官网查阅。

形式化地说，红黑树是这样的一颗平衡搜索树：

1. 每个节点有红黑两种颜色
2. 每个红点的儿子都是黑点
3. 从根到每一片叶子，经过的黑点数一样
4. NIL节点为黑色

### 平衡树的应用

**这里仅指算法竞赛中常见平衡树，如无旋Treap和Splay**

**事实上，并不是所有的平衡树都能有这些功能。例如红黑树就很难实现区间操作**

#### 区间操作：替代线段树

平衡树不仅仅能支持线段树的所有操作，而且还支持一个新操作：**区间移位和翻转**。

首先考虑线段树最重要的两个操作：区间加和区间查。

在每个节点上附加一个`sum`，表示以这个节点为根的子树的和，再附加一个`tag`作为懒标记。

再定义两个操作`pushup`和`pushdown`，分别表示维护节点信息和下传标记。那么，和线段树一样，最重要的部分就是设计这两个操作。当然，如果只是简单的区间和问题，那么并不需要复杂的设计。

接下来，考虑如何处理区间。

对于无旋treap，这十分简单。进行两次`split_by_rank`，即可把一棵完整的treap分成三部分，中间的一部分即要修改的区间。然后直接修改中间部分的根节点的`tag`，维护好`sum`和`val`即可。

对于splay，这也十分简单。首先，找到排名为`l-1`和`r+1`的两个节点。将`l-1`旋转到根，然后将`r+1`旋转到根的儿子。这个时候根节点就是`l-1`，其右儿子就是`r+1`，于是显然，根的右儿子的左子树就包括了整个要处理的区间。同样修改`tag`维护`sum`和`val`即可。

接着，我们考虑要在什么时候进行`pushup`和`pushdown`。这也很简单：要修改节点或修改子树的时候，就先`pushdown`下传标记，修改完后再`pushup`维护我自己。

然后我们考虑线段树所不能进行的操作：区间平移和区间翻转。

首先考虑区间平移。用无旋treap实现区间平移是很自然的：只需把要平移的区间`split`出来，作为一颗独立的treap，然后将剩下的部分从终点处`split`开，最后重新`merge`成一棵树即可。用splay实现同样很自然：只需如发炮制，将移动区间的左边和右边`splay`上来，然后断掉这个子树，接着把终点`splay`到根，把断下来的子树重新拼回去就好了。

然后考虑区间反转。事实上这可以看作特殊的区间修改。定义`tag`表示这个节点的左右子树是否互换，那么`pushdown`就是将两个儿子的`tag^=1`，然后交换两个儿子。

区间平移和区间翻转是两个相当有用的东西。如果配合树链剖分或是欧拉回路，那么即可使用splay或treap维护动态图上问题，例如动态图联通性和动态最小生成树。这就是大名鼎鼎的LCT和ETT了。当然这已经超出了所述范围，就不赘述了。

## B/B+ 树

B树严格来说已经不是二叉搜索树了，因为他是多叉的。而B+树甚至不是一棵树了，因为B+树在相邻节点之间有连边。

## 笛卡尔树

笛卡尔树是一种二叉搜索树，每个节点上存有一个二元组`(a, b)`，对于`a`满足BST性质，对于`b`满足堆性质。

发现了吗？这不就是Treap吗！事实上，Treap确实就是一种笛卡尔树。

### 笛卡尔树的应用

著名的Four-Russian算法，可以在 $O(n) - O(1)$ 的时间复杂度里完成RMQ问题。当然，他相当的复杂，有兴趣的话可以参考CSP-S 2021**初赛**最后一题，自行查阅资料。
