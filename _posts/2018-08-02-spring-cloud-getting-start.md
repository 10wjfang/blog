---
layout: post
title: Spring Cloud介绍
date: 2018-8-2 14:27:06
catalog: true
tags:
    - Spring Cloud
---

## 什么是Spring Cloud？

---

Spring Cloud是一系列框架的有序集合。它利用Spring Boot的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等，都可以用Spring Boot的开发风格做到一键启动和部署。Spring Cloud并没有重复制造轮子，它只是将目前各家公司开发的比较成熟、经得起实际考验的服务框架组合起来，通过Spring Boot风格进行再封装屏蔽掉了复杂的配置和实现原理，最终给开发者留出了一套简单易懂、易部署和易维护的分布式系统开发工具包。

## 特性

---

Spring Cloud专注于提供良好的开箱即用经验的典型用例和可扩展性机制覆盖。
- 分布式/版本化配置
- 服务注册和发现
- 路由
- service - to - service调用
- 负载均衡
- 断路器
- 分布式消息传递

## Spring Cloud组成

---

Spring Cloud的子项目，大致可分成两类，一类是对现有成熟框架”Spring Boot化”的封装和抽象，也是数量最多的项目；第二类是开发了一部分分布式系统的基础设施的实现，如Spring Cloud Stream扮演的就是kafka, ActiveMQ这样的角色。

| 组件 | 介绍 |
| :---: | :---: |
| Spring Cloud Config | 利用git集中管理程序的配置。配置资源直接映射到Spring的Environment，另一方面如果有需要也可以被非Spring应用使用。|
| Spring Cloud Netflix | 集成了许多Netflix的开源软件(Eureka, Hystrix, Zuul, Archaius, etc) |
| Spring Cloud Bus | 一个事件总线，利用分布式消息将服务和服务实例连接在一起，用于在一个集群中传播状态的变化，比如配置更改事件。 |
| Spring Cloud for Cloud Foundry | 利用Pivotal Cloudfoundry集成你的应用程序，提供了一个服务发现的实现也使得它很容易实现SSO和OAuth2保护资源，并创建一个Cloudfoundry服务代理。|
| Spring Cloud Cloud Foundry Service Broker | 为建立管理云托管服务的服务代理提供了一个起点。|
| Spring Cloud Cluster | 基于Zookeeper, Redis, Hazelcast, Consul实现的领导选举和平民状态模式的抽象和实现。|
| Spring Cloud Consul | 基于Hashicorp Consul实现的服务发现和配置管理。|
|Spring Cloud Security|在Zuul代理中为OAuth2 rest客户端和认证头转发提供负载均衡|
|Spring Cloud Sleuth|Spring Cloud 应用的分布式追踪系统，和Zipkin，HTrace，ELK兼容。|
|Spring Cloud Data Flow|一个云本地程序和操作模型，组成数据微服务在一个结构化的平台上。|
|Spring Cloud Stream|基于 Redis， Rabbit， Kafka实现的消息微服务，简单声明模型用以在Spring Cloud应用中收发消息。|
|Spring Cloud Stream Modules|可用于创建消息驱动的微服务。|
|Spring Cloud Task|短生命周期的微服务，为SpringBoot应用简单声明添加功能和非功能特性。比如说某些定时任务晚上就跑一次，或者某项数据分析临时就跑几次。|
|Spring Cloud Zookeeper|服务发现和配置管理基于Apache Zookeeper。|
|Spring Cloud Connectors|便于PaaS应用在各种平台上连接到后端像数据库和消息代理服务。|
|Spring Cloud Starters|SpringBoot风格starter项目，用以简化Spring Cloud客户端的依赖管理。（项目已经终止并且在Angel.SR2后的版本和其他项目合并）|
|Spring Cloud CLI|Spring Boot CLI 插件用Groovy快速的创建Spring Cloud组件应用。|

![img](../../../../img/in-post/post-spring-cloud/img1.png)

## Spring Cloud和Dubbo比较

---

### Dubbo简介

Dubbo 是阿里巴巴公司一个开源的高性能服务框架，致力于提供高性能和透明化的 RPC 远程服务调用方案，以及 SOA 服务治理方案，使得应用可通过高性能 RPC 实现服务的输出、输入功能和 Spring 框架无缝集成。Dubbo 包含远程通讯、集群容错和自动发现三个核心部分。

### 有何不同

| |Dubbo|Spring Cloud|
|:--:|:--:|:--:|
|服务注册中心|Zookeeper|	Spring Cloud Netflix Eureka|
|服务调用方式|	RPC|	REST API|
|服务监控	|Dubbo-monitor|	Spring Boot Admin|
|断路器	|不完善|	Spring Cloud Netflix Hystrix|
|服务网关|	无	|Spring Cloud Netflix Zuul|
|分布式配置|	无|	Spring Cloud Config|
|服务跟踪	|无	|Spring Cloud Sleuth|
|消息总线|	无	|Spring Cloud Bus|
|数据流	|无	|Spring Cloud Stream|
|批量任务|	无|	Spring Cloud Task|

这样对比是不够公平的，首先 Dubbo 是 SOA 时代的产物，它的关注点主要在于服务的调用，流量分发、流量监控和熔断。而 Spring Cloud 诞生于微服务架构时代，考虑的是微服务治理的方方面面，另外由于依托了 Spirng、Spirng Boot 的优势之上，两个框架在开始目标就不一致，**Dubbo 定位服务治理、Spirng Cloud 是一个生态**。

- Dubbo 支持更多的协议，如：rmi、hessian、http、webservice、thrift、memcached、redis 等。
- Dubbo 使用 RPC 协议效率更高，在极端压力测试下，Dubbo 的效率会高于 Spring Cloud 效率一倍多。
- Dubbo 有更强大的后台管理，Dubbo 提供的后台管理 Dubbo Admin 功能强大，提供了路由规则、动态配置、访问控制、权重调节、均衡负载等诸多强大的功能。
- 可以限制某个 IP 流量的访问权限，设置不同服务器分发不同的流量权重，并且支持多种算法，利用这些功能我们可以在线上做灰度发布、故障转移等，Spring Cloud 到现在还不支持灰度发布、流量权重等功能。

### 如何选择

如果公司对效率有极高的要求建议使用 Dubbo，相对比 RPC 的效率会比 HTTP 高很多；如果团队不想对技术架构做大的改造建议使用 Dubbo，Dubbo 仅仅需要少量的修改就可以融入到内部系统的架构中。但如果技术团队喜欢挑战新技术，建议选择 Spring Cloud，Spring Cloud 架构体系有有趣很酷的技术。如果公司选择微服务架构去重构整个技术体系，那么 Spring Cloud 是当仁不让之选，它可以说是目前最好的微服务框架没有之一。

最后，技术选型是一个综合的问题，需要考虑团队的情况、业务的发展以及公司的产品特征。最炫最酷的技术并不一定是最好的，选择适合自己团队、符合公司业务的框架才是最佳方案。技术的发展永远没有尽头，因此我们对技术也要保持空杯、保持饥饿、保持敬畏！

原文出处：[阿里Dubbo疯狂更新，关Spring Cloud什么事？](http://www.ityouknow.com/springcloud/2017/11/20/dubbo-update-again.html)

## 深入学习

---

[Spring Cloud系列教程](http://www.ityouknow.com/spring-cloud.html)

[Spring Cloud中文网-官方文档中文版](https://springcloud.cc/)

[Spring Cloud中国社区](http://springcloud.cn/)

[Spring Cloud官网](http://projects.spring.io/spring-cloud/)
