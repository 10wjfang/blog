---
layout: post
title: 较大分组的位置
date: 2018-8-8 20:19:14
catalog: true
tags:
    - 算法
---

[题目链接](https://leetcode-cn.com/contest/weekly-contest-83/problems/positions-of-large-groups/)

## 题目

在一个由小写字母构成的字符串 S 中，包含由一些连续的相同字符所构成的分组。

例如，在字符串 S = "abbxxxxzyy" 中，就含有 "a", "bb", "xxxx", "z" 和 "yy" 这样的一些分组。

我们称所有包含大于或等于三个连续字符的分组为较大分组。找到每一个较大分组的起始和终止位置。

最终结果按照字典顺序输出。


**示例1:**

```
输入: "abbxxxxzzy"
输出: [[3,6]]
解释: "xxxx" 是一个起始于 3 且终止于 6 的较大分组。
```

**示例2:**

```
输入: "abc"
输出: []
解释: "a","b" 和 "c" 均不是符合要求的较大分组。
```

**示例3:**

```
输入: "abcdddeeeeaabbbcd"
输出: [[3,5],[6,9],[12,14]]
```


**说明：** `1 <= S.length <= 1000`


## 答案

注意最后连续超过2个的情况：abbb。

**我的答案：**

```java
class Solution {
    public List<List<Integer>> largeGroupPositions(String S) {
        char[] chars = S.toCharArray();
        List<List<Integer>> res = new ArrayList<>();
        int count = 1;
        char cur = chars[0];
        int start = 0;
        for (int i=1; i<chars.length; i++) {
            if (cur == chars[i]) {
                count++;
            }
            else {
                if (count >= 3) {
                    List<Integer> item = new ArrayList<>();
                    item.add(start);
                    item.add(start+count-1);
                    res.add(item);
                }
                start = i;
                cur = chars[i];
                count = 1;
            }
        }
        if (count >= 3) {
            List<Integer> item = new ArrayList<>();
            item.add(start);
            item.add(start+count-1);
            res.add(item);
        }
        return res;
    }
}
```