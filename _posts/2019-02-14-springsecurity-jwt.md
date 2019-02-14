---
layout: post
title: Spring Security（四）：集成JWT
date: 2019-2-14 10:32:43
catalog: true
tags:
    - Spring Security
    - JWT
---

## 前言

目前大多数都是前后端分离系统等服务器端无状态的应用。Basic Auth是配合RESTful API 使用的最简单的认证方式，只需提供用户名密码即可，但由于有把用户名密码暴露给第三方客户端的风险，在生产环境下被使用的越来越少。因此，在开发对外开放的RESTful API时，尽量避免采用Basic Auth。一般Basic验证适用于开发阶段。这里使用Token认证方式。


## 开始之前

准备Spring Security环境：

1、添加Spring Security依赖包

2、继承`WebSecurityConfigurerAdapter`类

## 知识准备

#### BasicAuthenticationFilter

`BasicAuthenticationFilter`负责处理`HTTPHeader`中的基本认证信息。继承`OncePerRequestFilter`类。
工作原理：在header中获取特定key和特定形式的value，获取得到，即使用当前过滤器进行验证身份信息。获取不到，则继续执行正常的过滤链。

在使用无状态认证时，需要关闭CSRF。
```java
http.csrf().disable()
```

- doFilterInternal：获取头部`Authorization`，处理`Basic [Token]`认证方式，

#### UsernamePasswordAuthenticationFilter

继承`AbstractAuthenticationProcessingFilter`类，`AbstractAuthenticationProcessingFilter`继承`GenericFilterBean`类。该过滤器会拦截用户请求，看它是否是一个来自用户名/密码表单登录页面提交的用户登录认证请求，缺省使用的匹配模式是:`POST /login`。

`UsernamePasswordAuthenticationFilter`是默认的登陆校验Filter，实现顺序是 
`AbstractAuthenticationProcessingFilter.doFilter->UsernamePasswordAuthenticationFilter.attemptAuthentication->ProviderManager.authenticate->AbstractUserDetailsAuthenticationProvider.authenticate->DaoAuthenticationProvider.retrieveUser->自定义的UserDetailsService.loadUserByUsername`
这里在自定义的`UserDetailsService`里按username取出user，security会去给你判断密码是否相等。

- attemptAuthentication：获取用户名和密码，然后进行认证。
- successfulAuthentication：认证成功后处理业务逻辑。

#### OncePerRequestFilter

继承`GenericFilterBean`类，一次请求仅仅经过一个的filter，而不需要重复执行。

> 在servlet-2.3中，Filter会过滤一切请求，包括服务器内部使用forward转发请求和<%@ include file=”/index.jsp”%>的情况。 
到了servlet-2.4中Filter默认下只拦截外部提交的请求，forward和include这些内部转发都不会被过滤，但是有时候我们需要 forward的时候也用到Filter。

## 登录实现

#### 使用缺省login

1、**处理登录**，创建类继承`UsernamePasswordAuthenticationFilter`，当验证用户名密码正确后，生成一个token，并将token返回给客户端。

```java
public class JWTLoginFilter extends UsernamePasswordAuthenticationFilter {
    public static final Logger log = LoggerFactory.getLogger(JWTLoginFilter.class);
    private UserAuthRepository userAuthRepository;

    public JWTLoginFilter(AuthenticationManager authenticationManager, UserAuthRepository userAuthRepository) {
        setAuthenticationManager(authenticationManager);
        this.userAuthRepository = userAuthRepository;
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) throws IOException, ServletException {
        log.info("登录成功！");
        TUserAuthEntity userAuthEntity = userAuthRepository.findByIdentifier(authResult.getPrincipal().toString()).orElse(null);
        String token = JwtUtil.sign(userAuthEntity.getIdentifier(), userAuthEntity.getCredential());
        LoginInfoVo loginInfoVo = new LoginInfoVo();
        BeanUtils.copyProperties(userAuthEntity, loginInfoVo);
        loginInfoVo.setToken(token);
        response.setCharacterEncoding("utf-8");
        response.setContentType("application/json; charset=utf-8");
        response.addHeader("Authorization", token);
        response.getWriter().print(JSONObject.toJSON(ResponseGenerator.success(loginInfoVo)));
    }
}
```

*默认情况下只接收username和password两个参数，如果想要传递其它参数，可以重写`attemptAuthentication`方法，接收并解析用户凭证。*

2、**授权验证**，创建类继承`BasicAuthenticationFilter`，用户一旦登录成功后，会拿到token，后续的请求都会带着这个token，服务端会验证token的合法性。

