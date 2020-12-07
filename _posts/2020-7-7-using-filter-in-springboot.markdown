---
layout: post
title:  "SpringBoot 中使用 Filter 的正确姿势"
date:   2020-7-7 08:00:00 +0800
category: Java
author: Java课代表
excerpt: 三种在 SpringBoot 中使用拦截器的方式
---
Filter 是 JavaEE 中 Servlet 规范的一个组件，位于包`javax.servlet` 中，它可以在 HTTP 请求到达 Servlet 之前，被一个或多个Filter处理。

它的工作流程如图：
![image](https://zhengxl5566.github.io/img/article-img/2020-07/filter.png)


Filter的这个特性在生产环境中有很广泛的应用，如：修改请求和响应、防止xss攻击、包装二进制流使其可以多次读，等等。

实际工作中，我们都是使用 SpringBoot 进行业务开发，本文总结三种 Filter 用法，SpringBoot 版本采用目前最新的 `v2.3.1.RELEASE`

## 1. 编写Filter

要编写 Filter ，只需要实现`javax.servlet.Filter`接口就可以了

```java
public class MyFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("MyFilter");
        // 要继续处理请求，必须添加 filterChain.doFilter()
        filterChain.doFilter(servletRequest,servletResponse);
    }
}
```

Filter 接口有三个方法：`init()`,`doFilter()`,`destroy()`。

其中`doFilter()`需要自己实现，其余两个是`default`的，可以不用实现。

注意：**如果Filter要使请求继续被处理，就一定要调用filterChain.doFilter()！**

## 2. 配置Filter被 Spring 管理

让自定义的 Filter 被 Spring 的 IOC 容器管理，有三种实现方式，各有优缺点。下面课代表给大家总结一下：



### 1. 使用@Component+@Order

在刚刚定义的`MyFilter`类上加上@Component和@Order注解，即可被Spring管理

```java
@Component
@Order(1)
public class MyFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("MyFilter");
        // 要继续处理请求，必须添加 filterChain.doFilter()
        filterChain.doFilter(servletRequest,servletResponse);
    }
}
```

没错就这么简单，这样 MyFilter 就生效了，写个`Controller` 调用一下就可以看到效果。

当有多个Filter时，这里的`@Order(1)`注解会指定执行顺序，数字越小，越优先执行，如果只写`@Order`，默认顺序值是`Integer.MAX_VALUE`。

@Component + @Order 注解方式配置简单，支持自定义 Filter 顺序。缺点是只能拦截所有URL，不能通过配置去拦截指定的 URL。

### 2.@WebFilter+@ServletComponentScan

在 `MyFilter`上添加`@WebFilter`注解，并在启动类上增加`@ServletComponentScan("com.zhengxl.filterdemo.filter")`注解，参数就是Filter所在的包路径，相当于告诉 SpringBoot，去哪里扫描 Filter

```java
@WebFilter(urlPatterns = "/*")
public class MyFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("MyFilter");
        // 要继续处理请求，必须添加 filterChain.doFilter()
        filterChain.doFilter(servletRequest,servletResponse);
    }
}
```

```java
@SpringBootApplication
@ServletComponentScan("com.zhengxl.filterdemo.filter")
public class FilterDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(FilterDemoApplication.class, args);
    }

}
```

@WebFilter+@ServletComponentScan 注解方式支持对 Filter 匹配指定URL，但是不支持指定 Filter 的执行顺序。

### 3. JavaConfig 配置方式

```java
@Configuration
public class FilterConfig {
    @Bean
    public FilterRegistrationBean registerMyFilter(){
        FilterRegistrationBean<MyFilter> bean = new FilterRegistrationBean<>();
        bean.setOrder(1);
        bean.setFilter(new MyFilter());
        // 匹配"/hello/"下面的所有url
        bean.addUrlPatterns("/hello/*");
        return bean;
    }
    @Bean
    public FilterRegistrationBean registerMyAnotherFilter(){
        FilterRegistrationBean<MyAnotherFilter> bean = new FilterRegistrationBean<>();
        bean.setOrder(2);
        bean.setFilter(new MyAnotherFilter());
        // 匹配所有url
        bean.addUrlPatterns("/*");
        return bean;
    }
}
```

通过 Java 代码显式配置 Filter ，功能强大，配置灵活。只需要把每个自定义的 Filter 声明成 Bean 交给 Spring 管理即可，还可以设置匹配的 URL 、指定 Filter 的先后顺序。

## 3. 三种方式对比

以上介绍完 SpringBoot 中三种 Filter的使用姿势，非常简单，下面列个表格总结一下：

| 使用方式                            | 排序 | 指定URL |
| :---------------------------------- | :--: | :-----: |
| @Component<br>@Order                |  1   |    0    |
| @WebFilter<br>@ServletComponentScan |  0   |    1    |
| JavaConfig                          |  1   |    1    |

实际使用过程中，可以按照业务需求选择合适的使用方式，比如：如果编写的过滤器要拦截所有请求，不需要指定URL，那选择最简单的 @Component+@Order 就非常合适。

为方便同学们实战练习，本文代码已上传至
[Java 课代表的 GitHub](https://github.com/zhengxl5566/springboot-demo)

PS：其实还有第四种，web.xml配置，不过这都2020年了，用 SpringBoot 的自动装配或者 JavaConfig 不是更方便吗？

---

![qr-code-1270x300.jpg](https://zhengxl5566.github.io/img/java-helper/qr-code-1270x300.jpg)