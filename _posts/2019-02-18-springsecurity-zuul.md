---
layout: post
title: Spring Security（六）：集成Zuul搭建全局验证架构
date: 2019-2-18 17:56:34
catalog: true
tags:
    - Zuul
    - Spring Security
---

## 总体架构

![img](../../../../img/in-post/post-spring-security/4.jpg)

## 验证中心

用于处理登录请求，生成Token，并存储到Redis。

主要修改`login`接口：

```java
@RestController
public class UserLoginController {
    @Autowired
    private IUserAuthService userAuthService;
    @Autowired
    private AuthenticationManager authenticationManager;
    @Autowired
    private RedisTemplate<String, Serializable> redisTemplate;

    @RequestMapping(value = "/login")
    public ResponseDataDTO<LoginInfoVo> UserLogin(@RequestParam String username, @RequestParam String password) {
        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(username, password);
        Authentication authenticate = null;
        try {
            String token = UUID.randomUUID().toString();
            authenticate = authenticationManager.authenticate(authenticationToken);
            SecurityContextHolder.getContext().setAuthentication(authenticate);
            TUserAuthEntity entity = userAuthService.getUserAuth(username);
            redisTemplate.opsForHash().put(token, Constants.KEY_USER_INFO, authenticationToken);
            redisTemplate.opsForHash().put(token, Constants.KEY_SPRING_SECURITY_CONTEXT, SecurityContextHolder.getContext());
            redisTemplate.expire(token, Constants.TOKEN_EXPIRE, TimeUnit.MINUTES);
            LoginInfoVo loginInfoVo = new LoginInfoVo();
            BeanUtils.copyProperties(entity, loginInfoVo);
            loginInfoVo.setToken(token);
            return ResponseGenerator.success(loginInfoVo);
        } catch (AuthenticationException e) {
            e.printStackTrace();
        }
        return ResponseGenerator.errors(ErrorCodeEnum.USERNAME_PASSWORD_ERROR);
    }
}
```

增加了Context信息保存。

## 网关服务

- 定义验证中心、后台服务的路由
- 验证Token令牌
- 基于角色的权限校验

增加`RedisSecurityContextRepository`类，实现`SecurityContextRepository`接口：

```java
/**
 * 把security原来token放在HttpSession(session)里的方式重新自定义为放到Redis里
 *
 * @author fwj
 * @date 2019-02-18 14:27
 **/
public class RedisSecurityContextRepository implements SecurityContextRepository {
    private final Logger logger = LoggerFactory.getLogger(RedisSecurityContextRepository.class);
    private final RedisTemplate<String, Serializable> redisTemplate;
    private boolean isServlet3 = ClassUtils.hasMethod(ServletRequest.class, "startAsync");
    private AuthenticationTrustResolver trustResolver = new AuthenticationTrustResolverImpl();
    private boolean disableUrlRewriting = false;

    public RedisSecurityContextRepository(final RedisTemplate<String, Serializable> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    @Override
    public SecurityContext loadContext(HttpRequestResponseHolder requestResponseHolder) {
        HttpServletRequest request = requestResponseHolder.getRequest();
        logger.debug("url: {}", request.getRequestURI());
        HttpServletResponse response = requestResponseHolder.getResponse();
        String token = extractToken(request);
        SecurityContext context = this.readSecurityContextFromRedis(token);
        if (context == null) {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("No SecurityContext was available from the Redis: " + token + ". "
                        + "A new one will be created.");
            }

            context = this.generateNewContext();
        }

        SaveToRedisResponseWrapper wrappedResponse = new SaveToRedisResponseWrapper(response, request, context);
        requestResponseHolder.setResponse(wrappedResponse);
        if (this.isServlet3) {
            requestResponseHolder.setRequest(new Servlet3SaveToRedisRequestWrapper(request, wrappedResponse));
        }

        return context;
    }

    private SecurityContext generateNewContext() {
        return SecurityContextHolder.createEmptyContext();
    }

    private SecurityContext readSecurityContextFromRedis(String token) {
        boolean debug = this.logger.isDebugEnabled();
        if (token == null) {
            if (debug) {
                this.logger.debug("No Token currently exists");
            }

            return null;
        } else {
            Object contextFromRedis = loadContextFromRedis(token);
            if (contextFromRedis == null) {
                if (debug) {
                    this.logger.debug("Token returned null object for SPRING_SECURITY_CONTEXT");
                }

                return null;
            } else if (!(contextFromRedis instanceof SecurityContext)) {
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn(Constants.KEY_SPRING_SECURITY_CONTEXT
                            + " did not contain a SecurityContext but contained: '" + contextFromRedis
                            + "'; are you improperly modifying the HttpSession directly "
                            + "(you should always use SecurityContextHolder) or using the HttpSession attribute "
                            + "reserved for this class?");
                }

                return null;
            } else {
                if (debug) {
                    this.logger.debug("Obtained a valid SecurityContext from "
                            + Constants.KEY_SPRING_SECURITY_CONTEXT + ": '" + contextFromRedis + "'");
                }
                redisTemplate.expire(token, Constants.TOKEN_EXPIRE, TimeUnit.MINUTES);
                return (SecurityContext)contextFromRedis;
            }
        }
    }

    @Override
    public void saveContext(SecurityContext context, HttpServletRequest request, HttpServletResponse response) {
        SaveToRedisResponseWrapper responseWrapper = WebUtils.getNativeResponse(response,
                SaveToRedisResponseWrapper.class);
        if (responseWrapper == null) {
            throw new IllegalStateException("Cannot invoke saveContext on response " + response
                    + ". You must use the HttpRequestResponseHolder.response after invoking loadContext");
        } else {
            if (!responseWrapper.isContextSaved()) {
                responseWrapper.saveContext(context);
            }

        }
    }

    @Override
    public boolean containsContext(HttpServletRequest request) {
        String token = extractToken(request);
        if (token == null) {
            return false;
        }
        return loadContextFromRedis(token) != null;
    }

    final class SaveToRedisResponseWrapper extends SaveContextOnUpdateOrErrorResponseWrapper {
        private final HttpServletRequest request;
        private final SecurityContext contextBeforeExecution;
        private final Authentication authBeforeExecution;

        public SaveToRedisResponseWrapper(HttpServletResponse response, HttpServletRequest request,
                                          SecurityContext context) {
            super(response, disableUrlRewriting);
            this.request = request;
            this.contextBeforeExecution = context;
            this.authBeforeExecution = context.getAuthentication();
        }

        @Override
        protected void saveContext(SecurityContext context) {
            Authentication authentication = context.getAuthentication();
            String token = extractToken(request);
            if (authentication == null || trustResolver.isAnonymous(authentication)) {
                logger.debug("Security is empty or content is anonymous.");
                return;
            }
            if (token != null) {
                if (contextChanged(context) || loadContextFromRedis(token) == null) {
                    saveContextToRedis(token, context);
                }
            }
        }

        private boolean contextChanged(SecurityContext context) {
            return context != contextBeforeExecution || context.getAuthentication() != authBeforeExecution;
        }
    }

    private void saveContextToRedis(String token, SecurityContext context) {
        redisTemplate.opsForHash().put(token, Constants.KEY_SPRING_SECURITY_CONTEXT, context);
        redisTemplate.expire(token, Constants.TOKEN_EXPIRE, TimeUnit.MINUTES);
    }

    private Object loadContextFromRedis(String token) {
        return redisTemplate.opsForHash().get(token, Constants.KEY_SPRING_SECURITY_CONTEXT);
    }

    /**
     * 获取Token
     * @param request
     * @return
     */
    private String extractToken(HttpServletRequest request) {
        String token = request.getHeader(Constants.HEADER_TOKEN);
        if (token == null) {
            logger.debug("Token not found in headers. Trying request parameters.");
            token = request.getParameter(Constants.HEADER_TOKEN);
            if (token == null) {
                logger.debug("Token not found in parameters");
            }
        }
        return token;
    }

    private class Servlet3SaveToRedisRequestWrapper extends HttpServletRequestWrapper {
        private final SaveContextOnUpdateOrErrorResponseWrapper response;
        public Servlet3SaveToRedisRequestWrapper(HttpServletRequest request,
                                                 SaveContextOnUpdateOrErrorResponseWrapper wrappedResponse) {
            super(request);
            this.response = wrappedResponse;
        }

        @Override
        public AsyncContext startAsync() {
            this.response.disableSaveOnResponseCommitted();
            return super.startAsync();
        }

        @Override
        public AsyncContext startAsync(ServletRequest servletRequest, ServletResponse servletResponse)
                throws IllegalStateException {
            this.response.disableSaveOnResponseCommitted();
            return super.startAsync(servletRequest, servletResponse);
        }
    }
}
```

