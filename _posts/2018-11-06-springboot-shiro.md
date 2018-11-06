---
layout: post
title: Spring Boot（五）：登录认证和权限管理Apache Shiro
date: 2018-11-5 16:00:11
catalog: true
tags:
    - Spring Boot
    - Apache Shiro
---

## 前言

我们在编写Web应用时，经常需要对页面做一些安全控制，要实现访问控制的方法多种多样，可以通过AOP、拦截器实现，也可以通过框架实现（如：Apache Shiro、Spring Security），但由于Spring Security过于庞大和复杂，大多数公司会选择Apache Shiro来使用，本文将具体介绍在Spring Boot中如何使用Apache Shiro来进行登录认证和权限管理。

## Apache Shiro

Apache Shiro是一个功能强大、灵活的，开源的安全框架。它可以干净利落地处理`身份验证`、`授权`、`企业会话管理`和`加密`。

#### 作用

- 验证用户身份
- 用户访问权限控制，比如：1、判断用户是否分配了一定的安全角色。2、判断用户是否被授予完成某个操作的权限
- 在非 web 或 EJB 容器的环境下可以任意使用Session API
- 可以响应认证、访问控制，或者 Session 生命周期中发生的事件
- 可将一个或以上用户安全数据源数据组合成一个复合的用户 “view”(视图)
- 支持单点登录(SSO)功能
- 支持提供“Remember Me”服务，获取用户关联信息而无需登录

#### 功能特点

Shiro包含10个内容：

![img](../../../../img/in-post/post-spring-cloud/img8.png)

- **Authentication（认证）**：用户身份识别，通常被称为用户“登录”
- **Authorization（授权）**：访问控制。比如某个用户是否具有某个操作的使用权限。
- **Session Management（会话管理）**：特定于用户的会话管理,甚至在非web 或 EJB 应用程序。
- **Cryptography（加密）**：在对数据源使用加密算法加密的同时，保证易于使用。
- Web支持：Shiro 提供的 web 支持 api ，可以很轻松的保护 web 应用程序的安全。
- 缓存：缓存是 Apache Shiro 保证安全操作快速、高效的重要手段。
- 并发：Apache Shiro 支持多线程应用程序的并发特性。
- 测试：支持单元测试和集成测试，确保代码和预想的一样安全。
- “Run As”：这个功能允许用户假设另一个用户的身份(在许可的前提下)。
- “Remember Me”：跨 session 记录用户的身份，只有在强制需要时才需要登录。

#### 运行原理

![img](../../../../img/in-post/post-spring-cloud/img9.png)

- Subject：当前用户，Subject 可以是一个人，但也可以是第三方服务、守护进程帐户、时钟守护任务或者其它–当前和软件交互的任何事件。
- SecurityManager：管理所有Subject，SecurityManager 是 Shiro 架构的核心，配合内部安全组件共同组成安全伞。
- Realms：用于进行权限信息的验证，我们自己实现。Realm 本质上是一个特定的安全 DAO：它封装与数据源连接的细节，得到Shiro 所需的相关的数据。在配置 Shiro 的时候，你必须指定至少一个Realm 来实现认证（authentication）和/或授权（authorization）。

我们需要实现Realms的Authentication 和 Authorization。其中 Authentication 是用来验证用户身份，Authorization 是授权访问控制，用于对用户进行的操作授权，证明该用户是否允许进行当前操作，如访问某个链接，某个资源文件等。

#### 过滤器

|过滤器简称|对应的 Java 类|
|:--|:--|
|anon|org.apache.shiro.web.filter.authc.AnonymousFilter|
|authc	|org.apache.shiro.web.filter.authc.FormAuthenticationFilter
|authcBasic	|org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter
|perms	|org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter
|port|	org.apache.shiro.web.filter.authz.PortFilter
|rest	|org.apache.shiro.web.filter.authz.HttpMethodPermissionFilter
|roles	|org.apache.shiro.web.filter.authz.RolesAuthorizationFilter
|ssl|	org.apache.shiro.web.filter.authz.SslFilter
|user|	org.apache.shiro.web.filter.authc.UserFilter
|logout|	org.apache.shiro.web.filter.authc.LogoutFilter
|noSessionCreation	|org.apache.shiro.web.filter.session.NoSessionCreationFilter|

解释：

```
/admins/**=anon               # 表示该 uri 可以匿名访问
/admins/**=auth               # 表示该 uri 需要认证才能访问
/admins/**=authcBasic         # 表示该 uri 需要 httpBasic 认证
/admins/**=perms[user:add:*]  # 表示该 uri 需要认证用户拥有 user:add:* 权限才能访问
/admins/**=port[8081]         # 表示该 uri 需要使用 8081 端口
/admins/**=rest[user]         # 相当于 /admins/**=perms[user:method]，其中，method 表示  get、post、delete 等
/admins/**=roles[admin]       # 表示该 uri 需要认证用户拥有 admin 角色才能访问
/admins/**=ssl                # 表示该 uri 需要使用 https 协议
/admins/**=user               # 表示该 uri 需要认证或通过记住我认证才能访问
/logout=logout                # 表示注销,可以当作固定配置
```

