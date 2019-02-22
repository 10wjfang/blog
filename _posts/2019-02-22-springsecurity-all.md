---
layout: post
title: Spring Security（七）：完整的用户认证中心
date: 2019-2-22 15:58:51
catalog: true
tags:
    - Spring Security
---

## 总体介绍

该用户认证中心采用`Spring Security`框架搭建，结合`Redis`存储用户`Token`信息，实现用户登录、注册、登出接口，客户端密码传输使用`RSA`公钥加密，服务端接收到密码后使用私钥解密，本文主要介绍这3个接口的具体实现。

## 用户登录

#### 工作流程

1、使用私钥解密经过RSA加密的密码；

2、通过Spring Security进行身份认证；

3、产生token并保存到Redis；

4、获取登录返回信息返回给客户端。

#### 具体实现

1、新建登录参数请求类：

```java
@Data
public class LoginRequest implements Serializable {
    @NotBlank(message = "用户名不能为空")
    private String username;
    @NotBlank(message = "密码不能为空")
    private String password;
}
```

2、新建登录接口：

```java
public interface ILoginService {
    /**
     * 登录动作
     * @param model LoginRequest
     * @param request HttpServletRequest
     * @return LoginInfoVo
     */
    LoginInfoVo doLogin(LoginRequest model) throws Exception;
}
```

3、新建登录抽象类，实现ILoginService接口：

```java
public abstract class AbstractLoginService implements ILoginService {
    private static final Logger logger = LoggerFactory.getLogger(AbstractLoginService.class);
    @Override
    public LoginInfoVo doLogin(LoginRequest model) throws Exception {
        String decryptPassword = decrypt(model.getUsername(), model.getPassword());
        if (StringUtils.isNotBlank(decryptPassword)) {
            Authentication auth = authenticate(model.getUsername(), decryptPassword);
            SecurityContextHolder.getContext().setAuthentication(auth);
            if (auth != null || !auth.isAuthenticated()) {
                String token = createToken();
                saveTokenToRedis(token);
                LoginInfoVo loginInfoVo = getLoginInfo(model.getUsername());
                loginInfoVo.setToken(token);
                return loginInfoVo;
            }
        }
        return null;
    }

    /**
     * 解密密码
     * @param password 加密的密码
     * @param username redis的键
     * @return 解密的密码
     */
    public abstract String decrypt(String username, String password) throws Exception;

    /**
     * 获取登录信息
     * @param username 用户名
     * @return LoginInfoVo
     */
    protected abstract LoginInfoVo getLoginInfo(String username);

    /**
     * 保存token到Redis
     * @param token
     */
    protected abstract void saveTokenToRedis(String token);

    /**
     * 生成Token
     * @return
     */
    protected abstract String createToken();

    /**
     * 认证
     * @param username 用户名
     * @param password 密码
     * @return
     */
    protected abstract Authentication authenticate(String username, String password);
}
```

4、新建用户登录服务类，基础AbstractLoginService抽象类：

```java
@Service
public class UserPassLoginService extends AbstractLoginService {
    @Autowired
    private IUserAuthService userAuthService;
    @Autowired
    private IUserInfoService userInfoService;
    @Autowired
    private AuthenticationManager authenticationManager;
    @Autowired
    private RedisTemplate<String, Serializable> redisTemplate;
    @Autowired
    private IPasswordEncoder passwordEncoder;

    @Override
    public String decrypt(String username, String password) throws Exception {
        return passwordEncoder.decrypt(username, password);
    }

    @Override
    protected LoginInfoVo getLoginInfo(String username) {
        TUserAuthEntity entity = userAuthService.getUserAuth(username);
        TUserEntity userEntity = userInfoService.getUserInfo(entity.getUid());
        LoginInfoVo loginInfoVo = new LoginInfoVo();
        // ... 省略赋值操作
        userAuthService.save(entity);
        return loginInfoVo;
    }

    @Override
    protected void saveTokenToRedis(String token) {
        redisTemplate.opsForHash().put(token, Constants.KEY_SPRING_SECURITY_CONTEXT, SecurityContextHolder.getContext());
        redisTemplate.expire(token, Constants.TOKEN_EXPIRE, TimeUnit.MINUTES);
    }

    @Override
    protected String createToken() {
        return UUID.randomUUID().toString();
    }

    @Override
    protected Authentication authenticate(String username, String password) {
        UsernamePasswordAuthenticationToken auth = new UsernamePasswordAuthenticationToken(username, password);
        return authenticationManager.authenticate(auth);
    }
}
```

