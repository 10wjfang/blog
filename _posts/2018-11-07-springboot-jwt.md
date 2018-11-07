---
layout: post
title: Spring Boot（六）：JWT实现用户认证
date: 2018-11-7 16:39:22
catalog: true
tags:
    - Spring Boot
    - JWT
---

## 前言

随着技术的发展，分布式Web应用的普及，通过session管理用户登录状态成本越来越高，因此慢慢发展成为token的方式做身份校验。

## 传统的Session认证

在服务端生成用户相关的 session 数据，而发给客户端 sesssion_id 存放到 cookie 中，这样用客户端请求时带上 session_id 就可以验证服务器端是否存在 session 数据，以此完成用户认证。这种认证方式，可以更好的在服务端对会话进行控制，安全性比较高(session_id 随机），但是服务端需要存储 session 数据(如内存或数据库），这样无疑增加维护成本和减弱可扩展性(多台服务器)。 CSRF 攻击一般基于 cookie 。另外，如果是原生 app 使用这种服务接口，又因为没有浏览器 cookie 功能，所以接入会相对麻烦。

## 基于token的鉴权机制

基于 token 的用户认证是一种服务端无状态的认证方式，服务端不用存放 token 数据。用户验证后,服务端生成一个 token(hash 或 encrypt)发给客户端，客户端可以放到 cookie 或 localStorage 中，每次请求时在 Header 中带上 token ，服务端收到 token 通过验证后即可确认用户身份。这种方式相对 cookie 的认证方式就简单一些，服务端不用存储认证数据，易维护扩展性强， token 存在 localStorage 可避免 CSRF ， web 和 app 应用这用接口都比较简单。

## JWT

#### 什么是JWT

JWT，全称是Json Web Token，是一个开放式标准（RFC 7519），它定义了一种紧凑且自包含的方式，用于在各方之间以JSON对象安全传输信息，这些信息可以使用HMAC算法或RSA的公钥/私钥对进行签名。

#### JWT的构成

JWT实际上就是一个字符串，它有三部分构成：头部、载荷和签名。
例如：`eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NDE1ODAxNzUsInVzZXJuYW1lIjoiYWRtaW4ifQ.komSDoymxjUjWrjz9nqkQd8Nqo92E_XTwFjQ0GX3pc4`

**头部（header）**

描述关于该JWT的最基本的信息，例如其类型以及签名所用的算法等。
就像这样：
```json
{
    "typ": "JWT",
    "alg": "HS256"
}
```

然后将头部进行base64编码，可以通过反编码得到原来的JSON。

**载荷（Payload）**

载荷是存放有效信息的地方，Claim是一个JSON，Claim中存放的内容是JWT自身的标准属性，也可以存放一些自定义的属性，比如用户ID，为了安全起见，一般不会将用户名及密码等敏感信息存放在Claim中，将Claim通过Base64转码后生成Payload。

**签名（Signature）**

Signature是由Header和Payload组合而成，然后使用定义好的加密算法和一个密钥对这个字符串进行加密，形成一个新的字符串。

## 快速上手

**1. 添加依赖**

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.1.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.shiro</groupId>
        <artifactId>shiro-spring</artifactId>
        <version>1.4.0</version>
    </dependency>
    <dependency>
        <groupId>com.auth0</groupId>
        <artifactId>java-jwt</artifactId>
        <version>3.3.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

主要是`java-jwt`依赖。

**2. 编写JWT加密校验工具**

使用用户自己的密码充当加密密钥，并在token中附带了username信息，设置密钥5分钟过期。

```java
public class JWTUtil {
    /**
     * 校验token是否正确
     * @param token 密钥
     * @param username 用户名
     * @param secret 密码
     * @return 是否正确
     */
    public static boolean verify(String token, String username, String secret) {
        try {
            Algorithm algorithm = Algorithm.HMAC256(secret);
            JWTVerifier verifier = JWT.require(algorithm)
                    .withClaim("username", username)
                    .build();
            DecodedJWT jwt = verifier.verify(token);
            return true;
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * 获取token中的用户名
     * @param token 密钥
     * @return 用户名
     */
    public static String getUsername(String token) {
        try {
            DecodedJWT jwt = JWT.decode(token);
            return jwt.getClaim("username").asString();
        }
        catch (JWTDecodeException e) {
            e.printStackTrace();;
            return null;
        }
    }

    /**
     * 生成签名
     * @param username 用户名
     * @param secret 密码
     * @return 加密的token
     */
    public static String sign(String username, String secret) {
        Date date = new Date(System.currentTimeMillis() + 5*60*1000);//5分钟过期
        try {
            Algorithm algorithm = Algorithm.HMAC256(secret);
            return JWT.create()
                    .withClaim("username", username)
                    .withExpiresAt(date)
                    .sign(algorithm);
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

**3. 编写Controller**

创建登录接口，返回带有token的JSON对象；index接口用来测试。

```java
@RestController
public class WebController {
    @Autowired
    private UserInfoService userInfoService;

    @RequestMapping("/login")
    public ResponseBean login(@RequestParam("username") String username, @RequestParam("password") String password) {
        UserInfo userInfo = userInfoService.findByUsername(username);
        if (userInfo.getPassword().equals(password)) {
            return new ResponseBean(200, "success", JWTUtil.sign(username, password));
        }
        else {
            return new ResponseBean(40, "fail", null);
        }
    }

    @RequestMapping("/index")
    public ResponseBean index() {
        Subject subject = SecurityUtils.getSubject();
        if (subject.isAuthenticated()) {
            return new ResponseBean(200, "login", null);
        }
        else {
            return new ResponseBean(200, "guest",null);
        }
    }
}
```

**4. 实现Realm**

用于处理用户是否合法。

```java
public class MyRealm extends AuthorizingRealm {
    @Resource
    private UserInfoService userInfoService;

    @Override
    public boolean supports(AuthenticationToken token) {
        return token instanceof JWTToken;
    }

    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        SimpleAuthorizationInfo authorizationInfo = new SimpleAuthorizationInfo();
        String username = (String)principalCollection.toString();
        UserInfo userInfo = userInfoService.findByUsername(username);
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

    /**
     * 认证
     * @param authenticationToken
     * @return
     * @throws AuthenticationException
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
        String token = (String) authenticationToken.getCredentials();
        String username = JWTUtil.getUsername(token);
        if (username == null) {
            throw  new AuthenticationException("token invalid");
        }
        UserInfo userInfo = userInfoService.findByUsername(username);
        if (userInfo == null) {
            throw new AuthenticationException("user not exist");
        }
        if (!JWTUtil.verify(token, username, userInfo.getPassword())) {
            throw new AuthenticationException("username or password error");
        }
        return new SimpleAuthenticationInfo(token, token, getName());
    }
}
```

> 注意：需要重写supports方法，不然会报错。

**5. 实现JWTToken**

JWTToken差不多就是Shiro用户名和密码的载体。

```java
public class JWTToken implements AuthenticationToken {
    private String token;

    public JWTToken(String token) {
        this.token = token;
    }

    @Override
    public Object getPrincipal() {
        return token;
    }

    @Override
    public Object getCredentials() {
        return token;
    }
}
```

**6. 重写Filter**

所有的请求都会先经过Filter，重写鉴权的方法。
代码的执行流程preHander -> isAccessAllowed -> isLoginAttempt -> executeLogin

```java
public class JWTFilter extends BasicHttpAuthenticationFilter {
    @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        if (isLoginAttempt(request, response)) {
            try {
                executeLogin(request, response);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return true;//返回true，通过subject.isAuthenticated判断
    }

    /**
     * 检测header里面是否含有Authorization字段
     */
    @Override
    protected boolean isLoginAttempt(ServletRequest request, ServletResponse response) {
        HttpServletRequest req = (HttpServletRequest) request;
        String authorization = req.getHeader("Authorization");
        return authorization != null;
    }

    @Override
    protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        String auth = httpServletRequest.getHeader("Authorization");
        JWTToken token = new JWTToken(auth);
        getSubject(request, response).login(token);//提交给realm处理
        return true;
    }
}
```

只要使用了Shiro，不管是否登录，核心过滤器都会为我们构造Subject实例，当我们主动调用subject.login方法时，会间接调用我们自己实现的realm的doGetAuthenticationInfo，根据我们在数据库中获取的信息（存放在info中）和调用login方法时传递的AuthenticationToken中的信息对比。

**7. 配置Shiro**

```java
@Configuration
public class ShiroConfig {
    @Bean
    public ShiroFilterFactoryBean shiroFilter(SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager);
        // 添加自己的过滤器
        Map<String, Filter> filterMap = new HashMap<>();
        filterMap.put("jwt", new JWTFilter());
        shiroFilterFactoryBean.setFilters(filterMap);
        Map<String, String> chainMap = new HashMap<>();
        chainMap.put("/**", "jwt");
        chainMap.put("/login", "anon");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(chainMap);
        return shiroFilterFactoryBean;
    }

//    @Bean
//    public HashedCredentialsMatcher hashedCredentialsMatcher(){
//        HashedCredentialsMatcher hashedCredentialsMatcher = new HashedCredentialsMatcher();
//        hashedCredentialsMatcher.setHashAlgorithmName("md5");//散列算法:这里使用MD5算法;
//        hashedCredentialsMatcher.setHashIterations(2);//散列的次数，比如散列两次，相当于 md5(md5(""));
//        return hashedCredentialsMatcher;
//    }

    @Bean
    public MyRealm myShiroRealm(){
        MyRealm myShiroRealm = new MyRealm();
        //myShiroRealm.setCredentialsMatcher(hashedCredentialsMatcher());
        return myShiroRealm;
    }

    @Bean
    public SecurityManager securityManager(){
        DefaultWebSecurityManager securityManager =  new DefaultWebSecurityManager();
        securityManager.setRealm(myShiroRealm());
        DefaultSubjectDAO subjectDAO = new DefaultSubjectDAO();
        DefaultSessionStorageEvaluator defaultSessionStorageEvaluator = new DefaultSessionStorageEvaluator();
        defaultSessionStorageEvaluator.setSessionStorageEnabled(false);//关闭shiro自带的session
        subjectDAO.setSessionStorageEvaluator(defaultSessionStorageEvaluator);
        securityManager.setSubjectDAO(subjectDAO);
        return securityManager;
    }
//
//    @Bean
//    @DependsOn("lifecycleBeanPostProcessor")
//    public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
//        DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
//        defaultAdvisorAutoProxyCreator.setProxyTargetClass(true);
//        return defaultAdvisorAutoProxyCreator;
//    }
//
//    @Bean
//    public LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
//        return  new LifecycleBeanPostProcessor();
//    }
//
//    @Bean
//    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
//        AuthorizationAttributeSourceAdvisor advisor = new AuthorizationAttributeSourceAdvisor();
//        advisor.setSecurityManager(securityManager);
//        return advisor;
//    }
}
```

其它的内容和Shiro项目一样。

**8. 测试**

在浏览器访问：`http://localhost:8080/login?username=admin&password=d3c59d25033dbf980d29554025c23a75`

返回：

```json
{
"code": 200,
"msg": "success",
"data": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NDE1ODAxNzUsInVzZXJuYW1lIjoiYWRtaW4ifQ.komSDoymxjUjWrjz9nqkQd8Nqo92E_XTwFjQ0GX3pc4"
}
```

在Postman访问：`http://localhost:8080/index`

返回：

```json
{
    "code": 200,
    "msg": "guest",
    "data": null
}
```

添加头部Authorization：`eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NDE1ODAxNzUsInVzZXJuYW1lIjoiYWRtaW4ifQ.komSDoymxjUjWrjz9nqkQd8Nqo92E_XTwFjQ0GX3pc4`，发送请求

返回：

```json
{
    "code": 200,
    "msg": "login",
    "data": null
}
```

## 参考

[session和token实现用户认证的异同](https://blog.csdn.net/christwei/article/details/80016162)

[Shiro+JWT+Spring Boot Restfule简易教程](https://github.com/Smith-Cruise/Spring-Boot-Shiro)