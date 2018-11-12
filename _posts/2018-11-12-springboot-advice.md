---
layout: post
title: Spring Boot（六）：统一异常处理
date: 2018-11-12 14:27:31
catalog: true
tags:
    - Spring Boot
---

## 前言

我们在做Web应用的时候，请求处理过程中发生错误是非常常见的情况。Spring Boot提供了一个默认的映射：`/error`，当处理中抛出异常之后，会转到该请求中处理，并且该请求有一个全局的错误页面用来展示异常内容。

## 统一异常处理

虽然，Spring Boot中实现了默认的error映射，但是在实际应用中，上面你的错误页面对用户来说并不够友好，我们通常需要去实现我们自己的异常提示。

## 快速上手

**1. 新建异常信息类**

```java
public enum ExceptionMsg {
    SUCCESS("000000", "成功"),
    FAILED("999999", "失败"),
    ParamError("000001", "参数错误"),
    PasswordError("000002", "密码错误")
    ;
    private String code;
    private String msg;

    ExceptionMsg(String code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    public String getCode() {
        return code;
    }

    public void setCode(String code) {
        this.code = code;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

**2. 新建返回类**

```java
public class ResponseBean {
    private String code;
    private String msg;

    public ResponseBean(String code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    public ResponseBean(ExceptionMsg exceptionMsg) {
        this.code = exceptionMsg.getCode();
        this.msg = exceptionMsg.getMsg();
    }

    public String getCode() {
        return code;
    }

    public void setCode(String code) {
        this.code = code;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

带有数据的返回类：

```java
public class ResponseDataBean extends ResponseBean {
    private Object data;

    public ResponseDataBean(String code, String msg, Object data) {
        super(code, msg);
        this.data = data;
    }

    public ResponseDataBean(ExceptionMsg exceptionMsg, Object data) {
        super(exceptionMsg);
        this.data = data;
    }

    public Object getData() {
        return data;
    }

    public void setData(Object data) {
        this.data = data;
    }
}
```

**3. 自定义异常**

```java
public class ParamErrorException extends RuntimeException {
    public ParamErrorException() {
    }

    public ParamErrorException(String message) {
        super(message);
    }
}
```

**4. Controller抛出异常**

```java
@RestController
public class HomeController {
    @RequestMapping("/index")
    public ResponseBean index(String name) throws Exception {
        if (StringUtils.isEmpty(name)) {
            throw new ParamErrorException();
        }
        return new ResponseBean(ExceptionMsg.SUCCESS);
    }
}
```

**5. 创建异常处理类**

```java
@RestControllerAdvice
public class ExceptionController {
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ResponseBean globalException(HttpServletRequest request, Throwable ex) {
        return new ResponseBean("999999",ex.getMessage());
    }

    @ExceptionHandler(ParamErrorException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ResponseBean paramErrorException(HttpServletRequest request, Throwable ex) {
        return new ResponseBean(ExceptionMsg.ParamError);
    }
}
```

**6. 测试**

访问：`http://localhost:8080/index?name=123`，
返回：
```json
{
    "code": "000000",
    "msg": "成功"
}
```

访问：`http://localhost:8080/index`
返回：
```json
{
    "code": "000001",
    "msg": "参数错误"
}
```

## 总结

通过`@ControllerAdvice`统一定义不同Exception映射到不同错误处理页面；通过`@RestControllerAdvice`实现返回JSON格式的异常处理。

## 参考

[Spring Boot中Web应用的统一异常处理](http://blog.didispace.com/springbootexception/)