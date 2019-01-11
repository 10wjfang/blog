---
layout: post
title: commons-lang3工具类学习
date: 2019-1-11 10:15:23
catalog: true
tags:
    - Java
---

## 简介

apache提供的众多commons工具包，号称Java第二API，而common里面lang3包更是被我们使用得最多的。因此本文主要详细讲解lang3包里面几乎每个类的使用。提供日期、异常、事件、构造、多线程并发、数学、可变、元组、文本等工具。

## 使用

```xml
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-lang3</artifactId>
  <version>3.8.1</version>
</dependency>
```

## 常用API

#### ArrayUtils

用于对数组的操作，如添加、查找、删除、子数组、倒序、元素类型转换等。

#### StringUtils

- isBlank：检查字符串是否为空或null
- isNotBlank：检查字符串是否不为null、empty或空格字符，返回一个boolean
- abbreviate：返回一个指定长度加省略号的字符串，maxWidth必须大于3
- abbreviateMiddle：将字符串缩短到指定长度（length），字符串的中间部分用替换字符串（middle）显示
- capitalize：将字符串第一个字符大写并返回

#### DateFormatUtils

- format：将 java.util.Date 类型的日期数据转换成指定格式的 String 字符串

#### DateUtils

- parseDate：将 String 类型的日期数据转换成 Date 类型的日期数据

## 参考

[Apache Commons Lang 3.8 API](http://commons.apache.org/proper/commons-lang/javadocs/api-3.8/)