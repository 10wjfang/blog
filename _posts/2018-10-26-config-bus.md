---
layout: post
title: Spring Cloud（七）：配置中心和消息总线Spring Cloud Bus
date: 2018-8-25 09:50:00
catalog: true
tags:
    - Spring Cloud
    - Spring Cloud Bus
---

## 前言

前面讲到通过执行`refresh`使客户端获得最新的配置信息，当客户端越来越多时，就需要每个客户端都执行一遍，这种方案就不合适。所以使用Spring Cloud Bus来解决这个问题。本节我们讨论使用Spring Cloud Bus实现配置的自动更新。

## Spring Cloud Bus

Spring Cloud Bus提供了批量刷新配置的机制，它使用轻量级的消息代理（例如RabbitMQ、Kafka等）连接分布式系统的节点，这样就可以通过Spring Cloud Bus广播配置的变化或者其他的管理指令。使用Spring Cloud Bus后的架构如图所示：

![img](../../../../img/in-post/post-spring-cloud/bus.png)

#### 架构改进

上面的架构是通过请求某个微服务的/bus/refresh端点的方式来实现配置刷新，但这种方式并不优雅。原因如下：

- 打破了微服务的职责单一性。微服务本身是业务模块，它本不应该承担配置刷新的职责。

- 破坏了微服务各节点的对等性。

- 有一定的局限性。例如，微服务在迁移时，它的网络地址常常会发生变化，此时如果想要做到自动刷新，那就不得不修改WebHook的配置。

修改后的架构如图所示：

![img](../../../../img/in-post/post-spring-cloud/bus2.png)

## 项目示例

在第四节的项目基础上进行改造。

#### 服务端spring-cloud-config-server

1、添加`spring-cloud-starter-bus-amqp`依赖，增加对消息总线的支持

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.1.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.tmatesoft.svnkit</groupId>
        <artifactId>svnkit</artifactId>
        <version>1.9.2</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-consul-discovery</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Finchley.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

2、配置文件增加RabbitMq的相关配置

```yml
server:
  port: 8002
spring:
  cloud:
    config:
      server:
        svn:
          uri: http://*.*.*.*:8889/svn/398svn/config-repo
          username: ***
          password: ***
        default-label: trunk
    consul:
      host: localhost
      port: 8500
  profiles:
    active: subversion
  application:
    name: spring-cloud-config-server
  rabbitmq:
    host: 192.168.241.136
    port: 5672
    username: user
    password: 123456
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

#### 客户端spring-cloud-config-client

1、添加`spring-cloud-starter-bus-amqp`依赖，增加对消息总线的支持

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.1.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-consul-discovery</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Finchley.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

2、配置文件增加RabbitMq的相关配置

application.yml文件：

```yml
spring:
  application:
    name: spring-cloud-config-client
  cloud:
    consul:
      host: localhost
      port: 8500
  rabbitmq:
    host: 192.168.241.136
    port: 5672
    username: user
    password: 123456
server:
  port: 8003
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

bootstrap.yml文件：

```yml
spring:
  cloud:
    config:
      name: fang-config
      profile: test
      # uri: http://localhost:8002/
      label: trunk
      discovery:
        enabled: true
        service-id: spring-cloud-config-server
      fail-fast: true
    bus:
      trace:
        enabled: true #开启消息跟踪
```

#### 测试

依次启动consul、spring-cloud-config-server、spring-cloud-config-client项目。

|服务|IP|端口|
|:--:|:--:|:--:|
|consul|127.0.0.1|8500|
|spring-cloud-config-server|127.0.0.1|8002|
|spring-cloud-config-client|127.0.0.1|8003|
|rabbitmq|192.168.241.136|5672|

**注意：**
- /actuator/refresh ：刷新单个节点
- /actuator/bus-refresh: 刷新所有节点

修改`fang-config-test.properties`的内容，提交到SVN，在浏览器访问`http://localhost:8002/fang-config-test.properties`，可以看到内容已经修改。在win下使用命令执行`/actuator/bus-refresh`。

```
curl -X POST http://localhost:8002/actuator/bus-refresh
```

执行完成后，浏览器访问`http://localhost:8003/hello`，如果返回修改后的内容，说明客户端已经拿到最新的配置信息。

## 参考

[springcloud(九)：配置中心和消息总线（配置中心终结版）](http://www.ityouknow.com/springcloud/2017/05/26/springcloud-config-eureka-bus.html)

[Config Server——使用Spring Cloud Bus自动刷新配置](http://www.itmuch.com/spring-cloud/spring-cloud-bus-auto-refresh-configuration/)