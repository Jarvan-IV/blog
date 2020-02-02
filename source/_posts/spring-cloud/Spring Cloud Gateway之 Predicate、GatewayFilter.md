---
title: Spring Cloud Gateway之 Predicate、GatewayFilter
date: 2020-01-31 21:21
author: xp
categories:
- Java
- Spring Cloud
- Spring Cloud Gateway
tags:
- Java
- Spring Cloud
- 微服务
- 网关
- Spring Cloud Gateway
---

# Spring Cloud Gateway之 Predicate、GatewayFilter

## Route Predicate Factories

Spring Cloud Gateway将路由匹配作为Spring WebFlux HandlerMapping基础架构的一部分。 Spring Cloud Gateway包括许多内置的Route Predicate工厂。 所有的Predicate都与HTTP请求的不同属性匹配。你可以将多个Route Predicate工厂可以进行逻辑 `and` 组合，这里有一张图总结了 `Spring Cloud` 内置的11种 `Route Predicate Factory` 的作用（有细微变化），详情可参考官网 [Route Predicate Factory官网手册](https://cloud.spring.io/spring-cloud-gateway/reference/html/#gateway-request-predicates-factories)

![spring-cloud-gateway-predicate.png](https://i.loli.net/2020/02/01/vs1uDdX7lBfyZeH.png)

### 时间匹配

> 包括 `After Route Predicate Factory`、`Before Route Predicate Factory`、`Between Route Predicate Factory`

Predicate 支持设置一个时间，在请求进行转发的时候，可以通过判断在这个时间*之前*或者*之后*或者*之间*进行转发。比如这样配置：

`After Route Predicate Factory`示例：

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: time_after_route
          uri: http://www.ityouknow.com/
          predicates:
            - Path=/after/**
            - After=2020-02-01T16:50:00+08:00[Asia/Shanghai]
```

上面的示例是指，请求时间在 `2020-02-01 16:50:00` 之后的所有请求都转发到地址`http://ityouknow.com`。+08:00是指时间和 `UTC` 时间相差八个小时，时间地区为`Asia/Shanghai`。

Spring 是通过 ZonedDateTime 来对时间进行的对比，`ZonedDateTime`是 Java 8 中日期时间功能里，用于表示带时区的日期与时间信息的类，ZonedDateTime 支持通过时区来设置时间，中国的时区是：`Asia/Shanghai`。

顾名思义，`Before Route Predicate Factory` 与上例效果恰好相反：

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: time_before_route
          uri: http://www.ityouknow.com/
          predicates:
            - Path=/before/**
            - Before=2020-02-01T18:00:00+08:00[Asia/Shanghai]
```

上面的示例是指，请求时间在 `2020-02-01 18:00:00` 之前访问 `http://localhost:9005/before`都会被转发到 `http://www.ityouknow.com/`。

`Between Route Predicate Factory`示例：

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: time_between_route
          uri: http://www.ityouknow.com/
          predicates:
            - Path=/between/**
            - Between=2020-02-01T17:00:00+08:00[Asia/Shanghai], 2020-02-01T18:00:00+08:00[Asia/Shanghai]
```

上面的示例是指，在 `2020-02-01 17:00:00`和`2020-02-01 18:00:00` 之间请求访问 `http://localhost:9005/before`都会被转发到 `http://www.ityouknow.com/`。

### 根据 Cookie 匹配

The Cookie route predicate factory takes two parameters, the cookie name and a regexp (which is a Java regular expression). This predicate matches cookies that have the given name and whose values match the regular expression. The following example configures a cookie route predicate factory

`Cookie Route Predicate` 可以接收两个参数，一个是 Cookie name ,一个是正则表达式，路由规则会通过获取对应的 Cookie name 值和正则表达式去匹配。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: cookie_route
        uri: https://example.org
        predicates:
        - Cookie=chocolate, ch.p
```

此路由匹配 cookie 名字为 `chocolate`,匹配值为 `ch.p` 正则表达式。使用 `curl` 命令模拟请求：

```bash
curl http://localhost:9005 --cookie "chocolate=ch.p"
```

则会返回页面代码，如果去掉 `--cookie "chocolate=ch.p"`，后台汇报 404 错误。

### 根据 Header 属性匹配

`Header Route Predicate` 和 Cookie Route Predicate 一样，也是接收 2 个参数，一个 Header 中属性名称和一个正则表达式，这个属性值和正则表达式匹配则执行。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: https://example.org
        predicates:
        - Header=X-Request-Id, \d+
```

此路由匹配 请求header 中的 name为 `X-Request-Id`，值为 `\d+的正则表达式`(只能为一个或多个数字)。

使用 curl 测试，命令行输入并测试:

```bash
curl http://localhost:9005 -H "X-Request-Id:996"
```

### 根据 Host 匹配

`Host Route Predicate Factory` 接收一组参数，一组匹配的域名列表，这个模板是一个 ant 分隔的模板，用 `.` 号作为分隔符。它通过参数中的主机地址作为匹配规则。支持URI模板变量（例如{sub} .myhost.org）

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: https://example.org
        predicates:
        - Host=**.somehost.org,**.anotherhost.org
```

```bash
curl http://localhost:9005  -H "Host: www.somehost.org"
curl http://localhost:9005  -H "Host: beta.anotherhost.org"
```

此路由可以匹配 Host中的 header 值为 `www.somehost.org`、`beta.somehost.org`、`www.anotherhost.org`。

### 根据请求类型匹配

`Method Route Predicate Factory`,可以根据 `POST`、`GET`、`PUT`、`DELETE` 等不同的请求类型来进行路由。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: https://example.org
        predicates:
        - Method=GET,POST
```

此路由匹配请求方法是`GET`或`POST`.

```bash
curl http://localhost:9005
curl -X POST http://localhost:9005
```

### 根据请求路径匹配

`Path Route Predicate Factory` 接收一个匹配路径的参数来判断是否走路由。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: https://example.org
        predicates:
        - Path=/red/{segment},/blue/{segment}
```

此路由匹配，例如：`/red/1` 或 `/red/blue` 或 `/blue/green`。

```bash
curl http://localhost:9005/red/1
```

### 根据请求参数匹配

`Query Route Predicate` 支持传入两个参数，一个是属性名一个为属性值，属性值可以是正则表达式。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: https://example.org
        predicates:
          - Path=/query/**
          - Query=color, gree.
```

此路由匹配请求中，路径前缀为`query` 且包含 `color` 属性并且参数值是以 `gree开头的长度为五位的字符串`才会进行匹配和路由。

```bash
curl http://localhost:9005/query?color=green
```

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: query_name_route
        uri: https://example.org
        predicates:
        - Path=/query/**
        - Query=user
```

此路由匹配请求中 路径前缀为`query` 且包含 有 `user` 作为参数名(值可以为空)的请求。

```bash
curl http://localhost:9005/query?user=
```

### 根据地址进行匹配

`RemoteAddr Route Predicate Factory` 也支持通过设置某个 ip 区间号段的请求才会路由，它 接受 CIDR(无类别域间路由（Classless Inter-Domain Routing） 符号(IPv4 或 IPv6 )字符串的列表(最小大小为1)，例如 192.168.0.1/16 (其中 192.168.0.1 是 IP 地址，16 是子网掩码)。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: https://example.org
        predicates:
        - RemoteAddr=192.168.1.1/24
```

此路由匹配远程地址是例如192.168.1.10等

```bash
curl http://localhost:9005
```

### 根据权重来访问

`Weight Route Predicate Factory` 有两个参数，group和weight（一个int）,权重是按组计算的：

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: weight_high
        uri: https://weighthigh.org
        predicates:
        - Weight=group1, 8
      - id: weight_low
        uri: https://weightlow.org
        predicates:
        - Weight=group1, 2
```

这条路线会将大约80％的流量转发到weighthigh.org，将大约20％的流量转发到weightlow.org

## GatewayFilter

路由过滤器可用于修改进入的HTTP请求和返回的HTTP响应，路由过滤器只能指定路由进行使用。`Spring Cloud Gateway` 内置了多种路由过滤器，他们都由 `GatewayFilter` 的工厂类来产生。由于内置的 `GatewayFilter Factories` 比较多,详情可参考官网 [GatewayFilter Factories官网手册](https://cloud.spring.io/spring-cloud-gateway/reference/html/#gatewayfilter-factories)。接下来只以 `AddRequestHeader GatewayFilter`为例介绍：

在 在两个`service-provider`(地址为 localhost:9010和9012)的 `com.xp.controller.UserController.java`中，分别定义一个url:

```java
@RequestMapping("/req/color")
    public String getColor(@RequestParam(name = "color",required = false) String color) {
        return "color is " + color + " provider";
    }
```

```java
@RequestMapping("/req/color")
    public String getColor(@RequestParam(name = "color",required = false) String color) {
        return "color is " + color + " provider 2";
    }
```

然后分别重启两个`service-provider`。

在 `gateway-cloud`的 `application.yml` 中添加如下配置：

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: add_request_parameter_route
          uri: http://localhost:9010/
          filters:
            - AddRequestParameter=color, blue
          predicates:
            - Method=GET
```

这样就会给匹配的每个 `GET请求` 添加上 `color=blue` 的键值。

通过浏览器多次访问： `http://localhost:9005/req/color` 出现：

```json
color is blue provider
color is blue provider
color is blue provider
```

证明网关在转发的过程中已经通过 filter 添加了设置的参数和值（但是都是访问的localhost:9010,也就是 provider1，并没有负载均衡，想要做到负载均衡，uri 的配置栏 必须配置*服务名*）。

## 服务化路由转发

在 [GatewayFilter](GatewayFilter)中我们进行如下配置：

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: add_request_parameter_route
          uri: http://localhost:9010/
          filters:
            - AddRequestParameter=color, blue
          predicates:
            - Method=GET
```

由于 uri 配置为 `http://localhost:9010/` 所以访问时，都是访问 `localhost:9010`(provider)所在的服务，现在更改配置如下：

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: add_request_parameter_route
          #uri: http://localhost:9010/
          #格式为：lb://应用注册服务名
          uri: lb://SERVICE-PROVIDER
          filters:
            - AddRequestParameter=color, blue
          predicates:
            - Method=GET
```

然后再通过浏览器多次访问： `http://localhost:9005/req/color` 出现：

```json
color is blue provider
color is blue provider 2
......
```

provider和provider 2交替出现，说明我们的路由已经服务化，做到了负载均衡。

[示例github代码下载](https://github.com/Jarvan-IV/spring-cloud-demo)
