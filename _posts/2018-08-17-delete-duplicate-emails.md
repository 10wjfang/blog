---
layout: post
title: 删除重复的电子邮箱
date: 2018-8-17 16:46:20
catalog: true
tags:
    - 数据库
---

[题目链接](https://leetcode-cn.com/problems/delete-duplicate-emails/description/)

## 题目

编写一个 SQL 查询，来删除 Person 表中所有重复的电子邮箱，重复的邮箱里只保留 Id 最小 的那个。

```
+----+------------------+
| Id | Email            |
+----+------------------+
| 1  | john@example.com |
| 2  | bob@example.com  |
| 3  | john@example.com |
+----+------------------+
Id 是这个表的主键。
```

例如，在运行你的查询语句之后，上面的 Person 表应返回以下几行:

```
+----+------------------+
| Id | Email            |
+----+------------------+
| 1  | john@example.com |
| 2  | bob@example.com  |
+----+------------------+
```

## 答案分析

注意没有加`t`会出现报错信息：
You can't specify target table 'tempA' for update in FROM clause

需要查询的时候增加一层中间表，就可以避免该错误。

**我的答案：**

```sql
# Write your MySQL query statement below
delete from Person 
where Id not in 
(
    select Id from 
    (
        select min(Id) as Id from Person group by Email having count(*) > 1
    ) t
)
and 
Email in 
(
    select Email from 
    (
        select Email from Person group by Email having count(*) > 1
    ) t
)
```

**别人的答案：**

```sql
delete from Person
where id not in 
(select m.mi from
    (select min(id) mi from Person group by email) as m
) 
```