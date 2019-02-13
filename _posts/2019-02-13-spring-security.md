---
layout: post
title: Spring Security（三）：原理了解
date: 2019-2-13 13:54:05
catalog: true
tags:
    - Spring Security
---

## 前言

前面通过两个例子初步接触了`Spring Security`，但是每次处理认证授权时又是一头雾水，网上有很多文章知其然不知其所以然，所以有必要把`Spring Security`吃透，写一个比较完善的专题。

## Spring Security简介

`Spring Security`是一个能够为基于Spring的企业应用系统提供声明式的安全访问控制解决方案的安全框架。它提供了一组可以在Spring应用上下文中配置的Bean，充分利用了`Spring IoC`，`DI`（控制反转`Inversion of Control` ,`DI:Dependency Injection` 依赖注入）和`AOP`（面向切面编程）功能，为应用系统提供声明式的安全访问控制功能，减少了为企业系统安全控制编写大量重复代码的工作。

![img](../../../../img/in-post/post-spring-security/1.png)

## Spring Security认证过程

1、用户名和密码被封装成`Authentication`,通常情况下是`UsernamePasswordAuthenticationToken`这个实现类。

2、`AuthenticationManager` 身份管理器负责验证这个`Authentication`。

3、认证成功后，`AuthenticationManager`身份管理器返回一个被填充满了信息的（包括上面提到的权限信息，身份信息，细节信息，但密码通常会被移除）`Authentication`实例。

4、`SecurityContextHolder`安全上下文容器将第3步填充了信息的`Authentication`，通过`SecurityContextHolder.getContext().setAuthentication(…)`方法，设置到其中。

![img](../../../../img/in-post/post-spring-security/3.jpg)

## 核心组件

#### 1. 身份信息容器SecurityContextHolder

`SecurityContextHolder`用于存储安全上下文（`security context`）的信息。当前操作的用户是谁，该用户是否已经被认证，他拥有哪些角色权限…这些都被保存在`SecurityContextHolder`中。SecurityContextHolder默认使用ThreadLocal 策略来存储认证信息。Spring Security在用户登录时自动绑定认证信息到当前线程，在用户退出时，自动清除当前线程的认证信息。

#### 2. 身份信息Authentication

由这个顶级接口，我们可以得到用户拥有的权限信息列表，密码，用户细节信息，用户身份信息，认证信息。

#### 3. 身份认证器AuthenticationManager

`AuthenticationManager`（接口）是认证相关的核心接口，也是发起认证的出发点，因为在实际需求中，我们可能会允许用户使用用户名+密码登录，同时允许用户使用邮箱+密码，手机号码+密码登录，甚至，可能允许用户使用指纹登录（还有这样的操作？没想到吧），所以说`AuthenticationManager`一般不直接认证，`AuthenticationManager`接口的常用实现类`ProviderManager` 内部会维护一个`List<AuthenticationProvider>`列表，存放多种认证方式，实际上这是委托者模式的应用（`Delegate`）。也就是说，核心的认证入口始终只有一个：`AuthenticationManager`，不同的认证方式：用户名+密码（`UsernamePasswordAuthenticationToken`），邮箱+密码，手机号码+密码登录则对应了三个`AuthenticationProvider`。

`ProviderManager`源码：

```java
public class ProviderManager implements AuthenticationManager, MessageSourceAware,
        InitializingBean {
    // 维护一个AuthenticationProvider列表
    private List<AuthenticationProvider> providers = Collections.emptyList();
 
    public Authentication authenticate(Authentication authentication)
          throws AuthenticationException {
       Class<? extends Authentication> toTest = authentication.getClass();
       AuthenticationException lastException = null;
       Authentication result = null;
       // 依次认证
       for (AuthenticationProvider provider : getProviders()) {
          if (!provider.supports(toTest)) {
             continue;
          }
          try {
             result = provider.authenticate(authentication);
             if (result != null) {
                copyDetails(authentication, result);
                break;
             }
          }
          ...
          catch (AuthenticationException e) {
             lastException = e;
          }
       }
       // 如果有Authentication信息，则直接返回
       if (result != null) {
            if (eraseCredentialsAfterAuthentication
                    && (result instanceof CredentialsContainer)) {
                 //移除密码
                ((CredentialsContainer) result).eraseCredentials();
            }
             //发布登录成功事件
            eventPublisher.publishAuthenticationSuccess(result);
            return result;
       }
       ...
       //执行到此，说明没有认证成功，包装异常信息
       if (lastException == null) {
          lastException = new ProviderNotFoundException(messages.getMessage(
                "ProviderManager.providerNotFound",
                new Object[] { toTest.getName() },
                "No AuthenticationProvider found for {0}"));
       }
       prepareException(lastException, authentication);
       throw lastException;
    }
}
```

`ProviderManager` 中的`List`，会依照次序去认证，认证成功则立即返回，若认证失败则返回null，下一个`AuthenticationProvider`会继续尝试认证，如果所有认证器都无法认证成功，则`ProviderManager` 会抛出一个`ProviderNotFoundException`异常。

#### 4. UserDetails

UserDetails源码：

```java
public interface UserDetails extends Serializable {
   Collection<? extends GrantedAuthority> getAuthorities();
   String getPassword();
   String getUsername();
   boolean isAccountNonExpired();
   boolean isAccountNonLocked();
   boolean isCredentialsNonExpired();
   boolean isEnabled();
}
```

它和`Authentication`接口很类似，比如它们都拥有username，authorities，区分他们也是本文的重点内容之一。`Authentication`的`getCredentials()`与`UserDetails`中的`getPassword()`需要被区分对待，前者是用户提交的密码凭证，后者是用户正确的密码，认证器其实就是对这两者的比对。`Authentication`中的`getAuthorities()`实际是由`UserDetails`的`getAuthorities()`传递而形成的。

#### 5. UserDetailsService

UserDetailsService源码：

```java
public interface UserDetailsService {
   UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

`UserDetailsService`和`AuthenticationProvider`两者的职责常常被人们搞混。`UserDetailsService`只负责从特定的地方（通常是数据库）加载用户信息；`UserDetailsService`常见的实现类有`JdbcDaoImpl`，`InMemoryUserDetailsManager`，前者从数据库加载用户，后者从内存中加载用户，也可以自己实现`UserDetailsService`，通常这更加灵活。

![img](../../../../img/in-post/post-spring-security/2.png)

## 拦截器

- SecurityContextPersistenceFilter，用于将SecurityContext放入Session的Filter
- UsernamePasswordAuthenticationFilter, 登录认证的Filter,类似的还有CasAuthenticationFilter，BasicAuthenticationFilter等等。在这些Filter中生成用于认证的token，提交到AuthenticationManager，如果认证失败会直接返回。
- RememberMeAuthenticationFilter，通过cookie来实现remember me功能的Filter
- AnonymousAuthenticationFilter，如果一个请求在到达这个filter之前SecurityContext没有初始化，则这个filter会默认生成一个匿名SecurityContext。这在支持匿名用户的系统中非常有用。
- ExceptionTranslationFilter，捕获所有Spring Security抛出的异常，并决定处理方式
- FilterSecurityInterceptor， 权限校验的拦截器，访问的url权限不足时会抛出异常

## 参考

[Spring Security官网](https://projects.spring.io/spring-security/)

[Spring Security ( 一 ) ：架构概述](http://www.importnew.com/26712.html)

[Spring Security做JWT认证和授权](https://www.jianshu.com/p/d5ce890c67f7)