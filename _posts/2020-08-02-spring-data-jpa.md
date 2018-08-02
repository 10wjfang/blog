---
layout: post
title: Spring Data JPA介绍和使用
date: 2018-8-2 09:33:52
catalog: true
tags:
    - Spring Boot
---

## Spring Data JPA介绍
-----

Spring Data JPA(Java Persistence API)是Spring基于ORM 框架、JPA规范的基础上封装的一套JPA应用框架，可使开发者用极简的代码即可实现对数据的访问和操作。它提供了包括增删改查等在内的常用功能，且易于扩展！学习并使用 Spring Data JPA可以极大提高开发效率！

> 注意:JPA是一套规范，不是一套产品，那么像Hibernate,TopLink,JDO他们是一套产品，如果说这些产品实现了这个JPA规范，那么我们就可以叫他们为JPA的实现产品。

## 接口介绍
---

### Repository接口

Repository 接口是 Spring Data 的一个核心接口，它不提供任何方法，开发者需要在自己定义的接口中声明需要的方法。

**说明：**
- Repository是一个空接口，即是一个标记接口。
- 若我们定义的接口继承了Repository，则该接口会被IOC容器识别为一个Repository Bean注入到IOC容器中，只要遵循Spring Data的方法定义规范，就无需写实现类。
- 与继承 Repository 等价的一种方式，就是在持久层接口上使用 @RepositoryDefinition 注解，并为其指定 domainClass 和 idClass 属性。如下两种方式是完全等价的。

**其它接口：**
- CrudRepository： 继承 Repository，实现了一组 CRUD 相关的方法。
- PagingAndSortingRepository： 继承 CrudRepository，实现了一组分页排序相关的方法。
- JpaRepository： 继承 PagingAndSortingRepository，实现一组 JPA 规范相关的方法。

### CrudRepository接口

CrudRepository 接口提供了最基本的对实体类的添删改查操作：

- T save(T entity);//保存单个实体 
- Iterable<T> save(Iterable<? extends T> entities);//保存集合        
- T findOne(ID id);//根据id查找实体         
- boolean exists(ID id);//根据id判断实体是否存在         
- Iterable<T> findAll();//查询所有实体,不用或慎用!         
- long count();//查询实体数量         
- void delete(ID id);//根据Id删除实体         
- void delete(T entity);//删除一个实体 
- void delete(Iterable<? extends T> entities);//删除一个实体的集合         
- void deleteAll();//删除所有实体,不用或慎用! 

### PagingAndSortingRepository接口

PagingAndSortingRepository接口提供了分页与排序的功能：
- Iterable<T> findAll(Sort sort); //排序 
- Page<T> findAll(Pageable pageable); //分页查询（含排序功能） 

### JpaRepository接口

JpaRepository接口提供了JPA的相关功能： 
- List<T> findAll(); //查找所有实体 
- List<T> findAll(Sort sort); //排序、查找所有实体 
- List<T> save(Iterable<? extends T> entities);//保存集合 
- void flush();//执行缓存与数据库同步 
- T saveAndFlush(T entity);//强制执行持久化 
- void deleteInBatch(Iterable<T> entities);//删除一个实体集合 

## 基本查询
---

基本查询也分为两种，一种是spring data默认已经实现，一种是根据查询的方法来自动解析成SQL。

### 预先生成方法

spring data jpa默认预先生成了一些基本的CURD的方法，例如：增、删、改等等。

使用：
```java
public interface UserInfoRepository extends JpaRepository<UserInfo, Integer> {
}
```

### 自定义查询

如果预先生成的方法不满足，还可以自定义查询。自定义的简单查询就是根据方法名来自动生成SQL，主要的语法是findXXBy,readAXXBy,queryXXBy,countXXBy, getXXBy后面跟属性名称：

```java
UserInfo findUserInfoByUsername(String username);
```

也使用一些加一些关键字And、 Or：

```java
UserInfo findUserInfoByUsernameAndPassword(String username, String password);
```

修改、删除、统计也是类似语法：

```java
Long deleteById(Long id);

Long countByUserName(String userName);
```

基本上SQL体系中的关键词都可以使用，例如：LIKE、 IgnoreCase、 OrderBy：

```java
List<User> findByEmailLike(String email);

User findByUserNameIgnoreCase(String userName);
    
List<User> findByUserNameOrderByEmailDesc(String email);
```

具体的关键字，使用方法和生产成SQL如下表所示：