**注意：**

anon，authcBasic，auchc，user 是认证过滤器。

perms，roles，ssl，rest，port 是授权过滤器。

## 快速上手

**1. 添加依赖**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.shiro</groupId>
        <artifactId>shiro-spring</artifactId>
        <version>1.4.0</version>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**2. 配置文件**

```yml
spring:
  datasource:
    url: jdbc:mysql://192.168.241.131:3306/demo?useUnicode=true&characterEncoding=utf-8
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver

  jpa:
    database: mysql
    show-sql: true
    hibernate:
      ddl-auto: update
      naming:
        strategy: org.hibernate.cfg.DefaultComponentSafeNamingStrategy
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL5Dialect

  thymeleaf:
    cache: false
    mode: LEGACYHTML5
```

thymeleaf的配置是为了去掉html的校验

**3. 继承AuthorizingRealm**

```java
public class MyShiroRealm extends AuthorizingRealm {
    @Resource
    private IUserService userInfoService;
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        System.out.println("权限配置-->MyShiroRealm.doGetAuthorizationInfo()");
        SimpleAuthorizationInfo authorizationInfo = new SimpleAuthorizationInfo();
        UserInfo userInfo  = (UserInfo)principals.getPrimaryPrincipal();
        for(SysRole role:userInfo.getRoleList()){
            System.out.println(role.getRole());
            authorizationInfo.addRole(role.getRole());
            for(SysPermission p:role.getPermissions()){
                System.out.println(p.getPermission());
                authorizationInfo.addStringPermission(p.getPermission());
            }
        }
        return authorizationInfo;
    }

    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        System.out.println("MyShiroRealm.doGetAuthenticationInfo()");
        //获取用户的输入账号
        String username = (String)token.getPrincipal();
        System.out.println(username);
        UserInfo userInfo =userInfoService.findByUsername(username);
        if(userInfo == null){
            return null;
        }
        SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(
                userInfo, //用户名
                userInfo.getPassword(), //密码
                ByteSource.Util.bytes(userInfo.getCredentialsSalt()),//salt=username+salt
                getName()  //realm name
        );
        return authenticationInfo;
    }
}
```

**4. 创建ShiroConfig**

```java
@Configuration
public class ShiroConfig {
    @Bean
    public ShiroFilterFactoryBean shirFilter(SecurityManager securityManager) {
        System.out.println("ShiroConfiguration.shirFilter()");
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager);
        //拦截器.
        Map<String,String> filterChainDefinitionMap = new LinkedHashMap<String,String>();
        // 配置不会被拦截的链接 顺序判断
        filterChainDefinitionMap.put("/static/**", "anon");
        //配置退出 过滤器,其中的具体的退出代码Shiro已经替我们实现了
        filterChainDefinitionMap.put("/logout", "logout");
        //<!-- 过滤链定义，从上向下顺序执行，一般将/**放在最为下边 -->:这是一个坑呢，一不小心代码就不好使了;
        //<!-- authc:所有url都必须认证通过才可以访问; anon:所有url都都可以匿名访问-->
        //filterChainDefinitionMap.put("/register", "anon");
        //filterChainDefinitionMap.put("/user/**", "authc");
        filterChainDefinitionMap.put("/**", "authc");
        // 如果不设置默认会自动寻找Web工程根目录下的"/login.jsp"页面
        shiroFilterFactoryBean.setLoginUrl("/login");
        // 登录成功后要跳转的链接
        shiroFilterFactoryBean.setSuccessUrl("/user/list");

        //未授权界面;
        shiroFilterFactoryBean.setUnauthorizedUrl("/403");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        return shiroFilterFactoryBean;
    }

    @Bean
    public HashedCredentialsMatcher hashedCredentialsMatcher(){
        HashedCredentialsMatcher hashedCredentialsMatcher = new HashedCredentialsMatcher();
        hashedCredentialsMatcher.setHashAlgorithmName("md5");//散列算法:这里使用MD5算法;
        hashedCredentialsMatcher.setHashIterations(2);//散列的次数，比如散列两次，相当于 md5(md5(""));
        return hashedCredentialsMatcher;
    }

    @Bean
    public MyShiroRealm myShiroRealm(){
        MyShiroRealm myShiroRealm = new MyShiroRealm();
        myShiroRealm.setCredentialsMatcher(hashedCredentialsMatcher());
        return myShiroRealm;
    }

    @Bean
    public SecurityManager securityManager(){
        DefaultWebSecurityManager securityManager =  new DefaultWebSecurityManager();
        securityManager.setRealm(myShiroRealm());
        return securityManager;
    }
}
```

**注意：** filterChainDefinitionMap.put("/**", "authc");

**5. 测试**

访问：`http://localhost:8080/`，自动跳转到登录页面，登录后跳转到`http://localhost:8080/user/list`

## 参考

[springboot(十四)：springboot整合shiro-登录认证和权限管理](http://www.ityouknow.com/springboot/2017/06/26/springboot-shiro.html)

[Shiro 基础教程](https://www.cnblogs.com/moonlightL/p/8126910.html)