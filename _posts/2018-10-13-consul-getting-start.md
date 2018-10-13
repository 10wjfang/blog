---
layout: post
title: Spring Cloud（六）：服务注册中心Consul    
date: 2018-10-13 11:42:05
catalog: true
tags:
    - Spring Cloud
    - Consul
---

## 为什么使用Consul

6月，知名服务注册与服务发现工具 Eureka 的 GitHub Wiki 上显示其 2.0 版本的开源工作已经停止。这意味着如果开发者继续使用作为 2.x 分支上现有工作 repo 一部分发布的代码库和工件，则将自负风险。，对此，专家建议开发者尽快将相关业务迁移到 Consul/ZooKeeper/Etcd 等工具上。

## 对比

|Feature|euerka|Consul|zookeeper|etcd|
|:--:|:--:|:--:|:--:|:--:|
|服务健康检查|可配支持|服务状态，内存，硬盘等|(弱)长连接，keepalive|连接心跳|
|多数据中心|—|支持|—|—|
|kv 存储服务|—|支持|支持|支持|
|一致性|—|raft|paxos|raft|
|cap|ap|ca|cp|cp|
|使用接口(多语言能力)|http（sidecar）|支持 http 和 dns|客户端|http/grpc|
|watch 支持|支持 long polling/大部分增量|全量/支持long |polling	支持|支持 long polling|
|自身监控|metrics|metrics|—|metrics|
|安全|—|acl /https|acl|https 支持（弱）|
|spring cloud 集成|已支持|已支持|已支持|已支持|

## Consul介绍

Consul 是 HashiCorp 公司推出的开源工具，用于实现分布式系统的服务发现与配置。

#### 功能

- 服务发现：Consul client 可以提供服务，例如api或mysql，也可以使用Consul client来发现指定服务的提供者。 使用DNS或HTTP，应用程序可以轻松找到他们所依赖的服务。
- 健康检查：Consul client 可以提供任何数量的健康检查，或者与给定的服务（“Web服务器是否返回200 OK”），或与本地节点（“内存利用率是否低于90％”）相关联。 可以使用此信息来监控集群运行状况，服务发现组件使用此信息将流量从有问题的主机中移除出去。
- KV Store：应用程序可以使用Consul的分层键/值存储，包括动态配置，功能标记，协调，leader选举等等。 简单的HTTP API使其易于使用。
- 多数据中心：Consul支持多个数据中心。 这意味着Consul的用户不必担心构建额外的抽象层以扩展到多个区域。

#### 角色

- client: 客户端, 无状态, 将 HTTP 和 DNS 接口请求转发给局域网内的服务端集群。
- server: 服务端, 保存配置信息, 高可用集群, 在局域网内与本地客户端通讯, 通过广域网与其它数据中心通讯。 每个数据中心的 server 数量推荐为 3 个或是 5 个。

![img](../../../../img/in-post/post-spring-cloud/consul-server-client.png)

#### 工作原理

![img](../../../../img/in-post/post-spring-cloud/consol_service.png)

- 1、当 Producer 启动的时候，会向 Consul 发送一个 post 请求，告诉 Consul 自己的 IP 和 Port
- 2、Consul 接收到 Producer 的注册后，每隔10s（默认）会向 Producer 发送一个健康检查的请求，检验Producer是否健康
- 3、当 Consumer 发送 GET 方式请求 /api/address 到 Producer 时，会先从 Consul 中拿到一个存储服务 IP 和 Port 的临时表，从表中拿到 Producer 的 IP 和 Port 后再发送 GET 方式请求 /api/address
- 4、该临时表每隔10s会更新，只包含有通过了健康检查的 Producer

## 简单入门

#### 1、安装Consul

从[Consul官方网站](https://www.consul.io/downloads.html)下载Consul最新版本，这里下载的是Windows版`1.3.0`。

进入Consul.exe当前目录，使用cmd启动Consul：
```cmd
consul agent -dev # -dev表示开发模式运行，另外还有-server表示服务模式运行
```

启动成功后访问：`http://localhost:8500`，可以看到 Consul 的管理界面：
![img](../../../../img/in-post/post-spring-cloud/img2.png)

#### 2、开发服务提供者

创建一个 spring-cloud-consul-producer 项目。
- Spring Boot版本：`2.0.3.RELEASE`
- Spring Cloud版本：`Finchley.RELEASE`

1、添加依赖包，pom.xml文件如下：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.3.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>

<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-consul-discovery</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

2、修改配置文件，application.properties文件如下：

```properties
spring.application.name=spring-cloud-consul-producer
server.port=8501
spring.cloud.consul.host=localhost
spring.cloud.consul.port=8500
#注册到consul的服务名称

spring.cloud.consul.discovery.service-name=service-producer
```

3、启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```

`@EnableDiscoveryClient` 注解表示支持服务发现

4、创建服务

```java
@RestController
public class HelloController {

    @RequestMapping("/hello")
    public String hello() {
        return "helle world!";
    }
}
```

启动成功后，访问`http://localhost:8500`可以看到添加了一个服务。

#### 3、开发服务消费者

创建一个 spring-cloud-consul-consumer 项目，使用两种方式调用服务。

1、添加依赖包，pom.xml文件如下：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.3.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>

<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-consul-discovery</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

2、修改配置文件，application.properties文件如下：

```properties
spring.application.name=spring-cloud-consul-consumer
server.port=8503
spring.cloud.consul.host=localhost
spring.cloud.consul.port=8500
spring.cloud.consul.discovery.register=false
```

3、启动类

```java
@SpringBootApplication
@EnableFeignClients
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

添加`@EnableFeignClients` 注解为了提供另外一种调用方式

4、创建服务

```java
@RestController
public class HelloController {
    @Autowired
    private LoadBalancerClient loadBalancerClient;
    @Autowired
    private HelloRemote helloRemote;

    @RequestMapping("/call")
    public String call() {
        ServiceInstance service = loadBalancerClient.choose("producer-service");
        System.out.println("服务地址：" + service.getUri());
        System.out.println("服务名称：" + service.getServiceId());
        String result = new RestTemplate().getForObject(service.getUri().toString() + "/hello", String.class);
        return result;
    }

    @RequestMapping("/call2")
    public  String call2() {
        return helloRemote.hello2();
    }
}
```

- call方法采用LoadBalancerClient获取获取和RestTemplate调用服务
- call2采用Feign调用服务，需添加接口：
```java
@FeignClient(name = "producer-service")
public interface HelloRemote {
    @RequestMapping("/hello")
    public String hello2();
}
```

#### 4、测试

分别访问`http://localhost:8503/call`和`http://localhost:8503/call2`，正常的话都能返回hello world！

## 总结

Consul使用起来比Eureka方便，不需要写服务注册中心，而且UI比较简洁。部署多个服务提供者就可以实现负载均衡。

## 参考

[Consul官方网站](https://www.consul.io/)

[springcloud(十三)：注册中心 Consul 使用详解](http://www.ityouknow.com/springcloud/2018/07/20/spring-cloud-consul.html)

[Eureka 2.0 开源工作宣告停止，继续使用风险自负](https://www.oschina.net/news/97521/eureka-2-0-discontinued)

[Consul 介绍](http://blog.cxiangnet.cn/2018/04/10/consul-%E4%BB%8B%E7%BB%8D/)