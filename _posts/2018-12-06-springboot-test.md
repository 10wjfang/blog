---
layout: post
title: Spring Boot（九）：Spring Boot测试打包部署
date: 2018-12-6 16:12:47
catalog: true
tags:
    - Spring Boot
---

## 单元测试

1、添加`spring-boot-starter-test`依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<scope>test</scope>
</dependency>
```

2、开发测试类

在测试类的类头部需要添加：`@RunWith(SpringRunner.class)`和`@SpringBootTest`注解，在测试方法的顶端添加@Test即可，最后在方法上点击右键run就可以运行。

3、对Controller层进行测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class CompanyControllerTest {
    private MockMvc mvc;
    @Autowired
    private WebApplicationContext context;

    @Before
    public void before() throws Exception {
        // 不要用standaloneSetup，会报NPE
        mvc = MockMvcBuilders.webAppContextSetup(context).build();
    }

    @After
    public void after() throws Exception {
    }

    /**
     * Method: listCompany(@RequestBody CompanyQuery query)
     */
    @Test
    public void testListCompany() throws Exception {
        CompanyQuery query = new CompanyQuery();
        query.setKey("公司");
        query.setType("name");
        String json = JSONObject.toJSONString(query);
        System.out.println("json = " + json);
        mvc.perform(MockMvcRequestBuilders.post("/GetCompanyList").contentType(MediaType.APPLICATION_JSON_UTF8).content(json))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andDo(MockMvcResultHandlers.print())
                .andReturn();
    }
} 
```

## 打包

1、打包成jar包

```sh
cd 项目跟目录（和pom.xml同级）
mvn clean package
## 或者执行下面的命令
## 排除测试代码后进行打包
mvn clean package  -Dmaven.test.skip=true
```

2、打包成war包

```xml
<packaging>war</packaging>
```

排除Tomcat：

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-tomcat</artifactId>
	<scope>provided</scope>
</dependency>
```

创建ServletInitializer.java，继承SpringBootServletInitializer ，覆盖configure()，把启动类Application注册进去。外部web应用服务器构建Web Application Context的时候，会把启动类添加进去。

```java
public class ServletInitializer extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(Application.class);
    }
}
```

## 部署运行

1、启动jar

```sh
nohup java -jar target/spring-boot-1.0.0.jar &
```

2、部署war包

拷贝到tomcat服务器中启动即可。

3、注册成服务

init.d 例子:

```sh
ln -s /var/yourapp/yourapp.jar /etc/init.d/yourapp
chmod +x /etc/init.d/yourapp
```
这样就可以使用stop或者是restart命令去管理你的应用。

```sh
/etc/init.d/yourapp start|stop|restart
```
或者

```sh
service yourapp start|stop|restart
```

4、docker部署


## 参考

[springboot(十二)：springboot如何测试打包部署](http://www.ityouknow.com/springboot/2017/05/09/springboot-deploy.html)