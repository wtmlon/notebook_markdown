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


