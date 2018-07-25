---
layout: post
title: 整数拆分
date: 2018-7-25 17:47:19
catalog: true
tags:
    - 算法
    - 动态规划
---

# 343. 整数拆分

[题目链接](https://leetcode-cn.com/problems/integer-break/description/)

给定一个正整数 n，将其拆分为至少两个正整数的和，并使这些整数的乘积最大化。 返回你可以获得的最大乘积。

例如，给定 n = 2，返回1（2 = 1 + 1）；给定 n = 10，返回36（10 = 3 + 3 + 4）。

注意：你可以假设 n 不小于2且不大于58。

**我的答案：**

使用动态规划思想解决，50 / 50 个通过测试用例，执行用时：0 ms。

```java
class Solution {
    public int integerBreak(int n) {
        if (n == 2) {
            return 1;
        }
        if (n == 3) {
            return 2;
        }
        int res = 2;
        int a_3 = 1;
        int a_2 = 2;
        int a_1 = 3;
        for (int i = 4; i <= n; i++) {
            if (a_2 *2 > a_3 * 3)
                res = a_2 * 2;
            else
                res = a_3 * 3;
            a_3 = a_2;
            a_2 = a_1;
            a_1 = res;
        }
        return res;
    }
}
```

**别人的答案**

```java
class Solution {
    public int integerBreak(int n) {
        if(n<=3)
        	return n-1;
        else{
        	if(n%3==1){
        		int num = n/3 - 1;
        		return (int)Math.pow(3,num)*4;
        	}else if(n%3==2){
        		int num = n/3;
        		return (int)Math.pow(3,num)*2;
        	}else{
        		int num = n/3;
        		return (int)Math.pow(3,num);
        	}      	
        }
    }
}
```