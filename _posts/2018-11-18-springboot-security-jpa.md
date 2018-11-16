---
layout: post
title: Spring Boot（八）：Spring Security安全控制（2）
date: 2018-11-16 15:08:44
catalog: true
tags:
    - Spring Boot
    - Spring Security
---

## 前言

前面介绍Spring Security框架进行基本的安全控制，本文在上一节的基础上做修改，增加数据库进行安全控制。

## 快速上手

Web层和Web页面保持不变。

#### 添加依赖

```xml
<dependencies>
    ...
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
    ...
</dependencies>
```

#### 实体模型

创建实体模型类`UserInfo`，具体如下：

```java
@Entity
public class UserInfo {
    @Id
    @GeneratedValue
    private Integer uid;
    @Column(unique = true)
    private String username;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    private String name;
    private String password;
    private String salt;
    private byte state;

    public Integer getUid() {
        return uid;
    }

    public void setUid(Integer uid) {
        this.uid = uid;
    }

    public String getUsername() {
        return username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getSalt() {
        return salt;
    }

    public void setSalt(String salt) {
        this.salt = salt;
    }

    public byte getState() {
        return state;
    }

    public void setState(byte state) {
        this.state = state;
    }

    /**
     * 密码盐.
     * @return
     */
    public String getCredentialsSalt(){
        return this.username+this.salt;
    }

    @Override
    public String toString() {
        return "UserInfo{" +
                "uid=" + uid +
                ", username='" + username + '\'' +
                ", name='" + name + '\'' +
                ", password='" + password + '\'' +
                ", salt='" + salt + '\'' +
                ", state=" + state +
                '}';
    }
}
```

#### 数据库访问层

创建数据访问类`UserInfoRepository`，具体如下：

```java
public interface UserInfoRepository extends JpaRepository<UserInfo, Integer> {
    UserInfo findUserInfoByUsername(String username);
}
```

#### 业务逻辑层

创建业务逻辑类`CustomUserService`，实现通过用户名查找用户，具体如下：

```java
@Component
public class CustomUserService {
    @Autowired
    UserInfoRepository repository;

    public UserInfo loadUserByUsername(String s) throws UsernameNotFoundException {
        System.out.println("s = [" + s + "]");
        UserInfo userInfo = repository.findUserInfoByUsername(s);
        if (userInfo == null) {
            System.out.println("用户名不存在");
            throw new UsernameNotFoundException("用户名不存在");
        }
        return userInfo;
    }
}
```

#### 修改配置类信息

- 主要修改`configure(AuthenticationManagerBuilder auth)`，将内存方式去掉，增加`authenticationProvider()`，具体如下：

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

    @Bean
    public AuthenticationProvider authenticationProvider() {
        return new MyAuthenticationProvider();
    }

    @Override
    public void configure(AuthenticationManagerBuilder auth) throws Exception {
//        auth.inMemoryAuthentication()
//                .passwordEncoder(new BCryptPasswordEncoder())
//                .withUser("user").password(new BCryptPasswordEncoder().encode("12345")).roles("USER");
        auth.authenticationProvider(authenticationProvider());
    }
}
```

- 增加`MyAuthenticationProvider`，实现`AuthenticationProvider`接口，这里可以自定义密码加密方式。

```java
public class MyAuthenticationProvider implements AuthenticationProvider {
    @Autowired
    CustomUserService userService;
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = authentication.getName();
        String password = (String) authentication.getCredentials();
        UserInfo userInfo = userService.loadUserByUsername(username);
        String credential = password+userInfo.getSalt();
        String encodePassword = new MD5PasswordEncoder("").encode(new MD5PasswordEncoder("").encode(credential));
        if (!userInfo.getPassword().equals(encodePassword)) {
            System.out.println("MyAuthenticationProvider.authenticate 密码不正确: " + encodePassword);
            throw new DisabledException("密码不正确");
        }
        return new UsernamePasswordAuthenticationToken(username, password, null);//authorities参数必须传，省了登录不了
    }

    @Override
    public boolean supports(Class<?> aClass) {
        return true;
    }
}
```

**注意：authorities参数必须传，省了登录不了**

这里的密码是原始密码加上盐，然后两次MD5加密，加密类`MD5PasswordEncoder`如下：

```java
public class MD5PasswordEncoder {
    private final static String[] hexDigits = { "0", "1", "2", "3", "4", "5",
            "6", "7", "8", "9", "a", "b", "c", "d", "e", "f" };
    private String salt;
    private String algorithm = "MD5";

    public MD5PasswordEncoder(String salt) {
        this.salt = salt;
    }

    /**
     * 加密
     * @param password
     * @return
     */
    public String encode(String password) {
        String result = null;
        try {
            MessageDigest md = MessageDigest.getInstance(algorithm);
            byte[] digest = md.digest((password + salt).getBytes());
            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < digest.length; i++) {
                String hex = Integer.toHexString(digest[i] & 0xFF);
                if (hex.length() == 1) {
                    sb.append("0");
                }
                sb.append(hex.toUpperCase());
            }
            result = sb.toString();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        return result;
    }

    /**
     * 创建盐值
     * @return
     */
    public static String createSalt() {
        String str = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
        Random random = new Random();
        StringBuilder sb = new StringBuilder();
        for (int i=0; i<32; i++) {
            int index = random.nextInt(62);
            sb.append(str.charAt(index));
        }
        return sb.toString();
    }
}
```

#### 测试

访问`http://localhost:8080/`，点击链接，跳转到`http://localhost:8080/hello`，这时候被重定向到`http://localhost:8080/login`，输入错误的用户名或密码，如果用户身份认证失败，页面就重定向到`/login?error`，输入正确的用户名`admin`和密码`123456`，跳转到`/hello`页面。

## 参考

[Spring security 自定义密码验证（一）](https://blog.csdn.net/f1370335844/article/details/80084085)