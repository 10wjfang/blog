---
layout: post
title: Spring Cloud（九）：分布式链路跟踪Sleuth和Zipkin
date: 2018-11-5 16:00:11
catalog: true
tags:
    - Spring Cloud
    - Sleuth
    - Zipkin
---

## 前言

随着业务发展，系统拆分导致系统调用链路愈发复杂一个前端请求可能最终需要调用很多次后端服务才能完成，当整个请求变慢或不可用时，我们是无法得知该请求是由某个或某些后端服务引起的，这时就需要解决如何快读定位服务故障点，以对症下药。于是就有了分布式系统调用跟踪的诞生。

## Spring Cloud Sleuth

Spring Cloud Sleuth为服务之间调用提供链路追踪。通过Sleuth可以很清楚的了解到一个服务请求经过了哪些服务，每个服务处理花费了多长。从而让我们可以很方便的理清各微服务间的调用关系。

Spring Cloud Sleuth可以结合Zipkin，将信息发送到Zipkin，利用zipkin的存储来存储信息，利用zipkin ui来展示数据。

## Zipkin

Zipkin是一种分布式跟踪系统，由Twitter公司开源。它有助于收集解决微服务架构中延迟问题所需的时序数据。它管理这些数据的收集和查找。Zipkin的设计基于 Google Dapper论文。

应用程序用于向Zipkin报告时间数据。Zipkin用户界面还提供了一个依赖关系图，显示每个应用程序有多少跟踪请求。如果您正在解决延迟问题或错误问题，则可以根据应用程序，跟踪长度，注释或时间戳过滤或排序所有跟踪。选择跟踪后，您可以看到每个跨度所需的总跟踪时间百分比，从而可以识别问题应用程序。

## 快速上手

Spring Boot 2.0之后，使用EnableZipkinServer创建自定义的zipkin服务器已经被废弃，将无法启动。

**1. 下载Zipkin**

下载地址：[https://search.maven.org/remote_content?g=io.zipkin.java&a=zipkin-server&v=LATEST&c=exec](https://search.maven.org/remote_content?g=io.zipkin.java&a=zipkin-server&v=LATEST&c=exec)，通过`java -jar zipkin.jar`执行

![img](../../../../img/in-post/post-spring-cloud/img6.png)

在浏览器访问`http://localhost:9411`，可以看到页面：

![img](../../../../img/in-post/post-spring-cloud/img7.png)

**2. 项目添加Zipkin支持**

在项目spring-cloud-producer和spring-cloud-consumer中添加zipkin的支持。注意：因为我下载的Zipkin是2.11.8版本，需要Spring Boot 2.1.0版本才能跟踪请求。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```

修改配置文件：

```yml
spring:
  zipkin:
    base-url: http://localhost:9411/
    sender:
      type: web
  sleuth:
    sampler:
      probability: 1.0
    trace-id128: true
```

spring.zipkin.base-url指定了Zipkin服务器的地址，spring.sleuth.sampler.probability将采样比例设置为1.0，也就是全部都需要。

**3. 测试**

在浏览器输入：`http://localhost:8082/hello`，，然后再打开地址： http://localhost:9411/zipkin/点击对应按钮进行查看。

## 参考

[springcloud(十二)：使用Spring Cloud Sleuth和Zipkin进行分布式链路跟踪](http://www.ityouknow.com/springcloud/2018/02/02/spring-cloud-sleuth-zipkin.html)

[springcloud(五)：熔断监控Hystrix Dashboard和Turbine](http://www.ityouknow.com/springcloud/2017/05/18/hystrix-dashboard-turbine.html)

[Zipkin源码](https://github.com/openzipkin/zipkin/tree/master/zipkin-server)

[spring cloud Finchley.RC1 zipkin 无法将client的追踪信息发送到zipkinserver中](http://www.springcloud.cn/view/255)