---
layout: post
title: 三维形体投影面积
date: 2018-8-7 14:08:21
catalog: true
tags:
    - 算法
---

[题目链接](https://leetcode-cn.com/contest/weekly-contest-96/problems/projection-area-of-3d-shapes/)

## 题目

在 N * N 的网格中，我们放置了一些与 x，y，z 三轴对齐的 1 * 1 * 1 立方体。

每个值 v = grid[i][j] 表示 v 个正方体叠放在单元格 (i, j) 上。

现在，我们查看这些立方体在 xy、yz 和 zx 平面上的投影。

投影就像影子，将三维形体映射到一个二维平面上。

在这里，从顶部、前面和侧面看立方体时，我们会看到“影子”。

返回所有三个投影的总面积。


**示例1:**

```
输入：[[2]]
输出：5
```

**示例2:**

```
输入：[[1,2],[3,4]]
输出：17
解释：
这里有该形体在三个轴对齐平面上的三个投影(“阴影部分”)。
```
![img](../../../../img/in-post/post-3d-shapes/shadow.png)

**提示：**

- `1 <= grid.length = grid[0].length <= 50`
- `0 <= grid[i][j] <= 50`

## 答案

xy投影: >0个数，yz投影: 每行最大数相加，xz投影: 每列最大数相加。

**我的答案：**

```java
class Solution {
    // xy投影: >0个数
    // yz投影: 每行最大数相加
    // xz投影: 每列最大数相加
    public int projectionArea(int[][] grid) {
        int xy = 0;
        int yz = 0;
        int zx = 0;
        int[] maxRow = new int[grid.length];
        for (int i=0; i<grid.length; i++) {
            int maxCol = 0;
            for (int j=0; j<grid[i].length; j++) {
                maxRow[j] = Math.max(maxRow[j], grid[i][j]);
                if (grid[i][j] > 0) {
                    xy++;
                    maxCol = Math.max(maxCol, grid[i][j]);
                }
            }
            yz += maxCol;
        }
        int res = xy + yz;
        for (int j=0; j< grid.length; j++)
            res += maxRow[j];
        return res;
    }
}
```