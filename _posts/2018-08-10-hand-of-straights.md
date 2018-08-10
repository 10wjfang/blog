---
layout: post
title: 一手顺子
date: 2018-8-10 14:10:43
catalog: true
tags:
    - 算法
---

[题目链接](https://leetcode-cn.com/contest/weekly-contest-87/problems/hand-of-straights/)

## 题目

爱丽丝有一手（`hand`）由整数数组给定的牌。 

现在她想把牌重新排列成组，使得每个组的大小都是 `W`，且由 `W` 张连续的牌组成。

如果她可以完成分组就返回 `true`，否则返回 `false`。


**示例1:**

```
输入：hand = [1,2,3,6,2,3,4,7,8], W = 3
输出：true
解释：爱丽丝的手牌可以被重新排列为 [1,2,3]，[2,3,4]，[6,7,8]。
```

**示例2:**

```
输入：hand = [1,2,3,4,5], W = 4
输出：false
解释：爱丽丝的手牌无法被重新排列成几个大小为 4 的组。
```


**提示：**

1. `1 <= hand.length <= 10000`
2. `0 <= hand[i] <= 10^9`
3. `1 <= W <= hand.length`


## 答案分析

利用有序Map，存储每个数出现的次数，按顺序获取第一个数。

**别人的答案：**

```java
class Solution {
    public boolean isNStraightHand(int[] hand, int W) {
		if(hand.length % W != 0) return false;
		TreeMap<Integer, Integer> dp = new TreeMap<Integer, Integer>();
		for(int out: hand) {
			dp.put(out, 1 + dp.getOrDefault(out, 0));
		}
		while(!dp.isEmpty()) {
			int first = dp.firstKey();
			for(int a = 0; a < W; a++) {
				if(dp.getOrDefault(first+a, 0) == 0) return false;
				update(dp, first+a, -1);
			}
		}
		return true;
    }

	private void update(Map<Integer, Integer> m, int k, int v) {
		m.put(k, v + m.getOrDefault(k, 0));
		if(m.get(k) == 0) m.remove(k);
	}
}
```