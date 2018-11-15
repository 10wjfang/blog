---
layout: post
title: Spring Boot（七）：Spring Security安全控制
date: 2018-11-15 17:45:40
catalog: true
tags:
    - Spring Boot
    - Spring Security
---

## 前言

前面介绍了Apache Shiro框架，本文将介绍Spring Security框架进行安全控制，进行对比。

## 快速上手

#### Web层

```java
@Controller
public class HelloController {
    @RequestMapping("/")
    public String index() {
        return "index";
    }

    @RequestMapping("/hello")
    public String hello() {
        return "hello";
    }

    @RequestMapping("/login")
    public String login() {
        return "login";
    }
}
```

- `/`：映射到index.html
- `/hello`：映射到hello.html
- `/login`：映射到login.html

#### web页面

- index.html

```html
<!DOCTYPE html>
<html lang="en" xmlns="http://www.w3.org/1999/xhtml" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Index</title>
</head>
<body>
<h1>Index</h1>
<a th:href="@{/hello}" >你好！</a>
</body>
</html>
```

- hello.html

```html
<!DOCTYPE html>
<html lang="en" xmlns="http://www.w3.org/1999/xhtml" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Hello</title>
</head>
<body>
<h1>Hello，[[${#httpServletRequest.remoteUser}]]</h1>
<form th:action="@{/logout}" method="post">
    <input type="submit" value="注销">
</form>
</body>
</html>
```

`[[${#httpServletRequest.remoteUser}]]`显示登录用户名

- login.html

```html
<!DOCTYPE html>
<html lang="en" xmlns="http://www.w3.org/1999/xhtml" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <div th:if="${param.error}">
        用户名或密码错误
    </div>
    <form th:action="@{/login}" method="post">
        用户名：<input type="text" name="username" />
        密码：<input type="password" name="password" />
        <input type="submit" value="登录" />
    </form>
</body>
</html>
```

#### 添加依赖

```xml
<dependencies>
    ...
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    ...
</dependencies>
```

#### 访问配置

创建Spring Security配置类`WebSecurityConfig`，具体如下：

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/", "/index").permitAll()
                .anyRequest().authenticated()
                .and()
                .formLogin()
                .loginPage("/login")
                .permitAll()
                .and()
                .logout()
                .permitAll();
    }

    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .passwordEncoder(new BCryptPasswordEncoder())
                .withUser("user").password(new BCryptPasswordEncoder().encode("12345")).roles("USER");
    }
}
```

- 通过`@EnableWebSecurity`注解开启Spring Security的功能
- 继承`WebSecurityConfigurerAdapter`，并重写它的方法来设置一些web安全的细节
- 重写`configure(HttpSecurity http)`方法
  - 通过`authorizeRequests()`定义哪些URL需要被保护、哪些不需要被保护。例如以上代码指定了/和/home不需要任何认证就可以访问，其他的路径都必须通过身份验证。
  - 通`过formLogin()`定义当需要用户登录时候，转到的登录页面。
- `configureGlobal(AuthenticationManagerBuilder auth)`方法，在内存中创建了一个用户，该用户的名称为user，密码为12345，用户角色为USER。

#### 测试

访问`http://localhost:8080/`，点击链接，跳转到`http://localhost:8080/hello`，这时候被重定向到`http://localhost:8080/login`，输入错误的用户名或密码，如果用户身份认证失败，页面就重定向到`/login?error`，输入正确的用户名和密码，跳转到`/hello`页面。

**注意：Spring security 5.0中新增了多种加密方式，也改变了密码的格式，否则会报错：There is no PasswordEncoder mapped for the id “null”**

## 参考

[Spring Boot中使用Spring Security进行安全控制](http://blog.didispace.com/springbootsecurity/)

[Spring Security 无法登陆，报错：There is no PasswordEncoder mapped for the id “null”](https://blog.csdn.net/canon_in_d_major/article/details/79675033)