5、新建登录控制器：

```java
@RestController
public class UserAuthController {
    @Autowired
    private ILoginService loginService;
    @Autowired
    private IUserAuthService userAuthService;
    @Autowired
    private IUserInfoService userInfoService;
    @Autowired
    private IUserRoleService userRoleService;
    @Autowired
    private RedisTemplate<String, Serializable> redisTemplate;
    @Autowired
    private IPasswordEncoder passwordEncoder;

    @PostMapping("/login")
    public ResponseDTO userLogin(@RequestBody @Validated LoginRequest model) throws Exception {
        LoginInfoVo loginInfoVo = loginService.doLogin(model, request);
        if(loginInfoVo != null) {
            return ResponseGenerator.success(loginInfoVo);
        }
        return ResponseGenerator.error(ErrorCodeEnum.USERNAME_PASSWORD_ERROR);
    }

    // ... 省略其它方法
}
```

6、修改`UserAuthenticationProvider`类，增加密码校验：

```java
@Component
public class UserAuthenticationProvider implements AuthenticationProvider {
    private static final Logger logger = LoggerFactory.getLogger(UserAuthenticationProvider.class);
    @Autowired
    private CustomUserService customUserService;
    @Autowired
    private PasswordEncoder passwordEncoder;
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        logger.debug(JSONObject.toJSON(authentication).toString());
        UserDetails userDetails = customUserService.loadUserByUsername(authentication.getPrincipal().toString());
        if (userDetails == null) {
            throw new BadCredentialsException("Username not found!");
        }
        if (!passwordEncoder.matches(authentication.getCredentials().toString(), userDetails.getPassword())) {
            throw new BadCredentialsException("Password is wrong!");
        }
        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(userDetails.getUsername(),
                userDetails.getPassword(), userDetails.getAuthorities());
        token.setDetails(userDetails);
        return token;
    }

    @Override
    public boolean supports(Class<?> aClass) {
        return true;
    }
}
```

## 登出

#### 工作流程

1、从请求头部获取token

2、清除Spring Security上下文认证信息

3、从Redis删除对应token

#### 具体实现

```java
@GetMapping("/logout")
public ResponseDTO logout(HttpServletRequest request, HttpServletResponse response) throws Exception {
    String token = request.getHeader(Constants.HEADER_TOKEN);
    if (token == null) {
        return ResponseGenerator.error(ErrorCodeEnum.TOKEN_ISNULL);
    }
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    if (auth == null) {
        return ResponseGenerator.error(ErrorCodeEnum.NOT_LOGIN);
    }
    new SecurityContextLogoutHandler().logout(request, response, auth);
    redisTemplate.delete(token);
    return ResponseGenerator.success("成功退出登录");
}
```

## 注册

#### 工作流程

1、判断用户名是否存在，存在则返回，不存在则继续；

2、使用RSA私钥解密用户密码；

3、对原始密码进行加密，对实体对象进行赋值后保存到数据库。

#### 具体实现

