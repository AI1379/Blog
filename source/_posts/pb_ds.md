---
title: pbds食用教程
date: 2022-08-07 16:28:01
mathjax: true
tags:
  - 数据结构
  - STL
categories:
  - DataStructure
---

> 平板电视：比 STL 还 STL
>
> ——来自 luogu 某佬

## Intro 前言

`pb_ds`的主体部分在`__gnu_pb_ds`命名空间下，使用前需要`using namespace __gnu_pbds`

> WARNING: NOIP 评测的时候可能不允许使用`using namespace __gnu_pbds`。

头文件如下：

```cpp
#include <ext/pb_ds/assoc_container.hpp> // 各种类的定义 必须引用
#include <ext/pb_ds/tree_policy.hpp> // tree
#include <ext/pb_ds/hash_policy.hpp> // hash
#include <ext/pb_ds/trie_policy.hpp> // trie
#include <ext/pb_ds/priority_queue.hpp> // priority_queue
```

或者

```cpp
#include <bits/extc++.h>
// 类似bits/stdc++.h
// 据称这个文件在dev-c++里面会报找不到文件 因此建议上面的写法
```

**WARNING: pb_ds 是以 libstdc++为标准库的编译器中专属的库 仅 gcc 可用**

为什么要`pb_ds`？

它封装了可并堆，字典树，平衡树等一堆常用且难搞的东西，极大程度减少码量，甚至可能可以降低时间复杂度。

而且，就算考场上不敢使用 pbds，我们也可以用它来进行对拍，或者更加硬核，直接把 pbds 中用到的部分的源码复制到提交文件中（但是要注意提交文件大小），更可以参照着 pbds 自己重写一遍里面的数据结构。总而言之，就算不敢直接调用 pbds 库，它依然是十分有用的。

## hash 哈希表

pbds 中有两个哈希表：

```cpp
cc_hash_table<Key, Value>;
gp_hash_table<Key, Value>;
```

其中`cc_hash_table`是拉链法，`gp_hash_table`是探测法。探测法相对来说会快一些。

怎么用？他们的用法和`std::map`完全相同。

那他们有什么作用呢？

注意到，`std::map`是依靠平衡树实现的，总复杂度是$\Theta(n\log n)$，而哈希表的总复杂度是$\Theta(n)$。

> C++11 开始引入了`std::unordered_map`，同样是基于哈希表实现的。实测下来，开了 O2 的情况下，STL 相对会较快一些。所以如果比赛开了 C++11 和 O2 的话并没有必要使用 pbds hash

### 例题：P1333 瑞瑞的木棍

