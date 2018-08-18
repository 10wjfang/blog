---
layout: post
title: MySQL中CASE WHEN、IF用法
date: 2018-8-18 11:30:46
catalog: true
tags:
    - 数据库
    - MySQL
---

# CASE WHEN用法

    CASE  <单值表达式>

        WHEN <表达式值> THEN <SQL语句或者返回值>

        WHEN <表达式值> THEN <SQL语句或者返回值>

        ...

        WHEN <表达式值> THEN <SQL语句或者返回值>

        ELSE <SQL语句或者返回值>

    END

例子：

```sql
# 交换所有的 f 和 m 值
UPDATE salary SET sex  = (CASE WHEN sex = 'm' THEN 'f' ELSE 'm' END)
```

## IF用法

`IF(expr1,expr2,expr3)`，如果expr1的值为true，则返回expr2的值，如果expr1的值为false，则返回expr3的值。

例子：

```sql
# 交换所有的 f 和 m 值
update salary set sex = if (sex = 'm', 'f', 'm');

select name,if(sex=0,'女','男') as sex from student
```