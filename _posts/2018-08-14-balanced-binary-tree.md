---
layout: post
title: 平衡二叉树
date: 2018-8-14 19:43:29
catalog: true
tags:
    - 算法
---

[题目链接](https://leetcode-cn.com/problems/balanced-binary-tree/description/)

## 题目

给定一个二叉树，判断它是否是高度平衡的二叉树。

本题中，一棵高度平衡二叉树定义为：

> 一个二叉树每个节点 的左右两个子树的高度差的绝对值不超过1。


**示例1:**

给定二叉树 `[3,9,20,null,null,15,7]`

```
    3
   / \
  9  20
    /  \
   15   7
```
返回 `true` 。


**示例2:**

给定二叉树 `[1,2,2,3,3,null,null,4,4]`

```
       1
      / \
     2   2
    / \
   3   3
  / \
 4   4
```
返回 `false` 。


## 答案分析

后序遍历二叉树，计算节点高度。

**我的答案：**

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    boolean isBalance = true;
    
    public boolean isBalanced(TreeNode root) {
        TreeDepth(root);
        return isBalance;
    }
    
    public int TreeDepth(TreeNode root) {
        if (root == null) {
            return 0;
        }
        int left = TreeDepth(root.left);
        int right = TreeDepth(root.right);
        if (Math.abs(left - right) > 1) {
            isBalance = false;
        }
        return Math.max(left, right) + 1;
    }
}
```