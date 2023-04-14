# 动态规划

## 零钱兑换

多个值可无限取，求和为target的最小值

$$
dp[n] = min(dp[n], dp[n-val_i])
$$

## 最长递增子序列

1)同俄罗斯信封套娃问题，dp[n] 以n结尾最长递增子序列，

$$
dp[n]=max(dp[n], dp[0..n] + 1)
$$

2)二分解法，类似蜘蛛纸牌

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-04-14-49-21-image.png)

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-04-14-50-24-image.png)

拓展到二维俄罗斯套娃信封问题，以长升序，宽降序排列，对宽进行最长子序列计算即可（长已经递增）

## 最长子数组和

因为输入有负数所以不能slide win（因为无法判断窗口扩张和收缩）~~其实是可以的~~

```cpp
dp[n] = max(num[n], dp[n-1] + num[n]);
//因为dp[n]只和dp[n-1]有关
dp[n] = max(num[n], tmp + num[n]);
tmp = dp[n];
```

## 最长回文子序列

dp[i][j]为i到j子序列最长值

```cpp
//s[i] == s[j]
dp[i][j] = dp[i+1][j-1] + 2;
//else
dp[i][j] = max(dp[i+1][j], dp[i][j-1]);
```

## 使原串变为回文所需插入最少字符个数<mark>（难题）</mark>

dp[i][j]为i到j变成回文最少插入次数, dp[i][i] = 0，<mark>单个字符本身就回文<mark>

```cpp
//s[i] == s[j]
dp[i][j] = dp[i+1][j-1];
//else
dp[i][j] = min(dp[i+1][j], dp[i][j-1]) + 1;
//为何不用dp[i+1][j-1],因为s[i] != s[j]时，dp[i+1][j-1] d[i+1][j] dp[i][j-1](存疑)
```

## 编辑距离（把一个串变成另一个串增删改的最少次数）<mark>（难题） </mark>

dp[i][j]为s[0..i]变成s2[0..j]最少次数

```cpp
//s[i] == s2[j]
dp[i][j] = dp[i+1][j-1];
//else
dp[i][j] = min(dp[i+1][j], dp[i][j-1], dp[i+1][j-1]) + 1;
//为何会多一个dp[i+1][j-1]? 因为不能保证dp[i+1][j-1]都大于dp[i+1][j]和dp[i][j-1]（存疑）
```

## 田忌赛马问题（追求获胜次数最多）

两个队伍降序排列，打得过则打，打不过用最菜的去顶包。

## 区域修改问题（修改区域i-j）

使用查分数组

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-06-16-43-26-image.png)

## 雷区行走（求点到另一点所有路径离地雷的最近距离的最大值）

用二分搜索值cnt，离雷距离小于cnt的点视为不可见，如果在这样的情况下存在通路，则往cnt大的区间二分，否则往小于cnt的区间二分。

## 多彩的树（每个节点是红黄蓝三个色其中之一，求有多少条边可以移除，移除后的两个子树红黄蓝三个颜色的节点都有）

临接表建图，从某个点开始遍历，运用单向dfs遍历法。计算这个节点以后的子树所包含的三色节点个数，同时统计满不满足条件。

```python
#从node开始遍历，不走回头路（既pre点不能走）
def dfs(node, pre):
    cur = [0, 0, 0]
    if colors[node] == 'R': cur[0] += 1
    if colors[node] == 'G': cur[1] += 1
    if colors[node] == 'B': cur[2] += 1
    for nx in nxs[node]:
        if nx != pre:
            tmp = dfs(nx, node)
            for i in range(3): cur[i] += tmp[i]

    if cur[0] > 0 and cur[1] > 0 and cur[2] > 0 and all[0] - cur[0] > 0 and all[1] - cur[1] > 0 and all[2] - cur[2] > 0:
        global res
        res += 1
    return cur
```

## 水果成篮（把n个水果分成m份，每份根据水果大小和个数计算成本，求最优分配法）

使用动态规划，dp[i][j]为前i个水果分成j份的最低成本，则：

```cpp
dp[i][1] = max(fruit[0-i]);
dp[i][j] = INT_MAX // i < j
dp[i][j] = min(dp[k][j-1]+fun(fruit[k+1..i])) // i>=j, k = 1...i, fun()为成本函数
```

# K好数（对编程要求高）

找出一个区间[l, r]内的k好数，k好数的定义是：这个数可由k的幂次相加而成，比如：

$$
17 = 4^2 + 4^0
$$

所以17是4好数，分析规规律可知所有的k好数分布如下：

$$
第6（110）个k好数为 = k^2 + k^1 + k^0
$$

所以要找到[l, r]内第一个k好数和最后一个k好数，分别算出他们的k好数序号，做差。

由于序号是递增的，所以要用二分找到第一个大于l的序号left

$$
K(left) = k^n + k....k^0
$$

大致就这样。

# 线段树的概念（共支持Build, update, getsum)

线段树可以在logN的时间复杂度内实现单点修改、区间修改、区间查询（区间求和，求区间最大值，求区间最小值）等操作。

