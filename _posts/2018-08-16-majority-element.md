---
layout: post
title: 求众数
date: 2018-8-16 09:54:15
catalog: true
tags:
    - 算法
---

[题目链接](https://leetcode-cn.com/problems/majority-element/description/)

## 题目

给定一个大小为 n 的数组，找到其中的众数。众数是指在数组中出现次数大于` ⌊ n/2 ⌋ `的元素。

你可以假设数组是非空的，并且给定的数组总是存在众数。


**示例1:**

```
输入: [3,2,3]
输出: 3
```

**示例2:**

```
输入: [2,2,1,1,1,2,2]
输出: 2
```

## 答案分析

借助Map记录数值出现的次数，使用了额外的空间。

**我的答案：**

```java
class Solution {
    public int majorityElement(int[] nums) {
        Map<Integer, Integer> map = new HashMap<>();
        for (int n : nums) {
            map.put(n, map.getOrDefault(n, 0) + 1);
            if (map.get(n) > nums.length/2)
                return n;
        }
        return 0;
    }
}
```

**别人的答案：**

不需要使用额外空间。

```java
lass Solution {
  public int majorityElement(int[] nums) {
    int result = nums[0], count = 0;
    for (int num : nums) {
      if (count == 0) {
        result = num;
        count++;
      } else {
        if (result == num) {
          count++;
        } else {
          count--;
        }
      }
    }
    return result;
  }
}
```