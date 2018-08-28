---
layout: post
title: Spring Boot（三）：使用Swagger2构建API文档
date: 2018-8-28 10:27:25
catalog: true
tags:
    - Spring Boot
    - Swagger2
---

## 前言

手写Api文档的几个痛点：

- 文档需要更新的时候，需要再次发送一份给前端，也就是文档更新交流不及时。
- 接口返回结果不明确
- 不能直接在线测试接口，通常需要使用工具，比如postman
- 接口文档太多，不好管理

## 简介

为了解决上面这样的问题，本文将介绍RESTful API的重磅好伙伴Swagger2，它可以轻松的整合到Spring Boot中，并与Spring MVC程序配合组织出强大RESTful API文档。它既可以减少我们创建文档的工作量，同时说明内容又整合入实现代码中，让维护文档和修改代码整合为一体，可以让我们在修改代码逻辑的同时方便的修改文档说明。

## 简单入门

#### 1、添加依赖

在pom.xml添加Swagger2的依赖

```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.7.0</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.7.0</version>
</dependency>
```

可能会出现@EnableSwagger2注解找不到依赖的问题。

> 解决：这里Spring Cloud用的是2.0.1.RELEASE版，Swagger2最新版2.8.0不支持，需要根据Spring Cloud的版本选择Swagger2的版本，这里选择2.7.0，检查Maven包中是否有swagger-ui这个包

#### 2、创建配置类

在Application.java同级创建Swagger2的配置类。

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

- @Configuration：让Spring加载改类配置
- @EnableSwagger2：启用Swagger2
- apiInfo()：用来创建该Api的基本信息
- select()函数：返回一个ApiSelectorBuilder实例用来控制哪些接口暴露给Swagger来展现

#### 3、添加文档内容

```java
@Api(value = "/", tags = "收藏夹模块")
@RestController
public class CateController extends BaseController {
    @Autowired
    CategoryService service;

    @ApiOperation(value = "获取收藏夹列表", notes = "根据用户ID获取")
    @RequestMapping("/categorys")
    public List<Category> list() {
        return service.getCategoryList(getUserId());
    }
}
```

Swagger2相关注解说明：

```
@Api：用在请求的类上，表示对类的说明
    tags="说明该类的作用，可以在UI界面上看到的注解"
    value="该参数没什么意义，在UI界面上也看到，所以不需要配置"

@ApiOperation：用在请求的方法上，说明方法的用途、作用
    value="说明方法的用途、作用"
    notes="方法的备注说明"

@ApiImplicitParams：用在请求的方法上，表示一组参数说明
    @ApiImplicitParam：用在@ApiImplicitParams注解中，指定一个请求参数的各个方面
        name：参数名
        value：参数的汉字说明、解释
        required：参数是否必须传
        paramType：参数放在哪个地方
            · header --> 请求参数的获取：@RequestHeader
            · query --> 请求参数的获取：@RequestParam
            · path（用于restful接口）--> 请求参数的获取：@PathVariable
            · body（不常用）
            · form（不常用）    
        dataType：参数类型，默认String，其它值dataType="Integer"       
        defaultValue：参数的默认值

@ApiResponses：用在请求的方法上，表示一组响应
    @ApiResponse：用在@ApiResponses中，一般用于表达一个错误的响应信息
        code：数字，例如400
        message：信息，例如"请求参数没填好"
        response：抛出异常的类

@ApiModel：用于响应类上，表示一个返回响应数据的信息
            （这种一般用在post创建的时候，使用@RequestBody这样的场景，
            请求参数无法使用@ApiImplicitParam注解进行描述的时候）
    @ApiModelProperty：用在属性上，描述响应类的属性
```

#### 4、测试

访问：`http://localhost:8080/swagger-ui.html`

![img](../../../../img/in-post/post-spring-boot/swagger01.png)

如果访问出现404，则需要修改拦截器配置

```java
@Configuration
public class MyWebMvcConfig extends WebMvcConfigurationSupport {
    @Override
    protected void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new MyHandlerInterceptor());
    }

    @Override
    protected void addResourceHandlers(ResourceHandlerRegistry registry) {
        // 修复静态资源访问404的问题
        registry.addResourceHandler("/**").addResourceLocations("classpath:/static/");
        registry.addResourceHandler("swagger-ui.html").addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**").addResourceLocations("classpath:/META-INF/resources/webjars/");
    }
}
```

## 参考

[Swagger2离线文档：PDF和Html5格式](https://blog.csdn.net/fly910905/article/details/79131755)

[Spring Boot中使用Swagger2构建强大的RESTful API文档](http://blog.didispace.com/springbootswagger2/)

[SpringBoot配置SwaggerUI访问404错误](https://www.cnblogs.com/moncat/p/7218061.html)