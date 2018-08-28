---
layout: post
title: Spring Cloud（四）：配置中心Spring Cloud Config
date: 2018-8-25 09:50:00
catalog: true
tags:
    - Spring Cloud
    - Config
---

## Spring Cloud Config简介

在分布式系统中，由于服务数量巨多，为了方便服务配置文件统一管理，实时更新，所以需要分布式配置中心组件。在Spring Cloud中，有分布式配置中心组件spring cloud config ，它支持配置服务放在配置服务的内存中（即本地），也支持放在远程Git仓库或SVN中。

在spring cloud config 组件中，分两个角色，一是config server，二是config client。server提供配置文件的存储、以接口的形式将配置文件的内容提供出去，client通过接口获取数据、并依据此数据初始化自己的应用。

## 核心功能

- 提供服务端和客户端支持
- 集中管理各环境的配置文件
- 配置文件修改之后，可以快速的生效
- 可以进行版本管理
- 支持大的并发查询
- 支持各种语言

## SVN版本

### Server端

#### 1、在svn创建配置文件

首先在SVN上面创建了一个文件夹config-repo用来存放配置文件。

配置文件的命名要符合{spring.cloud.config.name}-{spring.cloud.config.profile}.preperties 或者后缀为.yml的配置文件；例如：fang-config-test.properties，spring.cloud.config.name=fang-config，spring.cloud.config.profile=test。

#### 2、添加依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
        <version>2.0.0.RC1</version>
    </dependency>
    <dependency>
        <groupId>org.tmatesoft.svnkit</groupId>
        <artifactId>svnkit</artifactId>
        <version>1.9.2</version>
    </dependency>
</dependencies>
```

#### 3、修改配置文件

```yml
server:
  port: 8002
spring:
  cloud:
    config:
      server:
        svn:
          uri: svn://*.*.*.*:8889/398svn/config-repo
          username: ***
          password: ***
        default-label: trunk
  profiles:
    active: subversion
  application:
    name: spring-cloud-config-server
```

和git版本稍有区别，需要显示声明subversion。

#### 4、启动类添加注解

添加@EnableConfigServer激活对配置中心的支持。

```java
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

#### 5、测试

访问`http://localhost:8002/fang-config/test`，返回信息如下：

```json
{
    "name": "fang-config",
    "profiles": [
        "test"
    ],
    "label": null,
    "version": "24",
    "state": null,
    "propertySources": [
        {
            "name": "http://*.*.*.*:8889/svn/398svn/config-repo/trunk/fang-config-test.properties",
            "source": {
                "fang.hello": "Hello World!"
            }
        }
    ]
}
```

访问：`http://localhost:8002/fang-config-test.properties`，返回信息如下：

```
fang.hello=Hello World!
```

### Client端

#### 1、添加依赖

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

#### 2、修改配置文件

额外增加bootstrap.yml配置文件。

application.yml如下：

```yml
spring:
  application: 
    name: spring-cloud-config-client
server: 
  port: 8003
```

bootstrap.yml如下：

```yml
spring:
  cloud:
    config:
      name: fang-config #对应{application}部分
      profile: test #对应{profile}部分
      uri: http://localhost:8002/ #配置中心的具体地址
      label: trunk # 对应svn分支
```

> bootstrap.yml（bootstrap.properties）用来程序引导时执行，应用于更加早期配置信息读取，如可以使用来配置application.yml中使用到参数等。bootstrap.yml 先于 application.yml 加载

#### 3、Controller测试

```java
@RestController
public class HelloController {
    @Value("${fang.hello}")
    private String hello;

    @GetMapping("/hello")
    public String sayHello() {
        return this.hello;
    }
}
```

启动项目后访问：`http://localhost:8003/hello`，返回`Hello World!`说明客户端获取配置信息一切正常。

修改fang-config-test.properties中fang.hello的值，再次请求接口，发现接口返回值没变。原因是客户端不能主动感知配置的变化。

#### 4、开启更新机制

添加依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

> spring-boot-starter-actuator：监控系统健康情况的工具

给HelloController类加上`@RefreshScope`注解，使用该注解的类，会在接到SpringCloud配置中心配置刷新的时候，自动将新的配置更新到该类对应的字段中。

```java
@RestController
@RefreshScope
public class HelloController {
    @Value("${fang.hello}")
    private String hello;

    @RequestMapping("/hello")
    public String sayHello() {
        return this.hello;
    }
}
```

注意：Spring boot 2.0的改动较大，/bus/refresh全部整合到actuator里面了，变成了/actuator/bus-refresh,所以之前1.x的management.security.enabled全部失效，不适用于2.0 。否则执行`curl -X POST http://localhost:8002/refresh`返回404

所以，需要给Server端和Client的配置文件都加上以下内容：

```yml
management:
  endpoints:
    web:
      exposure:
        include: refresh
```

目的是把refresh接口暴露出来，执行`curl -X POST http://localhost:8003/actuator/refresh`，返回`["config.client.version","fang.hello"]`，说明已经更新了，再次访问`http://localhost:8003/hello`就可以看到最新的值。

## Git版本

和SVN版本的区别在于服务器端的配置文件不同：

```yml
server:
  port: 8002
spring:
  application:
    name: spring-cloud-config-server
  cloud:
    config:
      server:
        git:
          uri: http://xxxx/    # 配置git仓库的地址
          search-paths: config-repo                             # git仓库地址下的相对地址，可以配置多个，用,分割。
          username:                                             # git仓库的账号
          password:                                             # git仓库的密码
```

还有不需要添加依赖svnkit。

## 参考

[springcloud(六)：配置中心git示例](http://www.ityouknow.com/springcloud/2017/05/22/springcloud-config-git.html)

[springcloud(七)：配置中心svn示例和refresh](http://www.ityouknow.com/springcloud/2017/05/23/springcloud-config-svn-refresh.html)