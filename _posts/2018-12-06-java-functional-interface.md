---
layout: post
title: 函数式接口
date: 2018-12-6 14:27:07
catalog: true
tags:
    - Java
---

## 介绍

函数式接口(Functional Interface)就是一个有且仅有一个抽象方法，但是可以有多个非抽象方法的接口。

函数式接口可以被隐式转换为 lambda 表达式。

## 引入

如定义了一个函数式接口如下：

```java
public interface GreetingService {
    void sayMessage(String message);
}
```

那么就可以使用Lambda表达式来表示该接口的一个实现(注：JAVA 8 之前一般是用匿名类实现的)：

```java
GreetingService greetingService = message -> System.out.println("message = " + message);
GreetingService greetingService1 = message -> {
            System.out.println("message length = " + message.length());
        };
greetingService.sayMessage("Hello");
greetingService1.sayMessage("World");
```

## JDK 1.8 新增加的函数接口

java.util.function 它包含了很多类，用来支持 Java的 函数式编程，该包中的函数式接口有：

|序号|接口|描述|
|:--|:--:|:--|
|1|`Consumer<T>`|代表了接受一个输入参数并且无返回的操作。|
|2|`Function<T,R>`|接受一个输入参数，返回一个结果。|
|3|`Predicate<T>`|接受一个输入参数，返回一个布尔值结果。|
|4|`Supplier<T>`|无参数，返回一个结果。|

## 参考

[Java 8 函数式接口](http://www.runoob.com/java/java8-functional-interfaces.html)