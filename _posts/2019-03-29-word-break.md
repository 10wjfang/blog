---
layout: post
title: 单词拆分
date: 2019-3-29 10:05:30
catalog: true
tags:
    - 算法
---

[题目链接](https://leetcode-cn.com/problems/word-break/)

## 题目

给定一个非空字符串 s 和一个包含非空单词列表的字典 wordDict，判定 s 是否可以被空格拆分为一个或多个在字典中出现的单词。

**说明：**

- 拆分时可以重复使用字典中的单词。
- 你可以假设字典中没有重复的单词。


**示例1:**

```
输入: s = "leetcode", wordDict = ["leet", "code"]
输出: true
解释: 返回 true 因为 "leetcode" 可以被拆分成 "leet code"。
```

**示例2:**

```
输入: s = "applepenapple", wordDict = ["apple", "pen"]
输出: true
解释: 返回 true 因为 "applepenapple" 可以被拆分成 "apple pen apple"。
     注意你可以重复使用字典中的单词。
```

**示例3:**

```
输入: s = "catsandog", wordDict = ["cats", "dog", "sand", "and", "cat"]
输出: false
```

## 答案分析

和背包问题类似，使用动态规划的算法解决，dp[i]表示字符串s的前i个字符能否被拆分成字典，`dp[i] = dp[j] && s(j,i) in wordDict`。

**我的答案：**

```java
class Solution {
    public boolean wordBreak(String s, List<String> wordDict) {
        boolean[] dp = new boolean[s.length()+1];
        dp[0] = true;
        for (int i=0; i<=s.length(); i++) {
            for (int j=0; j<i; j++) {
                if (dp[j] && wordDict.indexOf(s.substring(j, i)) > -1) {
                    dp[i] = true;
                    break;
                }
            }
        }
        return dp[s.length()];
    }
}
```

**优化：**

- j到i的长度不超过wordDict字典的最大长度：`j < i && i - j < max_length`