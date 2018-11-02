---
layout: post
title: Java动态代理：Spring AOP的实现
date: 2018-11-2 16:10:41
catalog: true
tags:
    - Java
---

## 什么是动态代理

调用Proxy返回一个代理对象，使用这个代理对象执行你的方法时，它会先执行代理对象里的方法再执行你的业务方法。