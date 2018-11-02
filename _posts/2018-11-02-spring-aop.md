---
layout: post
title: Java动态代理：Spring AOP的实现
date: 2018-11-2 16:10:41
catalog: true
tags:
    - Java
---

## 什么是动态代理

调用Proxy返回一个代理对象，使用这个代理对象执行你的方法时，它会先执行代理对象里的方法再执行你的业务方法。

## 作用

业务层代码专注业务处理，添加新功能时，如日志，事务等，不影响业务代码。

## 实践

1、添加一个接口：

```java
public interface UserManager {
    public void add(String username, String password);
    public void del(int userId);
}
```

2、实现接口：

```java
public class UserManagerImpl implements UserManager {
    @Override
    public void add(String username, String password) {
        System.out.println("---------UserMangerImpl.add()-----------");
    }

    @Override
    public void del(int userId) {
        System.out.println("---------UserMangerImpl.del()-----------");
    }
}
```

3、添加一个代理类：

```java
public class SecurityHandler implements InvocationHandler {
    private Object target;

    private void checkSecurity() {
        System.out.println("-------------check security------------");
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        checkSecurity();
        return method.invoke(target, args);
    }

    public Object createProxyInstance(Object target) {
        this.target = target;
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                this);
    }
}
```

4、测试

```java
public class Test {
    public static void main(String[] args) {
        UserManager userManager = new UserManagerImpl();
        userManager.add("aaa", "123");
        userManager.del(123);

        SecurityHandler handler = new SecurityHandler();
        UserManager userManagerProxy = (UserManager) handler.createProxyInstance(new UserManagerImpl());
        userManagerProxy.add("aaa", "123");
        userManagerProxy.del(123);
    }
}
```