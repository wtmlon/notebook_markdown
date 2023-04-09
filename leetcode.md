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
