---
layout: post
title: Spring Cloud（八）：Hystrix熔断器
date: 2018-11-5 14:09:44
catalog: true
tags:
    - Spring Cloud
    - Hystrix
---

## 前言

在微服务架构中，通常会有多个服务层调用，基础服务的故障可能会导致级联故障，进而造成整个系统不可用。熔断器的作用就是实现快速失败，熔断器的原理很简单，就像电力过载保护器，当它侦测到错误时，会强制其以后的调用快速失败，不再访问远程服务。同时熔断器也会侦测错误是否已经修正，如果已经修正，应用程序会再次尝试调用远程服务。

## Hystrix

#### 断路器机制

断路器很好理解, 当Hystrix Command请求后端服务失败数量超过一定比例(默认50%), 断路器会切换到开路状态(Open)。这时所有请求会直接失败而不会发送到后端服务。断路器保持在开路状态一段时间后(默认5秒), 自动切换到半开路状态(HALF-OPEN)。这时会判断下一次请求的返回情况, 如果请求成功, 断路器切回闭路状态(CLOSED), 否则重新切换到开路状态(OPEN)。Hystrix的断路器就像我们家庭电路中的保险丝, 一旦后端服务不可用, 断路器会直接切断请求链, 避免发送大量无效请求影响系统吞吐量, 并且断路器有自我检测并恢复的能力。

#### Fallback

Fallback相当于是降级操作。对于查询操作, 我们可以实现一个fallback方法, 当请求后端服务出现异常的时候, 可以使用fallback方法返回的值。fallback方法的返回值一般是设置的默认值或者来自缓存。

#### 资源隔离

在Hystrix中, 主要通过线程池来实现资源隔离。通常在使用的时候我们会根据调用的远程服务划分出多个线程池. 例如调用产品服务的Command放入A线程池, 调用账户服务的Command放入B线程池。这样做的主要优点是运行环境被隔离开了。这样就算调用服务的代码存在bug或者由于其他原因导致自己所在线程池被耗尽时, 不会对系统的其他服务造成影响。但是带来的代价就是维护多个线程池会对系统带来额外的性能开销. 如果是对性能有严格要求而且确信自己调用服务的客户端代码不会出问题的话, 可以使用Hystrix的信号模式(Semaphores)来隔离资源。

## 使用Hystrix

在服务消费者端进行代码修改即可。

**1. 添加依赖**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-hystrix</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-consul-discovery</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

**2. 修改配置文件**

```yml
spring:
  application:
    name: spring-cloud-comsumer
  cloud:
    consul:
      host: localhost
      port: 8500
server:
  port: 8082
feign:
  hystrix:
    enabled: true
```

关键在于`feign. hystrix.enabled: true`，如果没有这句，会报`com.netflix.client.ClientException: Load balancer does not have available server for client`

**3. 创建回调类**

```java
@Component
public class HelloRemoteHystrix implements HelloRemote {
    @Override
    public String hello() {
        return "hello, this message send failed";
    }
}
```

**4. 添加fallback属性**

```java
@FeignClient(name = "spring-cloud-producer", fallback = HelloRemoteHystrix.class)
public interface HelloRemote {
    @GetMapping("/hello")
    String hello();
}
```

**5. 测试**

依次启动Consul、spring-cloud-producer、spring-cloud-consumer。
在浏览器输入：`http://localhost:8082/hello`
返回：`hello world`
接下来手动停止spring-cloud-producer，
在浏览器输入：`http://localhost:8082/hello`
返回：`hello, this message send failed`
再次启动spring-cloud-producer，
重新访问时不会立刻返回正确信息，多次刷新后才能正常访问。说明熔断成功。

## Hystrix Dashboard

Hystrix-dashboard是一款针对Hystrix进行实时监控的工具，通过Hystrix Dashboard我们可以在直观地看到各Hystrix Command的请求响应时间, 请求成功率等数据。但是只使用Hystrix Dashboard的话, 你只能看到单个应用内的服务信息, 这明显不够. 我们需要一个工具能让我们汇总系统内多个服务的数据并显示到Hystrix Dashboard上, 这个工具就是Turbine.

**1. 添加依赖**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

除了添加以上依赖外，还需要添加以下依赖，否则启动时会出现`NoClassDefFoundError: com/netflix/hystrix/contrib/javanica/aop/aspectj/HystrixCommandAspect`错误。

```xml
<dependency>
    <groupId>com.netflix.hystrix</groupId>
    <artifactId>hystrix-javanica</artifactId>
    <version>1.5.12</version>
</dependency>
```

**2. 修改启动类**

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
@EnableHystrixDashboard
@EnableCircuitBreaker
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Bean
    public ServletRegistrationBean getServlet(){
        HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
        ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
        registrationBean.setLoadOnStartup(1);
        registrationBean.addUrlMappings("/hystrix.stream");
        registrationBean.setName("HystrixMetricsStreamServlet");
        return registrationBean;
    }
}
```

注意：需要添加`ServletRegistrationBean`，因为springboot 2.0的默认路径不是 "/hystrix.stream"

**3. 测试**

访问`http://localhost:8082/hystrix`

![img](../../../../img/in-post/post-spring-cloud/img3.png)

我们暂时只演示单个应用的所以在输入框中输入： `http://localhost:8082/hystrix.stream` ，输入之后点击 monitor，进入页面。

![img](../../../../img/in-post/post-spring-cloud/img4.png)

各项指标解释：

![img](../../../../img/in-post/post-spring-cloud/img5.png)

## 参考

[
NoClassDefFoundError: com/netflix/hystrix/contrib/javanica/aop/aspectj/HystrixCommandAspect](https://blog.csdn.net/superdangbo/article/details/78773590)

[springcloud(四)：熔断器Hystrix](http://www.ityouknow.com/springcloud/2017/05/16/springcloud-hystrix.html)

[springcloud(五)：熔断监控Hystrix Dashboard和Turbine](http://www.ityouknow.com/springcloud/2017/05/18/hystrix-dashboard-turbine.html)