修改`SecurityConfig`类，增加`http.securityContext().securityContextRepository(securityContextRepository());`：

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private RedisTemplate<String, Serializable> redisTemplate;
    @Value("${auth.permit.patterns}")
    private String[] permitPatterns;
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //禁用CSRF
        http.csrf().disable()
                //权限不足异常
                .exceptionHandling().accessDeniedHandler(new AccessDeniedHandlerImpl())
                //没有登录异常
                .authenticationEntryPoint(new RestAuthenticationEntryPoint());
        http.authorizeRequests().antMatchers(permitPatterns).permitAll().anyRequest().authenticated().and()
                //禁用session
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.NEVER);
        http.securityContext().securityContextRepository(securityContextRepository());
    }

    public SecurityContextRepository securityContextRepository() {
        return new RedisSecurityContextRepository(redisTemplate);
    }
}
```

#### RedisSecurityContextRepository

重点介绍一下`RedisSecurityContextRepository`类。

**1、SecurityContextPersistenceFilter**

该类在所有的Filter之前,是从SecurityContextRepository中取出用户认证信息,默认实现类为HttpSessionSecurityContextRepository,其会从Session中取出已认证用户的信息,提高效率,避免每一次请求都要查询用户认证信息.

取出之后会放入SecurityContextHolder中,以便其他filter使用,SecurityContextHolder使用ThreadLocal存储用户认证信息,保证了线程之间的信息隔离,最后再finally中清除该信息.可以配置http的security-context-repository属性来自己控制获取到已认证用户信息的方式,比如使用redis存储session等.

**2、RedisSecurityContextRepository**

这个实体在`spring security`启动中要传递给`SecurityContextPersistenceFilter`。这个filter根据request来加载`SecurityContext`。而`SecurityContextPersistenceFilter`就是从其内部的`SecurityContextRepositor`y来加载`SecurityContext`的。所以我们就需要重载上面代码中的三个方法，根据request来构造`SecurityContext`。

#### 配置文件

在`application.properties`增加以下内容：

```properties
auth.permit.patterns=/auth-server/**,/actuator/**
```

## 后台服务

提供业务服务，不需要做任何改动。

[示例代码：springcloud-auth]( https://github.com/10wjfang/spring-cloud-demo)

## 参考

[采用Zuul网关和Spring Security搭建一个基于JWT的全局验证架构](https://blog.csdn.net/daijinmingcn/article/details/79261610)

[Spring Security(二) -- Spring Security的Filter](https://zhuanlan.zhihu.com/p/27644391)