[题目链接](https://www.luogu.com.cn/problem/P1333)

这个题是一个图论题，基本上就是欧拉回路的模板题。问题在于它给边的时候并不是给两端的编号而是两个字符串。

一种做法是按照正常的字符串哈希让每个颜色映射到一个整数上然后做。事实上在C++11后STL里面也封装了哈希函数，但是它的值域很大，所以一般在OI中不会用他。

另外就是用`trie`或哈希表，然后正常图论该怎么做怎么做就完了。

代码来自 https://www.luogu.com.cn/blog/Chanis/gnu-pbds 。略有修改。

```cpp
#include <bits/stdc++.h>
#include <ext/pb_ds/assoc_container.hpp>
#include <ext/pb_ds/hash_policy.hpp>

using namespace std;
const int MAXN = 250050;
char l[15], r[15];
int e, f[MAXN];
bool in[MAXN];
__gnu_pbds::gp_hash_table<string, int> G; // hash_table定义
int find(int x) { return x == f[x] ? x : f[x] = find(f[x]); }
int main() {
  for (int i = 1; i < MAXN; ++i) {
    f[i] = i;
  }
  while (scanf("%s%s", l, r) != EOF) {
    if (!G[l])
      G[l] = ++e, f[e] = e;
    if (!G[r])
      G[r] = ++e, f[e] = e;
    if (e > 250010) {
      printf("Impossible\n");
      return 0;
    }
    in[G[l]] ^= 1;
    in[G[r]] ^= 1;
    f[find(G[l])] = find(G[r]);
  }
  int flag = 0;
  {
    for (int i = 1; i <= e; ++i)
      if (in[i]) {
        if (flag == 2) {
          printf("Impossible\n");
          return 0;
        } else
          ++flag;
      }
  }
  int father = find(1);
  for (int i = 2; i <= e; ++i) {
    if (find(i) ^ father) {
      printf("Impossible\n");
      return 0;
    }
  }
  printf("Possible\n");
  return 0;
}
```

## trie 字典树

模板定义：

```cpp
template <
    typename Key,
    typename Mapped,
    class E_Access_Traits,
    typename Tag = pat_trie_tag,
    template <
        typename Node_CItr,
        typename Node_Itr,
        class E_Access_Traits,
        typename Alloc>
    class Node_Update = null_trie_node_update,
    typename Alloc = std::allocator<char>>
class trie;
```

容易发现，`trie`和`tree`的模板几乎是一模一样的（除了`E_Access_Traits`）。当然，由于比赛中实际使用`trie`远远少于`tree`，因此我们并不会像平衡树那样细致地讲解使用和自定义。事实上，`trie`和`tree`自定义的操作几乎是一样的。

通常我们使用的`trie`是这样的：

```cpp
using Trie = __gnu_pbds::trie<string, null_type,
                              trie_string_access_traits<>,
                              pat_trie_tag,
                              trie_prefix_search_node_update>;
Trie trie;
```

然后我们可以有这些操作：

```cpp
trie.insert(str);
trie.erase(str);
trie.join(another_trie);

// 遍历某一个前缀的所有字符串
pair<Trie::iterator, Trie::iterator> range = trie.prefix_range(prefix);
for(auto it = range.first; it != range.second; ++it) {
  cout << *it << endl;
}
```

由于`trie`的实际应用很少，并且实现并不困难，因此我们就不过多阐述它了。

## tree 平衡树

平衡树的实现有三个：`rb_tree_tag`，`splay_tree_tag`和`ov_tree_tag`。**除了红黑树都不建议使用，除非是一道 Splay 维护区间的问题而又不会写 Splay。**

模板参数如下：

```cpp
template <
    typename Key,
    typename Mapped,
    typename Cmp_Fn = std::less<Key>,
    typename Tag = rb_tree_tag,
    template <
        typename Node_CItr,
        typename Node_Itr,
        typename Cmp_Fn_,
        typename _Alloc_ >
    class Node_Update = null_node_update,
    typename Alloc = std::allocator<char> >
class tree;
```

其中：

- `Key`是键类型，也就是平衡树中存储的类型

- `Mapped`映射类型，通常为`null_type`表示无映射

- `Cmp_Fn`比较函数

- `Tag`实现方式

- `Node_Update`节点更新方式 通常使用`tree_order_statistics_node_update`

- `Alloc`内存分配器

其中`Node_Update`是 pbds tree 最重要的一部分，也是自定义 pbds 平衡树行为的最重要的手段

在讲解 RB-Tree 和 Splay-Tree 的用法前，我们看一下`tree`的成员函数：

- `find` 查找权值对应节点

- `insert` 插入新节点

- `erase` 删除迭代器所指向的节点

- `lower_bound` 第一个大于等于某元素的迭代器

- `upper_bound` 第一个大于某元素的迭代器

- `join` 合并 合并后另一棵树会变成空

- `split(const Key &r, RBTree &other)` 分裂，将大于r的元素放进other里（如果使用的是greater则是小于r）

- `node_begin` 得到根节点的迭代器

- `node_end` 得到最后一个叶节点后的一个空节点，一般表示不存在

### RB-Tree 红黑树

红黑树的基本原理超出了我们的范围。我们所需要知道的只是：这是一种跑的飞快且极为难写难调的平衡二叉树。

平衡树的用途通常比较固定（除了 Splay），一般就这么几种：

1. 插入一个数

2. 删除一个数

3. 查询一个数的排名

4. 查询排名为 k 的数

5. 求一个数的前驱后继

[【模板】普通平衡树](https://www.luogu.com.cn/problem/P3369)

代码：

```cpp
// https://www.luogu.com.cn/problem/P3369
#include <bits/stdc++.h>
#include <ext/pb_ds/assoc_container.hpp>
#include <ext/pb_ds/tree_policy.hpp>
using namespace std;
#define ll long long
#define pll pair<ll, ll>
const ll INF = 1ll << (sizeof(ll) * 8 - 2);
__gnu_pbds::tree<pll, __gnu_pbds::null_type, std::less<pll>,
                 __gnu_pbds::rb_tree_tag,
                 __gnu_pbds::tree_order_statistics_node_update>
    rbt;
ll n;
int main() {
  ll opt, k;
  decltype(rbt)::point_iterator ptr;
  ios::sync_with_stdio(false);
  cin.tie(nullptr);
  cout.tie(nullptr);
  cin >> n;
  for (int i = 1; i <= n; i++) {
    cin >> opt >> k;
    switch (opt) {
    case 1:
      rbt.insert(make_pair(k, i));
      break;
    case 2:
      rbt.erase(rbt.lower_bound(make_pair(k, 0)));
      break;
    case 3:
      cout << rbt.order_of_key(make_pair(k, 0)) + 1 << endl;
      break;
    case 4:
      cout << (rbt.find_by_order(k - 1)->first) << endl;
      break;
    case 5:
      ptr = rbt.lower_bound(make_pair(k, 0));
      --ptr;
      cout << (ptr->first) << endl;
      break;
    case 6:
      ptr = rbt.upper_bound(make_pair(k, INF));
      cout << (ptr->first) << endl;
      break;
    }
  }
  return 0;
}
```

### Node_Update 的实现

> WARNING: 前方需要一些 C++高级特性 看不懂可以暂时跳过

首先我们要知道：`Node_Update`是`tree`的公有基类。也就是说，`Node_Update`中的所有成员方法都会在`tree`中公开。`Node_Update`是这样的一个模板类：

```cpp
template <
    typename Node_CItr,
    typename Node_Itr,
    typename Cmp_Fn,
    typename Alloc >
class Node_Update;
```

在 pbds 平衡树的每一个节点中，都带着一些附加的**元数据**，而这些数据的类型则是在`Node_Update`中定义的。在一个自定义的`Node_Update`中，我们至少需要定义两个东西：

- `metadata_type` 元数据的类型

- `void operator()(Node_Itr it, Node_CItr end_it)` 修改树的时候调用的函数

其中`Node_Itr`是指向树中结点的一个迭代器，有这些操作：

- `get_l_child()` 返回左子，没有左子则返回`node_end()`

- `get_r_child() `同上

- `get_metadata()` 获取元数据，类型是我们定义的`metadata_type`

- `**it` 访问迭代器所指向的节点中的值

我们来举个例子。假如说，我现在想要维护一棵红黑树，同时维护每一颗子树的大小。那么我们可以这样：

```cpp
struct NodeUpdateSize {
public:
  typedef ll metadata_type;
  void operator()(Node_Itr it, Node_CItr end_it) {
    Node_Itr l = it.get_l_child();
    Node_Itr r = it.get_r_child();
    int left = 0, right = 0;
    if (l != end_it)
      left = l.get_metadata();
    if (r != end_it)
      right = r.get_metadata();
    const_cast<ll &>(it.get_metadata()) = left + right + 1;
  }
};
```

注意`it.get_metadata()`返回的是一个`const reference`，是不可以直接赋值的，需要用`const_cast<metadata_type &>`来去掉`const`修饰。

更复杂一些，我们还可以实现查找某一个值的排名。要在平衡树上查找排名，我们得要先从树根开始往下找。很不幸的是，在`Node_Update`里我们还不能直接访问`tree`里的方法，但是我们可以通过`virtual`的方式先生成虚函数从而实现调用`tree`中方法的目的。

```cpp
struct MyNodeUpdate {
public:
  typedef ll metadata_type;
  void operator()(Node_Itr it, Node_CItr end_it) {
    Node_Itr l = it.get_l_child();
    Node_Itr r = it.get_r_child();
    int left = 0, right = 0;
    if (l != end_it)
      left = l.get_metadata();
    if (r != end_it)
      right = r.get_metadata();
    const_cast<ll &>(it.get_metadata()) = left + right + 1;
  }
  virtual Node_CItr node_begin() const = 0;
  virtual Node_CItr node_end() const = 0;
  ll order_of_key(ll x) {
    ll ans = 0;
    Node_CItr it = node_begin();
    while (it != node_end()) {
      Node_CItr l = it.get_l_child();
      Node_CItr r = it.get_r_child();
      if (Cmp_Fn()(x, **it))
        it = l;
      else {
        ans++;
        if(l != node_end()) ans += l.get_metadata();
        it = r;
      }
    }
    return ans;
  }
};
```

不过，`order_of_key`在`tree_order_statistics_node_update`中已经提供了，我们能不能通过继承这个类来实现复用呢？很遗憾，不行。因为`tree_order_statistics_node_update`中实现`order_of_key`和`find_by_order`需要使用`metadata`，而我们修改`metadata`后就无法继续维护原来的`metadata`导致失效。

### 例题：NOI2004郁闷的出纳员

[题目链接](https://www.luogu.com.cn/problem/P1486)

这个题目很容易让人想到平衡树，但是显然平衡树不能实现所有人的加和减操作。这个时候不妨换一个角度考虑：不直接修改所有人的工资，而是修改工资下界，然后记录当前工资下界与原始值的差，每次查询的时候减去这个差即可。

要注意的是，在平衡树里面放的类型不是`int`而是`pair<int, int>`，目的是为了防止重复的元素。这在平衡树的题目里是很常见的技巧。当然我们也可以维护`metadata`记录同样的数据出现的次数，但这显然会麻烦不少。

代码如下：

```cpp
#include <ext/pb_ds/assoc_container.hpp>
#include <ext/pb_ds/tree_policy.hpp>
#include <bits/stdc++.h>

using namespace std;
#define ll long long
#define pll pair<ll, ll>
ll n, m, delta = 0;
typedef __gnu_pbds::tree<pll, __gnu_pbds::null_type, greater<pll>,
                         __gnu_pbds::rb_tree_tag,
                         __gnu_pbds::tree_order_statistics_node_update>
    RBTree;
RBTree rbt, other;
int main() {
  char opt;
  ll k, ans = 0;
  cin >> n >> m;
  for (int i = 1; i <= n; i++) {
    cin >> opt >> k;
    if (opt == 'I') {
      k += delta;
      if (k >= m)
        rbt.insert(make_pair(k, i));
    } else if (opt == 'A') {
      m -= k;
      delta -= k;
    } else if (opt == 'S') {
      m += k;
      delta += k;
      rbt.split(make_pair(m, -1), other);
      ans += other.size();
      other.clear();
    } else {
      if (rbt.size() >= k) {
        cout << (rbt.find_by_order(k - 1)->first - delta) << endl;
      } else {
        cout << "-1" << endl;
      }
    }
  }
  cout << ans << endl;
  return 0;
}

```

## priority_queue 堆

> 注意：如果用`__gnu_pbds::priority_queue`那么建议不要引用万能头或者`using namespace __gnu_pbds`以免与`std::priority_queue`发生冲突导致 CE。

### 前置知识：如何自定义`priority_queue`的排序方式

~~其实这个前置知识并不会影响 pbds 堆的使用~~

我们都知道，对于一个结构体，我们不能直接把它丢进`priority_queue`，无论是 STL 还是 pbds。那么我们有两种方案：重载小于号，或者写一个**仿函数 functor**

仿函数听着玄乎，其实很简单，只是一个重载了括号运算符的结构体而已。

比方说，众所周知，有个东西叫`std::less`，它其实就是一个仿函数，实现也很简单：

```cpp
template<typename T>
struct less{
    bool operator()(const T& lhs,const T& rhs){
        return lhs<rhs;
    }
};
```

那么，我们同样也可以自己写一个`comp`，用来比较，就像这样：

```cpp
struct comp{
    bool operator()(int x, int y){
        return dis[x] > dis[y];
    }
};
```

使用的时候则这么用：

```cpp
std::priority_queue<int,std::vector<int>,comp> q;
```

于是就可以得到堆里面装了数组下标，以下标对应的`dis`值为关键字的小根堆

~~是不是非常简单~~

### 模板参数 & 成员函数

```cpp
template<
    typename ValueType,
    typename Comp = std::less<ValueType>,
    typename Tag = pairing_heap_tag,
    typename Alloc = std::allocator<char>>
class priority_queue;
```

其中：

- `ValueType`是堆里面的元素类型

- `Comp`是比较器，是一个 functor。注意 pbds 的堆和 STL 一样 传入 less 会得到大根堆

- `Tag`是堆的实现，总共有以下几种：

  - `pairing_heap_tag` 配对堆 最有用

  - `binary_heap_tag` 二叉堆

  - `binomial_heap_tag` 二项堆

  - `rc_binomial_heap_tag` 冗余计数二项堆

  - `thin_heap_tag` Tarjan 老爷子发明的一种 除了合并以外复杂度都和 Fibonacci 堆一样的堆

- `Alloc`是空间分配器，不用管他，省略即可

除了`pairing_heap_tag`以外，其他四个在 OI 中都不如`std::pirority_queue`。配对堆不仅优于 STL 的二叉堆，同时也优于`algorithm`头文件中的`make_heap`系列函数。因此，如果不做说明，下文所指的 pbds 堆都指配对堆。

五种 tag 都支持以下的操作：

- `push` 压入一个元素

- `pop` 弹出一个元素

- `top` 获得堆顶

- `size` 获得大小

- `empty` 获得是否为空

- `modify` 修改某一个位置的元素 并且重新调整结构

- `erase` 删除某一个位置的元素

- `join` 合并两个堆

### 复杂度

|                        | push                                    | pop                                     | modify                                                                            | erase                                   | join              |
| ---------------------- | --------------------------------------- | --------------------------------------- | --------------------------------------------------------------------------------- | --------------------------------------- | ----------------- |
| `std::priority_queue`  | 最坏$\Theta(n)$均摊$\Theta(\log n)$     | $\Theta(\log n)$                        | $\Theta(n\log n)$                                                                 | $\Theta(n\log n)$                       | $\Theta(n\log n)$ |
| `pairing_heap_tag`     | $O(1)$                                  | 最坏 $\Theta(n)$ 均摊 $\Theta(\log(n))$ | 最坏 $\Theta(n)$ 均摊 $\Theta(\log(n))$                                           | 最坏 $\Theta(n)$ 均摊 $\Theta(\log(n))$ | $O(1)$            |
| `binary_heap_tag`      | 最坏 $\Theta(n)$ 均摊 $\Theta(\log(n))$ | 最坏 $\Theta(n)$ 均摊 $\Theta(\log(n))$ | $\Theta(n)$                                                                       | $\Theta(n)$                             | $\Theta(n)$       |
| `binomial_heap_tag`    | 最坏 $\Theta(\log(n))$ 均摊 $O(1)$      | $\Theta(\log(n))$                       | $\Theta(\log(n))$                                                                 | $\Theta(\log(n))$                       | $\Theta(\log(n))$ |
| `rc_binomial_heap_tag` | $O(1)$                                  | $\Theta(\log(n))$                       | $\Theta(\log(n))$                                                                 | $\Theta(\log(n))$                       | $\Theta(\log(n))$ |
| `thin_heap_tag`        | $O(1)$                                  | 最坏 $\Theta(n)$ 均摊 $\Theta(\log(n))$ | 最坏 $\Theta(\log(n))$ 均摊 $O(1)$或者$\Theta(\log n)$ 这取决于修改是增加还是减少 | 最坏 $\Theta(n)$ 均摊 $\Theta(\log(n))$ | $\Theta(n)$       |

### 应用

#### Dijkstra & Prim

![SPFA](D:\Projects\OITraining\Templates\pb_ds\SPFA.jpg)

OI 中堆最常见的应用之一就是图论里最短路和最小生成树。

我们知道，对于非负权图的最短路，有两大经典算法：Dijkstra 和~~已死的~~SPFA。其中，`std::priority_queue`优化的 Dijkstra 复杂度是$\Theta(m\log m)$的，而手搓堆和线段树则是$\Theta(m\log n)$，对于 Fibonacci 堆则是吓人的$\Theta(m+n\log n)$。对于很稠密的图来说，我们有 $m=O(n^2)$，此时 dijkstra 的常数会涨得飞快，而在考场上手搓 Fib 堆又并不现实，因此我们需要一种简单的方式对 dijkstra 进一步提速。

同样，面对最小生成树，我们也有一样的问题：Kruskal 在稠密图上的表现不尽人意，而单独`std::priority_queue`优化的 Prim 算法复杂度和 Kruskal 相同，常数还更大，因此面对稠密图我们需要更快的最小生成树算法，典型例子就是[Moo Network G](https://www.luogu.com.cn/problem/P8191)。

> **WARINING: 事实上，如果 pbds 堆优化 dij 是标算的一部分，那么它不应该给出`std::priority_queue`不能通过的测试点。因此，pbds 堆优化更多可以看作是一种从标算不是最短路但是可以用最短路解的问题中骗分的方式。**

> GCC 认为`thin_heap_tag`在 Dijkstra 上表现会优于`pairing_heap_tag`，并且单从复杂度分析它也确实更优（等同于 Fib 堆），但是由于常数比较大，实测下来它的性能并没有配对堆优秀。

那么为什么 pbds 堆会比 STL 堆跑的快呢？究其原因有两点：

1. pbds 堆支持`modify`操作

2. pbds 堆`modify`操作时，如果堆中元素是减少的，那么复杂度是$o(\log n)$的（注意这里不是上确界）。事实上配对堆的`modify`复杂度在学术界还无法给出精确解~~（就连 Tarjan 老爷子都没算出来）~~。目前认为的下界是$\Omega(\log\log n)$，上界是$O(2^{\sqrt{\log\log n}})$。

事实上，Fib 堆跑 dijkstra 和 prim 快的原因也在于此：Fib 堆中元素减小时修改的复杂度是$\Theta(1)$的，这也是为什么`thin_heap_tag`有更优的理论复杂度。而 dijkstra 和 prim 中，每次松弛后，dis 显然是减小的。

> 有一个很奇怪的事情，pbds 的 dij 在 luogu P4779 里比 stl 堆快得多，但是我本机 benchmark 测无论是随机图还是网格套菊花两者速度都不相上下，开 O2 后 STL 堆甚至远远快过了手打堆和线段树。个人猜测是由于配对堆常数较大以及 pbds 堆底层数据结构内存不连续导致的，但是这并不能解释为什么 STL 堆能比手打堆和线段树快。

代码:

```cpp
#include <bits/stdc++.h>
#include <bits/extc++.h>
using namespace std;
#define ll long long
#define pll pair<ll,ll>
constexpr const ll MAXN = 100010;
constexpr const ll INF = 1ll << (sizeof(ll) * 8 - 2);
static ll n, m, s;
static ll dis[MAXN];
vector<pll> graph[MAXN];
__gnu_pbds::priority_queue<pll, greater<pll>> q;
decltype(q)::point_iterator its[MAXN];
void dijkstra() {
  ll u, v, d;
  for (int i = 1; i <= n; i++) {
    dis[i] = INF;
  }
  dis[s] = 0;
  for (int i = 1; i <= n; i++) {
    its[i] = q.push(make_pair(dis[i], i));
  }
  while (!q.empty()) {
    u = q.top().second;
    q.pop();
    for (pll obj : graph[u]) {
      v = obj.first;
      d = obj.second;
      if (dis[v] > dis[u] + d) {
        dis[v] = dis[u] + d;
        q.modify(its[v], make_pair(dis[u] + d, v));
      }
    }
  }
}
int main() {
  ll u, v, w;
  ios::sync_with_stdio(false);
  cin.tie(nullptr);
  cout.tie(nullptr);
  cin >> n >> m >> s;
  for (int i = 1; i <= m; i++) {
    cin >> u >> v >> w;
    graph[u].push_back(make_pair(v, w));
  }
  dijkstra();
  for (int i = 1; i <= n; i++) {
    cout << dis[i] << ' ';
  }
  cout << endl;
  return 0;
}
```

#### 可并堆

相比起优化 dijkstra 和 prim，pbds 堆在这一方面应用更广——毕竟 STL 中并没有数据结构能直接实现可并堆，而配对堆和左偏树都不是很好写。

**注意：`a.join(b)`后，`b`这个堆中的元素将会被清空。**

[【模板】左偏树（可并堆）](https://www.luogu.com.cn/problem/P3377)

代码：

```cpp
#include <bits/stdc++.h>
#include <bits/extc++.h>
using namespace std;
#define ll long long
#define pll pair<ll,ll>
const ll MAXN = 100010;
ll n, m;
ll fa[MAXN];
bool deleted[MAXN];
__gnu_pbds::priority_queue<pll, greater<pll>> heaps[MAXN];
ll find(ll x) {
  return fa[x] == x ? x : fa[x] = find(fa[x]);
}
int main() {
  ll opt, x, y, fax, fay;
  pll obj;
  cin >> n >> m;
  for (int i = 1; i <= n; i++) {
    fa[i] = i;
    deleted[i] = false;
    cin >> x;
    heaps[i].push(make_pair(x, i));
  }
  while (m--) {
    cin >> opt;
    if (opt == 1) {
      cin >> x >> y;
      fax = find(x);
      fay = find(y);
      if (fax == fay || deleted[x] || deleted[y]) {
        continue;
      }
      if (heaps[fax].size() >= heaps[fay].size()) {
        fa[fay] = fax;
        heaps[fax].join(heaps[fay]);
      } else {
        fa[fax] = fay;
        heaps[fay].join(heaps[fax]);
      }
    } else {
      cin >> x;
      if (deleted[x]) {
        cout << "-1" << endl;
        continue;
      }
      fax = find(x);
      obj = heaps[fax].top();
      cout << obj.first << endl;
      deleted[obj.second] = true;
      heaps[fax].pop();
    }
  }
  return 0;
}
```

可并堆可以解决很多类型的问题，比如某些树上问题。这些题目一般是一个节点维护一个堆，然后儿子的堆不断向父亲合并，过程中进行计算。这些题虽然常常有省选及以上的难度，但主要原因在于可并堆并不好实现。另外，可并堆通常会与并查集一块出现（比如上面的模板题）

## rope

~~（我也不知道他算什么数据结构）~~

但是它可以实现可持久化数组。它的底层结构是树状的。常数很大但复杂度并不高。

注意：严格来讲`rope`并不属于pbds。事实上，`rope`本身跟pbds是同级的。它的头文件是`rope`，命名空间则是`__gnu_cxx`而不是pbds。

`__gnu_cxx::rope`支持以下操作：

- `at(x)` & `operator[]` 同`vector`，复杂度$\Theta(1)$
- `push_back` & `append` 同`vector`，复杂度$\Theta(\log n)$
- `insert(x, other)` 在`x`的位置插入另一个串 复杂度最好$\Theta(\log n)$最坏$\Theta(n)$
- `erase(x, len)` 删除`x`开始的`len`个元素。复杂度$\Theta(\log n)$
- `replace(x, len, other)` 从第`x`位开始的`len`个元素替换为`other`。也可以省略`len`。复杂度$\Theta(\log n)`
- `substr(x, len)` 从`x`开始`len`位，复杂度$\Theta(\log n)$
- `operator+` 连接两个`rope`

我们来讲一讲`rope`一种常见的应用：可持久化数组。

[【模板】可持久化线段树 1（可持久化数组）](https://www.luogu.com.cn/problem/P3919)

> **WARNING: 由于`rope`的常数较大，而这个题目的数据比较凶悍，因此不保证`rope`能AC**

首先我们定义一个历史记录数组，里面存放了所有版本的`rope`的指针。每次创建一个新版本的时候，就从历史记录中取出那个版本创建一个新的`rope`。这看上去复杂度很高，但是由于`rope`底层的实现，实际上这个操作只是换了一个新的根节点，复杂度是$\Theta(1)$的。

接下来的事情就很简单了：读入，判断类型，修改或者输出即可。注意一下下标从0开始即可

代码：

```cpp
#include <bits/stdc++.h>
#include <ext/rope>
using namespace std;
#define ll long long
const ll MAXN = 1000010;
static ll n, m, x;
static ll v, k, s, t;
__gnu_cxx::rope<ll> *versions[MAXN];
int main() {
  ios::sync_with_stdio(false);
  cin.tie(nullptr);
  cout.tie(nullptr);
  cin >> n >> m;
  versions[0] = new __gnu_cxx::rope<ll>();
  for (int i = 1; i <= n; i++) {
    cin >> x;
    versions[0]->append(x);
  }
  for (int i = 1; i <= m; i++) {
    cin >> v >> k;
    versions[i] = new __gnu_cxx::rope<ll>(*versions[v]);
    if (k == 1) {
      cin >> s >> t;
      versions[i]->replace(s - 1, t);
    } else {
      cin >> x;
      cout << versions[i]->at(x - 1) << endl;
    }
  }
  return 0;
}
```

## 例题

### 平衡树

 - [[TJOI2007] 书架](https://www.luogu.com.cn/problem/P3850)
 - [[TJOI2019] 甲苯先生的滚榜](https://www.luogu.com.cn/problem/P5338)

### 可并堆

- [P2713 罗马游戏](https://www.luogu.com.cn/problem/P2713)
- [P1456 Monkey King](https://www.luogu.com.cn/problem/P1456)
- [P4331 [BOI2004]Sequence](https://www.luogu.com.cn/problem/P4331)

### rope

- [[NOI2003] 文本编辑器](https://www.luogu.com.cn/problem/P4008)

## END

这里有我自己测的hash和priority_queue的性能数据 https://github.com/AI1379/pbds_benchmark

参考：

https://www.luogu.com.cn/blog/Chanis/gnu-pbds

https://github.com/OI-Wiki/libs/blob/master/lang/pb-ds/C%2B%2B%E7%9A%84pb_ds%E5%BA%93%E5%9C%A8OI%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8.pdf

https://www.luogu.com.cn/training/9391
