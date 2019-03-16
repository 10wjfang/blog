---
layout: post
title: 括号生成
date: 2019-3-16 11:04:12
catalog: true
tags:
    - 算法
---

[题目链接](https://leetcode-cn.com/problems/generate-parentheses/)

## 题目

给出 n 代表生成括号的对数，请你写出一个函数，使其能够生成所有可能的并且有效的括号组合。

**示例1:**

例如，给出 n = 3，生成结果为：

```
[
  "((()))",
  "(()())",
  "(())()",
  "()(())",
  "()()()"
]
```

## 答案分析

利用递归的思想，将`()`插入到上一步的字符串中，使用`Set`去重。

**我的答案：**

```java
class Solution {
    Set<String> res = new HashSet<>();
    public List<String> generateParenthesis(int n) {
        generate("()", n);
        return new ArrayList<>(res);
    }
    
    public void generate(String last, int n) {
        if (n == 1) {
            res.add(last);
        }
        else {
            for (int i=0; i<last.length(); i++) {
                StringBuilder sb = new StringBuilder(last);
                sb.insert(i, "()");
                generate(sb.toString(), n-1);
            }
            //generate("(" + last + ")", n-1);
        }
    }
}
```

执行用时：	82 ms，用时较长，中间产生了很多重复的字符串，产生了重复的递归调用。

**别人的答案：**

更快的方法，左括号小于n则产生左括号，右括号小于左括号则产生右括号：

```java
class Solution {
    List<String> list = new ArrayList<>();

    public List<String> generateParenthesis(int n) {
        if (n < 1) {
            return list;
        }
        add(0, 0, n, "");
        return list;
    }

    public void add(int open, int close, int n, String str) {
        if (str.length() == n * 2) {
            list.add(str);
            return;
        }
        if (open < n) {
            add(open + 1, close, n, str + "(");
        }
        if (close < open) {
            add(open, close + 1, n, str + ")");
        }
    }
}
```

用时：2ms