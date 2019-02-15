---
layout: post
title: Spring Security（五）：使用Redis
date: 2019-2-15 21:11:43
catalog: true
tags:
    - Spring Security
    - Redis
---

## Redis介绍

Redis是目前业界使用最广泛的内存数据存储。相比memcached，Redis支持更丰富的数据结构，例如hashes, lists, sets等，同时支持数据持久化。除此之外，Redis还提供一些类数据库的特性，比如事务，HA，主从库。可以说Redis兼具了缓存系统和数据库的一些特性，因此有着丰富的应用场景。

## 引入Redis

Redis包有两个版本：
- `spring-boot-starter-redis`：对应Spring Boot 1.*的版本
- `spring-boot-starter-data-redis`：对应Spring Boot 2.*的版本

## Spring的支持

根据Redis的不同的Java客户端，Spring Data Redis提供了如下的ConnectionFactory： 
- `JedisConnectionFactory`：使用Jedis作为Redis客户端。 
- `JredisConnectionFactory`：使用Jredis作为Redis客户端。 
- `LettuceConnectionFactory`：使用Lettuce作为Redis客户端。 
- `SrpConnectionFactory`：使用Spullara/redis-protocol作为Redis客户端。 

spring boot框架中已经集成了redis，在1.x.x的版本时默认使用的jedis客户端，现在是2.x.x版本默认使用的lettuce客户端，两种客户端的区别如下：

Jedis和Lettuce都是Redis Client。
- Jedis 是直连模式，在多个线程间共享一个 Jedis 实例时是**线程不安全**的，
- 如果想要在多线程环境下使用 Jedis，需要使用连接池，每个线程都去拿自己的 Jedis 实例，当连接数量增多时，物理连接成本就较高了。
- Lettuce的连接是基于Netty的，连接实例可以在多个线程间共享，所以，一个多线程的应用可以使用同一个连接实例，而不用担心并发线程的数量。当然这个也是可伸缩的设计，一个连接实例不够的情况也可以按需增加连接实例。通过异步的方式可以让我们更好的利用系统资源，而不用浪费线程等待网络或磁盘I/O。

## 如何使用

1、引入spring-boot-starter-data-redis

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

2、添加配置文件

```properties
spring.redis.host=192.168.241.132
spring.redis.port=6379
# 连接池最大连接数（使用负值表示没有限制）
spring.redis.pool.max-active=8  
# 连接池最大阻塞等待时间（使用负值表示没有限制）
spring.redis.pool.max-wait=-1  
# 连接池中的最大空闲连接
spring.redis.pool.max-idle=8  
# 连接池中的最小空闲连接
spring.redis.pool.min-idle=0  
# 连接超时时间（毫秒）
spring.redis.timeout=0
```

3、添加Redis配置类

```java
@Configuration
@ConfigurationProperties(prefix = "spring.redis")
public class RedisConfig {
    private String host;
    private Integer port;

    public String getHost() {
        return host;
    }

    public void setHost(String host) {
        this.host = host;
    }

    public Integer getPort() {
        return port;
    }

    public void setPort(Integer port) {
        this.port = port;
    }

    /**
     * 获取LettuceConnectionFactory工厂
     * @return RedisConnectionFactory
     */
    @Bean
    public RedisConnectionFactory getRedisConnectionFactory() {
        return new LettuceConnectionFactory(new RedisStandaloneConfiguration(host, port));
    }

    @Bean
    public RedisTemplate<String, Serializable> getRedisTemplate() {
        RedisTemplate<String, Serializable> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(getRedisConnectionFactory());
        setMySerializer(redisTemplate);
        return redisTemplate;
    }

    /**
     * 设置序列化方法
     * @param redisTemplate
     */
    private void setMySerializer(RedisTemplate<String, Serializable> redisTemplate) {
        JdkSerializationRedisSerializer redisSerializer = new JdkSerializationRedisSerializer();
        redisTemplate.setKeySerializer(redisTemplate.getStringSerializer());
        redisTemplate.setValueSerializer(redisSerializer);
    }
}
```

