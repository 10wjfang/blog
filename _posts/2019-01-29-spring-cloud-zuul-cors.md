---
layout: post
title: Spring Cloud跨域解决方案
date: 2019-1-29 10:27:10
catalog: true
tags:
    - Zuul
    - CORS
---

## CORS简介

Cross-Origin Resource Sharing（CORS）跨来源资源共享是一份浏览器技术的规范，提供了 Web 服务从不同域传来沙盒脚本的方法，以避开浏览器的同源策略，是 JSONP 模式的现代版。与 JSONP 不同，CORS 除了 GET 要求方法以外也支持其他的 HTTP 要求。用 CORS 可以让网页设计师用一般的 XMLHttpRequest，这种方式的错误处理比 JSONP 要来的好。另一方面，JSONP 可以在不支持 CORS 的老旧浏览器上运作。现代的浏览器都支持 CORS。

- **CORS**是W3c工作草案，它定义了在跨域访问资源时浏览器和服务器之间如何通信。CORS背后的基本思想是使用自定义的HTTP头部允许浏览器和服务器相互了解对方，从而决定请求或响应成功与否。
- **同源策略**：是浏览器最核心也最基本的安全功能；同源指的是：同协议，同域名和同端口。

## 结合Filter使用

```java
@Component
public class SimpleCORSFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        response.setHeader("Access-Control-Allow-Origin", request.getHeader("Origin"));
        response.setHeader("Access-Control-Allow-Credentials", "true");
        response.setHeader("Access-Control-Allow-Methods", "*");
        response.setHeader("Access-Control-Max-Age", "3600");
        response.setHeader("Access-Control-Allow-Headers", "Content-Type, Accept, X-Requested-With, remember-me");

        chain.doFilter(req, res);
    }

    @Override
    public void init(FilterConfig filterConfig) {
    }

    @Override
    public void destroy() {
    }
}
```

- 容器会扫描到`@Component`注解的Filter
- `Access-Control-Allow-Origin`：标识允许哪个域的请求，一般用法：
  - `*`：不允许携带认证头和cookies
  - `指定域`：如http://172.20.0.206，一般的系统中间都有一个nginx，所以推荐这种
  - `动态设置`：多人协作时，多个前端对接一个后台，这样很方便
- `Access-Control-Allow-Credentials`：是否允许后续请求携带认证信息（cookies）,该值只能是true,否则不返回
- `Access-Control-Request-Method`：该次请求的请求方式
- `Access-Control-Request-Headers`：该次请求的自定义请求头字段
- `Access-Control-Max-Age`：用来指定本次预检请求的有效期，单位为秒，在此期间，不用发出另一条预检请求