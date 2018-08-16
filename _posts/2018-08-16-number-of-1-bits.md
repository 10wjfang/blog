---
layout: post
title: 位1的个数
date: 2018-8-16 10:51:30
catalog: true
tags:
    - 算法
---

[题目链接](https://leetcode-cn.com/problems/number-of-1-bits/description/)

## 题目

编写一个函数，输入是一个无符号整数，返回其二进制表达式中数字位数为 ‘1’ 的个数（也被称为汉明重量）。

**示例1:**

```
输入: 11
输出: 3
解释: 整数 11 的二进制表示为 00000000000000000000000000001011
```

**示例2:**

```
输入: 128
输出: 1
解释: 整数 128 的二进制表示为 00000000000000000000000010000000
```

## 答案分析

关键将符号位计算进去。

**我的答案：**

```java
public class Solution {
    // you need to treat n as an unsigned value
    public int hammingWeight(int n) {
        int res = 0;
        if (n < 0) {
            n = n & 0x7FFFFFFF;
            System.out.println(n);
            res++;
        }
        while (n > 0) {
            if (n % 2 == 1)
                res++;
            n = n >> 1;
        }
        return res;
    }
}
```

**别人的答案：**

更简洁的方法：

```java
public class Solution {
    // you need to treat n as an unsigned value
    public int hammingWeight(int n) {
        int count = 0;
        while(n != 0) {
            n = n & (n-1);
            count++;
        }
        return count;
    }
}
```