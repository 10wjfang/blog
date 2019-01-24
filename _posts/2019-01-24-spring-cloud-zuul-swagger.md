---
layout: post
title: Zuul整合Swagger
date: 2019-1-24 14:08:07
catalog: true
tags:
    - Zuul
    - Swagger2
---

## 简介

使用 Zuul 作为分布式系统网关，整个系统的文档。

## 快速上手

1、配置Swagger配置类SwaggerConfig

```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.fang.controller"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("Spring Boot使用Swagger2构建Restful API")
                .description("这里是描述")
                .termsOfServiceUrl("http://weijunfang.coding.me/blog/")
                .contact(new Contact("fang", "http://weijunfang.coding.me/blog/", "email@qq.com"))
                .version("1.0")
                .build();
    }
}
```

和单个服务配置Swagger一样。

2、增加Swagger资源文档配置类DocumentationConfig

```java
@Component
@Primary
public class DocumentationConfig implements SwaggerResourcesProvider{

    @Override
    public List<SwaggerResource> get() {
        List resources = new ArrayList<>();
        resources.add(swaggerResource("订单系统", "/order/v2/api-docs", "2.0"));
        resources.add(swaggerResource("支付系统", "/pay/v2/api-docs", "2.0"));
        return resources;
    }

    private SwaggerResource swaggerResource(String name, String location, String version) {
        SwaggerResource swaggerResource = new SwaggerResource();
        swaggerResource.setName(name);
        swaggerResource.setLocation(location);
        swaggerResource.setSwaggerVersion(version);
        return swaggerResource;
    }
}
```

## 测试

当访问 `http://localhost:8081/swagger-ui.html` 时，可在下拉菜单中选择不同微服务的API文档

可使用通过遍历路由方式自动添加所有微服务 API 文档

```java
@Component
@Primary
public class DocumentationConfig implements SwaggerResourcesProvider{

    private final RouteLocator routeLocator;

    public DocumentationConfig(RouteLocator routeLocator) {
        this.routeLocator = routeLocator;
    }

    @Override
    public List<SwaggerResource> get() {
        List resources = new ArrayList<>();
        List<Route> routes = routeLocator.getRoutes();
        System.out.println(Arrays.toString(routes.toArray()));
        routes.forEach(route -> {
            resources.add(swaggerResource(route.getId(), route.getFullPath().replace("**", "v2/api-docs"),"2.0"));
        });
        return resources;
    }

    private SwaggerResource swaggerResource(String name, String location, String version) {
        SwaggerResource swaggerResource = new SwaggerResource();
        swaggerResource.setName(name);
        swaggerResource.setLocation(location);
        swaggerResource.setSwaggerVersion(version);
        return swaggerResource;
    }
}
```

## 参考

[Swagger2 Zuul 整合](https://www.jianshu.com/p/af4ff19afa04)