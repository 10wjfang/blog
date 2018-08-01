---
layout: post
title: Spring Boot框架介绍
date: 2018-8-1 09:29:10
catalog: true
tags:
    - Spring Boot
---

## 你将学到：

1. 什么是微服务架构
2. Spring注解
3. 创建一个简单的Spring Boot应用
4. 可以使用JPA进行数据库操作

## 微服务的由来

单体架构：
![单体图片](/img/in-post/post-spring-boot/post-art1.png)
微服务架构：
![微服务](/img/in-post/post-spring-boot/post-art2.png)

## 什么是Spring Boot？

Spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来简化新Spring应用的初始搭建以及开发过程。

Spring Boot是简化Spring应用开发的一个框架，整个Spring技术栈的一个大整合，J2EE开发的一站式解决方案，采用`习惯优于配置`的方式，帮我们默认配置了一些内容，从而可以使我们轻松使用即可。

## Spring Boot的特性

- 为所有Spring开发提供一个从根本上更快，且随处可得的入门体验。
- 开箱即用，但通过不采用默认设置可以快速摆脱这种方式。
- 提供一系列大型项目常用的非功能性特征，比如：内嵌服务器，安全，指标，健康检测，外部化配置。
- 绝对没有代码生成，也不需要XML配置。

## 快速入门

### 基础要求

开发工具：JDK、Maven、IntelliJ IDEA
前期准备：
1、配置JDK环境
2、配置Maven环境，修改Maven仓库路径，使用阿里云镜像
3、 IntelliJ IDEA中配置Maven
知识储备：Maven、Spring注解

### 目录结构介绍

![img](/img/in-post/post-spring-boot/post01.png)

官方推荐目录：

![img](/img/in-post/post-spring-boot/post02.png)

* Application.java 建议放到根目录下面,主要用于做一些框架配置
* domain目录主要用于实体（Entity）与数据访问层（Repository）
* service 层主要是业务类代码
* controller 负责页面访问控制

### pom.xml介绍

![img](/img/in-post/post-spring-boot/post03.png)

### Application.java主程序

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

@SpringBootApplication = (默认属性)@Configuration + @EnableAutoConfiguration + @ComponentScan

### HelloController.java控制器介绍

```
@RestController
public class HelloController {
    @GetMapping("/hello/{name}")
    public String Hello(@PathVariable("name") String name) {
        return "Hello " + name;
    }

    @GetMapping("/hello")
    public String Hi(@RequestParam("name") String name) {
        return "Hello " + name;
    }
}
```

@RestController = @Controller + @ResponseBody

@GetMapping是一个组合注解，是@RequestMapping(method = RequestMethod.GET)的缩写

@PathVariable获取url中的数据
@RequestParam获取请求参数的值

### 项目属性配置-application.properties

server.port  --启动的端口
server.context-path --路径
spring.profiles.avtive=xxx --指明要使用的配置文件

### 启动运行方式

1. IDEA运行

![img](/img/in-post/post-spring-boot/post04.png)

2. mvn spring-boot:run

3. mvn package、java -jar 文件名.jar

运行成功：
![img](/img/in-post/post-spring-boot/post05.png)

浏览器访问：
![img](/img/in-post/post-spring-boot/post06.png)

## 数据库操作

- Spring-Data-JPA(Java Persistence API)：Spring 基于 ORM 框架、JPA 规范的基础上封装的一套JPA应用框架
- 依赖：spring-boot-starter-data-jpa、mysql-connector-java
- 配置：
```
spring.datasource.driver-class-name: com.mysql.jdbc.Driver
spring.datasource.url: jdbc:mysql://localhost:3306/test
spring.datasource.username: XXX
spring.datasource.password: XXX
spring.jpa.properties.hibernate.hbm2ddl.auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect
 ```
- 预先生成CURD方法、自定义简单查询、分页查询、自定义SQL查询、多表查询

## 深入学习

[Spring Boot官网](https://spring.io/projects/spring-boot)
[Spring Boot参考指南-中文版](https://qbgbook.gitbooks.io/spring-boot-reference-guide-zh/content/)
[开源书籍-《微服务：从设计到部署》](https://github.com/DocsHome/microservices)

> 技术上不是问题，意识比工具重要