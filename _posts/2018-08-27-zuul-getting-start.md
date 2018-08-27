---
layout: post
title: 服务网关Spring Cloud Zuul    
date: 2018-8-27 14:25:04
catalog: true
tags:
    - Spring Cloud
    - Zuul
---

## 简介

spring cloud zuul是netflix提供的一个组件，功能类似于nginx，用于反向代理，可以提供动态路由、监控、授权、安全、调度等边缘服务。

## 什么是API Gateway?

微服务场景下，每一个微服务对外暴露了一组细粒度的服务。客户端的请求可能会涉及到一串的服务调用，如果将这些微服务都暴露给客户端，那么会增加客户端代码的复杂度。

参考GOF设计模式中的Facade模式，将细粒度的服务组合起来提供一个粗粒度的服务，所有请求都导入一个统一的入口，那么整个服务只需要暴露一个api，对外屏蔽了服务端的实现细节，也减少了客户端与服务器的网络调用次数。这就是api gateway。

## Zuul的功能

- 动态路由
- 监控
- 安全
- 认证鉴权
- 压力测试
- 金丝雀测试
- 审查
- 服务迁移
- 负载剪裁
- 静态应答处理

## 简单入门

#### 1、添加依赖

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.0.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zuul</artifactId>
        <version>2.0.0.M2</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
        <version>2.0.0.M2</version>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Finchley.RC1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

引入eureka的目的是为了后续与eureka做整合，使用serviceId做路由。

#### 2、修改配置文件

```properties
spring.application.name=gateway-service-zuul
server.port=8888

eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka/
```

Zuul路由有两种方式：
- url映射：zuul.routes.api-a.url: http://192.168.1.10:8081
- serviceId映射
  - 默认serviceId：path默认是`微服务在Eureka上的serviceId/**`
  - 指定serviceId：zuul.routes.api-a.path=/**
zuul.routes.api-a.serviceId=spring-cloud-producer

#### 3、修改启动类

添加@EnableZuulProxy注解

```java
@SpringBootApplication
@EnableZuulProxy
@EnableDiscoveryClient
public class GatewayServiceZuulApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayServiceZuulApplication.class, args);
    }
}
```

#### 4、测试

启动项目，访问`http://localhost:8888/spring-cloud-producer/hello?name=fang`，返回`hello, fang!`，说明访问网关的请求转发到了spring-cloud-producer。

## 参考

[Spring Cloud技术分析（4）- spring cloud zuul](http://tech.lede.com/2017/05/16/rd/server/SpringCloudZuul/)

[spring cloud 中国社区](http://docs.springcloud.cn/user-guide/zuul/#zuul_3)

[springcloud(十)：服务网关zuul初级篇](http://www.ityouknow.com/springcloud/2017/06/01/gateway-service-zuul.html)