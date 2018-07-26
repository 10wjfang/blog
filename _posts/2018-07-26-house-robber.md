---
layout: post
title: 打家劫舍
date: 2018-7-26 17:41:18
catalog: true
tags:
    - 算法
    - 动态规划
---

[题目链接](https://leetcode-cn.com/problems/house-robber/description/)

## 题目

你是一个专业的小偷，计划偷窃沿街的房屋。每间房内都藏有一定的现金，影响你偷窃的唯一制约因素就是相邻的房屋装有相互连通的防盗系统，**如果两间相邻的房屋在同一晚上被小偷闯入，系统会自动报警**。

给定一个代表每个房屋存放金额的非负整数数组，计算你**在不触动警报装置的情况下，**能够偷窃到的最高金额。

**示例 1:**

```
**输入:** [1,2,3,1]
**输出:** 4
**解释:** 偷窃 1 号房屋 (金额 = 1) ，然后偷窃 3 号房屋 (金额 = 3)。
     偷窃到的最高金额 = 1 + 3 = 4 。
```

**示例 2:**

```
**输入:** [2,7,9,3,1]
**输出:** 12
**解释:** 偷窃 1 号房屋 (金额 = 2), 偷窃 3 号房屋 (金额 = 9)，接着偷窃 5 号房屋 (金额 = 1)。
     偷窃到的最高金额 = 2 + 9 + 1 = 12 。
```

## 答案

采用动态规划思想，当前解为上上个数最优解和当前数相加，再和最大值比较。69 / 69 个通过测试用例，执行用时：17 ms

**我的答案：**

```java
class Solution {
    // [4,2,3,8]
    public int rob(int[] nums) {
        if (nums.length == 0)
            return 0;
        if (nums.length == 1)
            return nums[0];
        if (nums.length == 2)
            return Math.max(nums[0], nums[1]);
        int max_1 = Math.max(nums[0], nums[1]);
        int max_2 = nums[0];
        int max = max_1;
        int res = max;
        for (int i = 2; i < nums.length; i++) {
            max = max_2 + nums[i];
            max_2 = max_1;
            if (res < max)
                res = max;
            max_1 = res;
        }
        return res;
    }
}
```

**别人的答案：**

```java
class Solution {
    public int rob(int[] nums) {
        if(nums==null||nums.length<1){
            return 0;
        }
        int n=nums.length;
        if(n==1){
            return nums[0];
        }
        if(n==2){
            return Math.max(nums[1],nums[0]);
        }
        int f_n=0;
        int f_n_1=Math.max(nums[1],nums[0]);
        int f_n_2=nums[0];
        int i=2;
        while(i<n){
            f_n=Math.max(f_n_1,f_n_2+nums[i]);
            f_n_2=f_n_1;
            f_n_1=f_n;
            i++;
        }
        return f_n;
    }
}
```