```java
@PostMapping("/register")
@Transactional
public ResponseDTO userRegister(@RequestBody @Validated RegisterRequest model) throws Exception {
    if (userAuthService.isExists(model.getUserName())) {
        return ResponseGenerator.error(ErrorCodeEnum.USERNAME_EXIST);
    }
    String rawPassword = passwordEncoder.decrypt(model.getUserName(), model.getPassword());
    if (StringUtils.isBlank(rawPassword)) {
        return ResponseGenerator.error(ErrorCodeEnum.PARAM_ERROR);
    }
    TUserEntity userEntity = new TUserEntity();
    if (RegexUtils.checkMobile(model.getUserName())) {
        userEntity.setTelephone(model.getUserName());
    }
    // 注册送积分
    userEntity.setIntegration(500);
    // 默认0
    userEntity.setStatus((byte) 0);
    // 保存到用户表
    userInfoService.save(userEntity);
    TUserAuthEntity entity = new TUserAuthEntity();
    entity.setUid(userEntity.getUid());
    entity.setIdentifier(model.getUserName());
    entity.setCredential(passwordEncoder.encode(rawPassword));
    // 手机号注册
    if (RegexUtils.checkMobile(model.getUserName())) {
        entity.setIdentiyType((byte) RegistTypeEnum.Mobile.getOrdinal());
    }
    // 保存到用户认证表
    userAuthService.save(entity);
    RUserRoleEntity userRoleEntity = new RUserRoleEntity();
    userRoleEntity.setUid(userEntity.getUid());
    userRoleEntity.setRoleId(RoleTypeEnum.CommonUser.getOrdinal());
    // 保存到用户角色关系表
    userRoleService.save(userRoleEntity);
    return ResponseGenerator.success("注册成功");
}
```

## 单元测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserAuthControllerTest {
    private MockMvc mvc;
    @Autowired
    private WebApplicationContext context;
    final String username = "13729395540";
    final String password = "Pw123456";

    @Before
    public void setUp() throws Exception {
        mvc = MockMvcBuilders.webAppContextSetup(context).build();
    }

    @Test
    public void userLogin() throws Exception {
        doLogin();
    }

    private String doLogin() throws Exception {
        String encryptPassword = getEncryptPassword(username, password);
        LoginRequest model = new LoginRequest();
        model.setUsername(username);
        model.setPassword(encryptPassword);
        String json = mvc.perform(MockMvcRequestBuilders.post("/login").contentType(MediaType.APPLICATION_JSON_UTF8)
                .content(JSONObject.toJSONString(model)))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.code").value("000000"))
                .andDo(MockMvcResultHandlers.print()).andReturn().getResponse().getContentAsString();
        String result = JSONObject.parseObject(json).get("result").toString();
        return JSONObject.parseObject(result).get("token").toString();
    }

    private String getEncryptPassword(String username, String password) throws Exception {
        String json = mvc.perform(MockMvcRequestBuilders.get("/secret?username=" + username)
                .contentType(MediaType.APPLICATION_JSON_UTF8)).andDo(MockMvcResultHandlers.print())
                .andReturn().getResponse().getContentAsString();
        JSONObject responseDTO = JSONObject.parseObject(json);
        return Base64Utils.encodeToUrlSafeString(RSAUtil.encryptByPublicKey(password.getBytes(),
                Base64Utils.decodeFromUrlSafeString(responseDTO.get("result").toString())));
    }

    @Test
    public void userLogout() throws Exception {
        String token = doLogin();
        System.out.println("token = " + token);
        mvc.perform(MockMvcRequestBuilders.get("/logout").header("token", token))
                .andDo(MockMvcResultHandlers.print())
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.code").value("000000"))
                .andReturn();
    }

    @Test
    public void register() throws Exception {
        RegisterRequest model = new RegisterRequest();
        model.setUserName(username);
        model.setPassword(getEncryptPassword(username, password));
        mvc.perform(MockMvcRequestBuilders.post("/register").contentType(MediaType.APPLICATION_JSON_UTF8)
                .content(JSONObject.toJSONString(model)))
                .andDo(MockMvcResultHandlers.print())
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.code").value("000000"))
                .andReturn();
    }
}
```

## FAQ

1. JPA保存实体对象后没有返回自增ID。

    解决：在实体类添加注解：`@GeneratedValue(strategy = GenerationType.IDENTITY)`