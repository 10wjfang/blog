---
layout: post
title: Swagger访问安全的API
date: 2019-1-29 13:58:32
catalog: true
tags:
    - Swagger2
---

## 前言

Swagger UI提供了许多非常有用的功能，但是如果我们的API是安全的并且不可访问的话，我们不能真正使用其中的大部分。我们将使用SecurityScheme和SecurityContext支持配置Swagger以访问我们的安全API。

```java
@Bean
public Docket api() {
    return new Docket(DocumentationType.SWAGGER_2).select()
        .apis(RequestHandlerSelectors.any())
        .paths(PathSelectors.any())
        .build()
        .securitySchemes(securityScheme())
        .securityContexts(Arrays.asList(securityContext()));
}
```

## 安全配置

Swagger配置中定义一个  `SecurityConfiguration` `bean` - 并设置一些默认值：

```java
@Bean
SecurityConfiguration security() {
    return SecurityConfigurationBuilder.builder()
            .clientId("test-app-client-id")
            .clientSecret("test-app-client-secret")
            .realm("test-app-realm")
            .appName("test-app")
            .scopeSeparator(",")
            .additionalQueryStringParams(null)
            .useBasicAuthenticationWithAccessCodeGrant(false)
            .build();
}
```

## SecurityScheme

定义我们的SecurityScheme ; 这用于描述我们的API如何保护（基本认证，OAuth2，API密钥...）。这里我们使用API密钥。

```java
private List<ApiKey> securitySchemes() {
    List<ApiKey> apiKeyList= new ArrayList();
    apiKeyList.add(new ApiKey("x-auth-token", "token", "header"));
    return apiKeyList;
}
```

## Security Context

最后，我们需要为我们的示例API定义一个安全上下文：

```java
private SecurityContext securityContext() {
    return SecurityContext.builder()
            .securityReferences(Arrays.asList(new SecurityReference("x-auth-token", scopes())))
            .forPaths(path -> !StringUtils.equalsAny(path, permit_patterns))
            .build();
}
```

请注意，我们在此处使用的名称（参考 -  `x-auth-token`）与我们之前使用的名称在`SecurityScheme`中同步。

定义范围：

```java
private AuthorizationScope[] scopes() {
    AuthorizationScope[] scopes = { 
      new AuthorizationScope("global", "accessEverything")
    };
    return scopes;
}
```

## 参考

[具有OAuth安全API的Swagger UI](http://www.leftso.com/blog/393.html)