| Keyword | Sample | JPQL snippet |
| --- | --- | --- |
| And | findByLastnameAndFirstname | … where x.lastname = ?1 and x.firstname = ?2 |
| Or | findByLastnameOrFirstname | … where x.lastname = ?1 or x.firstname = ?2 |
| Is,Equals | findByFirstnameIs,findByFirstnameEquals | … where x.firstname = ?1 |
| Between | findByStartDateBetween | … where x.startDate between ?1 and ?2 |
| LessThan | findByAgeLessThan | … where x.age < ?1 |
| LessThanEqual | findByAgeLessThanEqual | … where x.age ⇐ ?1 |
| GreaterThan | findByAgeGreaterThan | … where x.age > ?1 |
| GreaterThanEqual | findByAgeGreaterThanEqual | … where x.age >= ?1 |
| After | findByStartDateAfter | … where x.startDate > ?1 |
| Before | findByStartDateBefore | … where x.startDate < ?1 |
| IsNull | findByAgeIsNull | … where x.age is null |
| IsNotNull,NotNull | findByAge(Is)NotNull | … where x.age not null |
| Like | findByFirstnameLike | … where x.firstname like ?1 |
| NotLike | findByFirstnameNotLike | … where x.firstname not like ?1 |
| StartingWith | findByFirstnameStartingWith | … where x.firstname like ?1 (parameter bound with appended %) |
| EndingWith | findByFirstnameEndingWith | … where x.firstname like ?1 (parameter bound with prepended %) |
| Containing | findByFirstnameContaining | … where x.firstname like ?1 (parameter bound wrapped in %) |
| OrderBy | findByAgeOrderByLastnameDesc | … where x.age = ?1 order by x.lastname desc |
| Not | findByLastnameNot | … where x.lastname <> ?1 |
| In | findByAgeIn(Collection ages) | … where x.age in ?1 |
| NotIn | findByAgeNotIn(Collection age) | … where x.age not in ?1 |
| TRUE | findByActiveTrue() | … where x.active = true |
| FALSE | findByActiveFalse() | … where x.active = false |
| IgnoreCase | findByFirstnameIgnoreCase | … where UPPER(x.firstame) = UPPER(?1) |

## 复杂查询
---

在实际的开发中我们需要用到分页、删选、连表等查询的时候就需要特殊的方法或者自定义SQL。

### 分页查询

分页查询在实际使用中非常普遍了，spring data jpa已经帮我们实现了分页的功能，在查询的方法中，需要传入参数Pageable ,当查询中有多个参数的时候Pageable建议做为最后一个参数传入。

```java
Page<User> findALL(Pageable pageable);
    
Page<User> findByUserName(String userName,Pageable pageable);
```

Pageable 是spring封装的分页实现类，使用的时候需要传入页数、每页条数和排序规则。

```java
@Test
public void testPageQuery() throws Exception {
	int page=1,size=10;
	Sort sort = new Sort(Direction.DESC, "id");
    Pageable pageable = PageRequest.of(page, size, sort);
    userRepository.findALL(pageable);
    userRepository.findByUserName("testName", pageable);
}

### 限制查询

有时候我们只需要查询前N个元素，或者支取前一个实体。

```java
User findFirstByOrderByLastnameAsc();

User findTopByOrderByAgeDesc();

Page<User> queryFirst10ByLastname(String lastname, Pageable pageable);

List<User> findFirst10ByLastname(String lastname, Sort sort);

List<User> findTop10ByLastname(String lastname, Pageable pageable);
```

### 自定义SQL

由于某些原因我们想使用自定义的SQL来查询，spring data也是完美支持的；在SQL的查询方法上面使用@Query注解，如涉及到删除和修改在需要加上@Modifying。注意: JPQL不支持使用 INSERT。也可以根据需要添加 @Transactional 对事务的支持，查询超时的设置等。UPDATE 或 DELETE 操作需要使用事务, 此时需要定义 Service 层，在 Service 层的方法上添加事务操作。默认情况下, SpringData 的每个方法上有事务, 但都是一个只读事务，他们不能完成修改操作!

```java
@Modifying
@Query("update User u set u.userName = ?1 where u.id = ?2")
int modifyByIdAndUserId(String  userName, Long id);//返回值int，表示影响行数
	
@Transactional
@Modifying
@Query("delete from User where id = ?1")
void deleteByUserId(Long id);
  
@Transactional(timeout = 10)
@Query("select u from User u where u.emailAddress = ?1")
User findByEmailAddress(String emailAddress);
```
> 注意： 这里SQL语句里字段的名称是实体类的属性名称，不是数据库表的字段名称。

@Query的nativeQuery=true，这样可以使用原生的SQL查询。

```java
@Query(value="SELECT count(id) FROM jpa_persons", nativeQuery=true)
long getTotalCount();
```

在写jpql语句时，查询条件的参数的表示有以下2种方式：

- 索引参数：索引值从1开始，查询中'?x'的个数要和方法的参数个数一致，且顺序也要一致。
- 命名参数（推荐使用这种方式）：可以用':参数名'的形式，在方法参数中使用@Param（"参数名"）注解，这样就可以不用按顺序来定义形参。

```java
@Query("select u from User u where u.emailAddress = :email")
User findByEmailAddress(@Param("email") String emailAddress);
```


### 多表查询

多表查询在spring data jpa中有两种实现方式，第一种是利用hibernate的级联查询来实现，第二种是创建一个结果集的接口来接收连表查询后的结果，这里主要第二种方式。

首先需要定义一个结果集的接口类：

```java
public interface LinkView {
    int getId();
    String getTitle();
    String getUrl();
    String getIconUrl();
    String getCategory();
    long getCreatetime();
}
```

查询的方法返回类型设置为新创建的接口：

```java
@Query("select l.title as title,l.id as id,l.url as url,l.iconUrl as iconUrl,l.categoryId as categoryId," +
            "l.createtime as createtime,c.name as category from Link l left join Category c on l.categoryId = c.id" +
            " where l.uId = ?1")
