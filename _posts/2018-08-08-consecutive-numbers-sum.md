---
layout: post
title: 连续整数求和
date: 2018-8-8 20:22:55
catalog: true
tags:
    - 算法
---

[题目链接](https://leetcode-cn.com/contest/weekly-contest-83/problems/consecutive-numbers-sum/)

## 题目

给定一个正整数 `N`，试求有多少组连续正整数满足所有数字之和为 `N`?


**示例1:**

```
输入: 5
输出: 2
解释: 5 = 5 = 2 + 3，共有两组连续整数([5],[2,3])求和后为 5。
```

**示例2:**

```
输入: 9
输出: 3
解释: 9 = 9 = 4 + 5 = 2 + 3 + 4
```

**示例3:**

```
输入: 15
输出: 4
解释: 15 = 15 = 8 + 7 = 4 + 5 + 6 = 1 + 2 + 3 + 4 + 5
```


**说明：** `1 <= N <= 10 ^ 9`


## 答案

奇数个时，平均值为整数；偶数个正整数时，平均值小数位0.5

**我的答案：**

```java
class Solution {
    // 15 / 1 = 15
    // 15 / 2 = 7.5 = 8 + 7
    // 15 / 3 = 5 = 4 + 5 + 6
    // 15 / 4 = 3.75
    // 15 / 5 = 3 = 1 + 2 + 3 + 4 + 5
    // 15 / 6 = 2.5
    // 14 = 2+3+4+5 = 14/4
    // 7 8 9 10 34
    // 43156417 4
    public int consecutiveNumbersSum(int N) {
        int res = 1;
        int i = 2;
        double div = 1.0* N / i;
        while (2*div > i) {
            if (i % 2 == 0 && div - (int)div == 0.5) {
                //System.out.println(2*div);
                res++;
            }
            if (i % 2 != 0 && div == (int)div) {
                //System.out.println(2*div);
                res++;
            }
            i++;
            div = 1.0* N / i;
        }
        return res;
    }
}
```