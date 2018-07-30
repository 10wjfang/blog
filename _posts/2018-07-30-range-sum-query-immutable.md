---
layout: post
title: 区域和检索 - 数组不可变
date: 2018-7-30 15:54:09
catalog: true
tags:
    - 算法
    - 动态规划
---

[题目链接](https://leetcode-cn.com/problems/range-sum-query-immutable/description/)

## 题目

给定一个整数数组  nums，求出数组从索引 i 到 j  (i ≤ j) 范围内元素的总和，包含 i,  j 两点。

**示例:**

```
给定 nums = [-2, 0, 3, -5, 2, -1]，求和函数为 sumRange()

sumRange(0, 2) -> 1
sumRange(2, 5) -> -1
sumRange(0, 5) -> -3
```

**说明:**

你可以假设数组不可变。

会多次调用 sumRange 方法。

## 答案

采用动态规划思想，先计算0到每个位置的和，再去相减。16 / 16 个通过测试用例，执行用时：266 ms

**我的答案：**

```java
class NumArray {

    private int[] nums;
    private Map<Integer, Integer> sumList = new HashMap<>();
    
    public NumArray(int[] nums) {
        this.nums = nums;
        int sum = 0;
        for (int i = 0; i < nums.length; i++) {
            sum += nums[i];
            sumList.put(i, sum);
        }
    }
    
    // [[[-2,0,3,-5,2,-1]],[0,2],[2,5],[0,5]]
    // [0,2] -> [0,1] + [2]
    public int sumRange(int i, int j) {
        int start = 0;
        int end = 0;
        start = sumList.get(i);
        end = sumList.get(j);
        //System.out.println(start + ", " + end);
        return end - start + nums[i];
    }
}

/**
 * Your NumArray object will be instantiated and called as such:
 * NumArray obj = new NumArray(nums);
 * int param_1 = obj.sumRange(i,j);
 */
```

**别人的答案：**

用时：126ms

```java
class NumArray {

   int nums[];
	public NumArray(int[] nums) {
        this.nums = nums;
		for (int i = 1; i < nums.length; i++) {
			nums[i] += nums[i-1];
		}
		
    }
    
    public int sumRange(int i, int j) {
    	if(i==0)
    		return nums[j];
    	else
    		return nums[j]-nums[i-1];
    }
}

/**
 * Your NumArray object will be instantiated and called as such:
 * NumArray obj = new NumArray(nums);
 * int param_1 = obj.sumRange(i,j);
 */
```