- `@Configuration` 注解是用于定义配置类
- `@ConfigurationProperties` 注解是用于读取配置文件的信息

spring-data-redis中序列化类有以下几个：

- `GenericToStringSerializer`：可以将任何对象泛化为字符创并序列化
- `Jackson2JsonRedisSerializer`：序列化Object对象为json字符创（与JacksonJsonRedisSerializer相同）,被序列化对象不需要实现Serializable接口，被序列化的结果清晰，容易阅读，而且存储字节少，速度快。
- `JdkSerializationRedisSerializer`：序列化java对象，被序列化对象必须实现Serializable接口，被序列化除属性内容还有其他内容，长度长且不易阅读。
- `StringRedisSerializer`：简单的字符串序列化，一般如果key、value都是string字符串的话，就是用这个就可以了。

4、添加登录接口

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
            redisTemplate.opsForHash().put(token, Constants.KEY_USER_INFO, entity);
            redisTemplate.expire(token, 60, TimeUnit.MINUTES);
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

- `UUID.randomUUID().toString()`是JDK提供的一个自动生成主键的方法。UUID(Universally Unique Identifier)全局唯一标识符。
- `redisTemplate.opsForHash().put(token, Constants.KEY_USER_INFO, entity)`：登录成功后将token存到Redis里。

5、授权验证

在上一篇的基础上加上`TokenFilter`类，用来过滤权限请求。

```java
public class TokenFilter extends BasicAuthenticationFilter {
    public static final Logger logger = LoggerFactory.getLogger(TokenFilter.class);

    private RedisTemplate<String, Serializable> redisTemplate;

    public TokenFilter(AuthenticationManager authenticationManager, RedisTemplate redisTemplate) {
        super(authenticationManager);
        this.redisTemplate = redisTemplate;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        String authentication = request.getHeader(Constants.HEADER_TOKEN);
        logger.debug("Token: {}", authentication);
        TUserAuthEntity user = (TUserAuthEntity) redisTemplate.opsForHash().get(authentication, Constants.KEY_USER_INFO);
        if (user == null) {
            throw new InvalidTokenException();
        }
        UsernamePasswordAuthenticationToken auth = new UsernamePasswordAuthenticationToken(user.getIdentifier(), user.getCredential());
        SecurityContextHolder.getContext().setAuthentication(getAuthenticationManager().authenticate(auth));
        chain.doFilter(request, response);
    }
}
```

6、修改SecurityConfig

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
    private RedisTemplate<String, Serializable> redisTemplate;
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
        http.addFilter(new TokenFilter(authenticationManager(), redisTemplate));
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        // 解决authenticationManager 无法注入
        return super.authenticationManagerBean();
    }
}
```

## 测试

1、访问test接口，返回需要登录信息。

2、访问登录接口，返回带token信息。

3、使用token访问test接口，返回正确信息。

[示例代码](https://github.com/10wjfang/spring-boot-demo/tree/master/springboot-security-redis)

## 参考

[springboot(三)：Spring boot中Redis的使用](http://www.ityouknow.com/springboot/2016/03/06/spring-boot-redis.html)

[SpringBoot之Redis的支持](https://blog.csdn.net/smartdt/article/details/78894013)

[关于Spring Data redis几种对象序列化的比较](https://www.cnblogs.com/JKayFeng/p/5911549.html)

[Spring Security 学习笔记-securityContext过滤器](https://www.cnblogs.com/mingluosunshan/p/5329037.html)

[springSecurity工作流程学习](https://blog.csdn.net/qq_15042899/article/details/73467049)

[Spring-Security 运行流程](https://blog.csdn.net/u012156496/article/details/53486723)

[spring boot 集成 redis lettuce](https://www.cnblogs.com/taiyonghai/p/9454764.html)