```java
public class JWTAuthenticationFilter extends BasicAuthenticationFilter {
    public static final Logger logger = LoggerFactory.getLogger(JWTAuthenticationFilter.class);
    private UserAuthRepository userAuthRepository;

    public JWTAuthenticationFilter(AuthenticationManager authenticationManager) {
        super(authenticationManager);
    }

    public JWTAuthenticationFilter(AuthenticationManager authenticationManager, UserAuthRepository userAuthRepository) {
        this(authenticationManager);
        this.userAuthRepository = userAuthRepository;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        String authentication = request.getHeader("Authorization");
        logger.info("Authorization: {}", authentication);
        if (authentication == null) {
            chain.doFilter(request, response);
            return;
        }
        String username = JwtUtil.getUsername(authentication);
        TUserAuthEntity userAuthEntity = userAuthRepository.findByIdentifier(username).orElse(null);
        if (userAuthEntity == null) {
            throw new BadCredentialsException("user not exist");
        }
        if (JwtUtil.verify(authentication, username, userAuthEntity.getCredential())) {
            UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(username, userAuthEntity.getCredential());
            Authentication authenticate = null;
            try {
                authenticate = getAuthenticationManager().authenticate(token);
                SecurityContextHolder.getContext().setAuthentication(authenticate);
            } catch (AuthenticationException e) {
                e.printStackTrace();
            }
        }
        chain.doFilter(request, response);
    }
}
```

3、**SpringSecurity配置**，通过SpringSecurity的配置，将上面的方法组合在一起。

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private AccessDeniedHandlerImpl accessDeniedHandler;
    @Autowired
    private RestAuthenticationEntryPoint authenticationEntryPoint;
    @Autowired
    private UserAuthRepository userAuthRepository;
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //禁用CSRF
        http.csrf().disable()
                //权限不足异常
                .exceptionHandling().accessDeniedHandler(accessDeniedHandler)
                //没有登录异常
                .authenticationEntryPoint(authenticationEntryPoint);
        http.authorizeRequests().antMatchers("/login").permitAll().anyRequest().authenticated().and()
                //禁用session
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.NEVER);
       http.addFilter(new JWTLoginFilter(authenticationManager(), userAuthRepository))
               .addFilter(new JWTAuthenticationFilter(authenticationManager(), userAuthRepository));
        // http.addFilter(new JWTAuthenticationFilter(authenticationManager(), userAuthRepository));
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        // 解决authenticationManager 无法注入
        return super.authenticationManagerBean();
    }
}
```

```java
.addFilter(new JWTLoginFilter(authenticationManager(), userAuthRepository))
.addFilter(new JwtAuthenticationFilter(authenticationManager())) 
```

这两行，将我们定义的JWT方法加入SpringSecurity的处理流程中。

#### 自定义login接口

1、**处理登录**，创建`UserLoginController`控制器，返回带有token的结果类。

```java
@RestController
public class UserLoginController {
    @Autowired
    private IUserAuthService userAuthService;
    @Autowired
    private AuthenticationManager authenticationManager;

    @RequestMapping(value = "/login")
    public ResponseDataDTO<LoginInfoVo> UserLogin(@RequestParam String username, @RequestParam String password) {
        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(username, password);
        Authentication authenticate = null;
        try {
            authenticate = authenticationManager.authenticate(token);
            SecurityContextHolder.getContext().setAuthentication(authenticate);
            TUserAuthEntity entity = userAuthService.getUserAuth(username);
            LoginInfoVo loginInfoVo = new LoginInfoVo();
            BeanUtils.copyProperties(entity, loginInfoVo);
            loginInfoVo.setToken(JwtUtil.sign(username, entity.getCredential()));
            return ResponseGenerator.success(loginInfoVo);
        } catch (AuthenticationException e) {
            e.printStackTrace();
        }
        return ResponseGenerator.errors(ErrorCodeEnum.USERNAME_PASSWORD_ERROR);
    }
}
```

2、授权验证和第一种方式一样，配置`SecurityConfig`类

```java
http.addFilter(new JWTAuthenticationFilter(authenticationManager(), userAuthRepository));
```

只需加入一行即可。

## 测试

1、测试接口

```sh
curl http://localhost:8899/test
```

返回请登录提示信息。

2、登录

```sh
curl -i -H"Content-Type: application/json" -X POST -d '{"username":"admin","password":"password"}' http://localhost:8080/login
```

返回带有token的信息。

3、用登录成功后拿到的token再次请求接口

```sh
curl -H"Authorization:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE1NTAxMjUxMzIsInVzZXJuYW1lIjoiMTExIn0.f-R89ILSCOVS6WBgJ-l7G_TKMSejjDKKGGZ9tBKgFAk" http://localhost:8899/test
```

返回hello。

## 参考

[Spring Boot+Spring Security+JWT 实现 RESTful Api 权限控制](https://blog.csdn.net/christwei/article/details/80016162)

[Spring Boot整合Spring Security简记-Basic认证（五）](https://www.jianshu.com/p/6ec845c17492)
