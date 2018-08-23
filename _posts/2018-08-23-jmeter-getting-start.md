---
layout: post
title: 负载均衡Ribbon基本使用
date: 2018-8-21 11:43:52
catalog: true
tags:
    - Spring Cloud
    - Ribbon
---

> - Ribbon版本：1.4.4.RELEASE
> - Eureka版本：1.4.4.RELEASE
> - Spring Boot版本：2.0.3.RELEASE
> - Spring Cloud版本：Finchley.RELEASE

## Ribbon介绍

Ribbon是Netflix发布的负载均衡器，它有助于控制HTTP和TCP的客户端的行为。为Ribbon配置服务提供者地址后，Ribbon就可基于某种负载均衡算法，自动地帮助服务消费者去请求。Ribbon默认为我们提供了很多负载均衡算法，例如轮询、随机等。当然，我们也可为Ribbon实现自定义的负载均衡算法。

>  Feign已经默认使用了Ribbon

## 和RestTemplate相结合

传统情况下在java代码里访问restful服务，一般使用Apache的HttpClient。不过此种方法使用起来太过繁琐。spring提供了一种简单便捷的模板类来进行操作，这就是RestTemplate。为了实现负载均衡，可以给RestTemplate添加注解@LoadBalanced。

> 依赖：spring-boot-starter-web，避免报Unregistering JMX-exposed beans on shutdown错误

## 原理

Ribbon的负载均衡，主要通过LoadBalancerClient来实现的，而LoadBalancerClient具体交给了ILoadBalancer来处理，ILoadBalancer通过配置IRule、IPing等信息，并向EurekaClient获取注册列表的信息，并默认10秒一次向EurekaClient发送“ping”,进而检查是否更新服务列表，最后，得到注册列表后，ILoadBalancer根据IRule的策略进行负载均衡。

而RestTemplate 被@LoadBalance注解后，能过用负载均衡，主要是维护了一个被@LoadBalance注解的RestTemplate列表，并给列表中的RestTemplate添加拦截器，进而交给负载均衡器去处理。

## 实践

#### 准备工作

1、启动Eureka服务注册中心。

2、启动Eureka服务提供方。

3、新建一个spring boot项目，修改pom.xml文件

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.3.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
        <version>1.4.4.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-ribbon</artifactId>
        <version>1.4.4.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Finchley.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <fork>true</fork>
            </configuration>
        </plugin>
    </plugins>
</build>
```

> 注意：
> 1. 需要添加spring-cloud-starter-eureka依赖，不然发现不了服务，其实 Ribbon 也支持脱离 Eureka 使用，以适应不具备 Eureka 使用条件，需要修改application.properties文件：
`service-provider.ribbon.listOfServers=http://localhost:8000`
> 2. Spring Cloud和Spring Boot版本要匹配

4、新建Application.java文件

```java
@EnableDiscoveryClient
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

- @EnableDiscoveryClient：添加发现服务能力
- @LoadBalanced注解开启均衡负载能力，如果不加@LoadBalanced注解的话，会报java.net.UnknownHostException

5、添加application.properties

```properties
server.port=8010
spring.application.name=spring-cloud-ribbon
eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka/
```

6、创建Controller类

```java
@RestController
public class HelloController {
    @Autowired
    private RestTemplate restTemplate;

    @RequestMapping("/hello/{name}")
    public String hello(@PathVariable(name = "name") String name) {
        return restTemplate.getForObject("http://SPRING-CLOUD-PRODUCER/hello?name="+name, String.class);
    }
}
```

- SPRING-CLOUD-PRODUCER：服务提供方应用程序名称，不需要加端口

## 总结

在Spring cloud 中服务之间通过restful方式调用有两种方式 
- restTemplate+Ribbon 
- feign

ribbon是对服务之间调用做负载，是服务之间的负载均衡，zuul是可以对外部请求做负载均衡。 

## 参考

[深入理解Ribbon之源码解析](https://blog.csdn.net/forezp/article/details/74820899)

[Spring Cloud构建微服务架构（二）服务消费者](http://blog.didispace.com/springcloud2/)