Page<LinkView> findAllLink(Integer uId, Pageable pageable);

@Query("select l.title as title,l.id as id,l.url as url,l.iconUrl as iconUrl,l.categoryId as categoryId," +
            "l.createtime as createtime,c.name as category from Link l left join Category c on l.categoryId = c.id" +
            " where l.uId = ?1 and l.categoryId in ?2")
Page<LinkView> findAllLinkByCategoryIn(Integer uId, List<Integer> cIds, Pageable pageable);
```

> 在运行中Spring会给接口（LinkView）自动生产一个代理类来接收返回的结果，代码汇总使用getXX的形式来获取。

## 实体类介绍

---

实体类的注解，会根据注解自动生成表字段。

- @Table指定表名，不填，默认为类名。
- @Column指定字段名，不填，默认为属性名。
  - columnDefinition：自定义字段，如`columnDefinition = "varchar(50)"`
- @Id，声明此属性为主键。
- @GeneratedValue 指定主键的生成策略。
  - TABLE：使用表保存id值
  - IDENTITY：identitycolumn
  - SEQUENCR ：sequence
  - AUTO：根据数据库的不同使用上面三个

### 使用枚举

使用枚举的时候，我们希望数据库中存储的是枚举对应的String类型，而不是枚举的索引值，需要在属性上面添加 `@Enumerated(EnumType.STRING)` 注解。

```java
@Enumerated(EnumType.STRING) 
@Column(nullable = true)
private UserType type;
```

### 不需要和数据库映射的属性

正常情况下我们在实体类上加入注解@Entity，就会让实体类和表相关连如果其中某个属性我们不需要和数据库来关联只是在展示的时候做计算，只需要加上`@Transient`属性既可。

```java
@Transient
private String category;
```


## 快速上手

---

### 配置文件

在pom包里面添加jpa的相关包引用。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

在application.properties中添加配置

```properties
spring.datasource.url=jdbc:mysql://192.168.241.131:3306/test?useUnicode=true&characterEncoding=utf-8&characterSetResult=utf-8
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.jdbc.Driver

spring.jpa.properties.hibernate.hbm2ddl.auto=update
spring.jpa.properties.hibernate.dialect=com.fang.config.MySQL5DialectUTF8
spring.jpa.show-sql=true
```

### 数据库层代码

实体类映射数据库表

```java
@Entity
public class User {
    @Id
    @GeneratedValue
    private long id;
    @Column(nullable = false, unique = true)
    private String userName;
    @Column(nullable = false)
    private String password;
    @Column(nullable = false)
    private int age;
    ...
}
```

继承JpaRepository类

```java
public interface UserRepository extends JpaRepository<User, Long> {
    User findById(long id);
    Long deleteById(Long id);
}
```

### 业务层处理

service调用jpa实现相关的增删改查

```java
@Service
public class UserServiceImpl implements UserService{

    @Autowired
    private UserRepository userRepository;

    @Override
    public List<User> getUserList() {
        return userRepository.findAll();
    }

    @Override
    public User findUserById(long id) {
        return userRepository.findById(id);
    }

    @Override
    public void save(User user) {
        userRepository.save(user);
    }

    @Override
    public void edit(User user) {
        userRepository.save(user);
    }

    @Override
    public void delete(long id) {
        userRepository.delete(id);
    }
}
```

> 注意：在用springboot+jpa逆向生成表时候,遇到一个问题就是,数据库的编码是utf-8,但是默认生成的表的编码却是latin,造成了中文不能写入的问题。
出现这个问题的原因:一般情况我们使用的mysql方言为:org.hibernate.dialect.MySQL5Dialect它默认返回的是@OverridepublicStringgetTableTypeString(){return"ENGINE=InnoDB";}
解决办法:覆写MySql5InnoDBDialect 的getTableTypeString 方法.
```java
public class MySQL5DialectUTF8 extends MySQL5InnoDBDialect {
    @Override
    public String getTableTypeString() {
        return "ENGINE=InnoDB DEFAULT CHARSET=utf8";
    }
}
```