线段树维护的信息，需要满足可加性，即能以可以接受的速度合并信息和修改信息，包括在使用懒惰标记时，标记也要满足可加性（例如取模就不满足可加性，对 4取模然后对 3取模，两个操作就不能合并在一起做）。

假设有个数组 a = [10, 11, 12, 13, 14]，要将其转化为线段树，有以下做法：设线段树的根节点编号为 1，用数组 d来保存我们的线段树，  d_i 用来保存线段树上编号为i的节点的值（这里每个节点所维护的值就是这个节点所表示的区间总和），如图所示：

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-11-17-26-29-image.png)

类似堆排序建树,

$$
d_i 的左儿子为d_{2*i}, 右孩子为d_{2*i+1}， 其中，假设d_i为[s, t]闭区间内的和
$$

$$
那么d_{2*i}为[s, (s+t)/2]区间的和，d_{2*i+1}为[(s+t)/2 + 1, t]区间和。 
$$

## 建树代码

```c
void build(int s, int t, int p) {
  // 对 [s,t] 区间建立线段树,当前根的编号为 p
  if (s == t) {
    d[p] = a[s];
    return;
  }
  int m = (s + t) / 2;
  build(s, m, p * 2), build(m + 1, t, p * 2 + 1);//递归到左右子树
  // 递归对左右区间建树
  d[p] = d[p * 2] + d[(p * 2) + 1];
}
//in main
build(1, n, 1); // a内数据的排列应该为1..n而不是1..n-1
```

## 区间查找代码（无懒更新）

```c
// 查找p为根结点的树的[l, r]区间
int getsum(int l, int r, int s, int t, int p) {
  // [l,r] 为查询区间,[s,t] 为当前节点包含的区间,p 为当前节点的编号
  if (l <= s && t <= r)
    return d[p];  // 当前区间为询问区间的子集时直接返回当前区间的和
  int m = (s + t) / 2, sum = 0;
  if (l <= m) sum += getsum(l, r, s, m, p * 2);
  // 如果左儿子代表的区间 [l,m] 与询问区间有交集,则递归查询左儿子
  if (r > m) sum += getsum(l, r, m + 1, t, p * 2 + 1);
  // 如果右儿子代表的区间 [m+1,r] 与询问区间有交集,则递归查询右儿子
  return sum;
}
```

# 数值更新，优化：懒更新

为什么要懒更新，因为如果有区间更新的需求，一次全更新会使复杂度上升，所以懒更新机制只有在有区间更新才有必要使用，如果每次更新只更新只更新一个节点则无必要。

```c
void update(int l, int r, int c, int s, int t, int p) {
  // [l,r] 为修改区间,c 为被修改的元素的变化量,[s,t] 为当前节点包含的区间,p
  // 为当前节点的编号
  if (l <= s && t <= r) {
    d[p] += (t - s + 1) * c, b[p] += c;
    return;
  }  // 当前区间为修改区间的子集时直接修改当前节点的值,然后打标记,结束修改
  int m = (s + t) / 2;
  if (b[p] && s != t) {
    // 如果当前节点的懒标记非空,则更新当前节点两个子节点的值和懒标记值
    d[p * 2] += b[p] * (m - s + 1), d[p * 2 + 1] += b[p] * (t - m);
    b[p * 2] += b[p], b[p * 2 + 1] += b[p];  // 将标记下传给子节点
    b[p] = 0;                                // 清空当前节点的标记
  }
  if (l <= m) update(l, r, c, s, m, p * 2);
  if (r > m) update(l, r, c, m + 1, t, p * 2 + 1);
  d[p] = d[p * 2] + d[p * 2 + 1];
}
```

加入懒更新机制的getsum函数，除了获取值以外还要负责更新

```c
int getsum(int l, int r, int s, int t, int p) {
  // [l,r] 为查询区间,[s,t] 为当前节点包含的区间,p为当前节点的编号
  if (l <= s && t <= r) return d[p];
  // 当前区间为询问区间的子集时直接返回当前区间的和
  int m = (s + t) / 2;
  if (b[p]) {
    // 如果当前节点的懒标记非空,则更新当前节点两个子节点的值和懒标记值
    d[p * 2] += b[p] * (m - s + 1), d[p * 2 + 1] += b[p] * (t - m),
        b[p * 2] += b[p], b[p * 2 + 1] += b[p];  // 将标记下传给子节点
    b[p] = 0;                                    // 清空当前节点的标记
  }
  int sum = 0;
  if (l <= m) sum = getsum(l, r, s, m, p * 2);
  if (r > m) sum += getsum(l, r, m + 1, t, p * 2 + 1);
  return sum;
}
```

# 字符串切分（分为k个子串，使k个子串的含义（可以是代价，长度..看题目）最大值最小）

典型的二分问题，二分一个mid，从左到右开始分割子串，使每个子串的含义最长且不大于mid，看看能否分k分，可以则扩大mid，不可以则缩小mid。
