---
title: Spring Cloud 网关Gateway入门
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

# Spring Cloud 网关Gateway入门

Spring Cloud Gateway提供了一个在Spring生态系统之上构建的API网关，包括：Spring 5，Spring Boot 2和Project Reactor。它旨在微服务架构提供一种简单有效的统一的 API 路由管理方式，目标是替代 Netflix Zuul，其不仅提供统一的路由方式，并且基于 Filter 链的方式提供了网关基本的功能，例如：安全、监控、埋点和限流等。

- 可以对路由指定 Predicate（断言）和 Filter（过滤器）
- 集成Hystrix的断路器功能
- 集成 Spring Cloud 服务发现功能
- 易于编写的 Predicate（断言）和 Filter（过滤器）
- 请求限流功能
- 支持路径重写

## 术语

- Route（路由）：网关的基本构建块。它由ID，目标URI，一组断言和一组过滤器定义组成。如果断言为真，则路由匹配
- Predicate（断言）：这是[Java 8](https://docs.oracle.com/javase/8/docs/api/java/util/function/Predicate.html)的断言。输入类型是[Spring Framework ServerWebExchange](https://docs.spring.io/spring/docs/5.0.x/javadoc-api/org/springframework/web/server/ServerWebExchange.html)。它可以匹配HTTP请求中的所有内容，例如 请求头或请求参数
- Filter（过滤器）：这是 org.springframework.cloud.gateway.filter.GatewayFilter 的实例，我们可以使用它修改请求和响应。

## 工作流程

下图概述了Spring Cloud Gateway的工作原理：

![springcloud gateway.jpg](https://i.loli.net/2020/01/31/4peEh1DrABvVbRw.jpg)

客户端向 Spring Cloud Gateway 发出请求。如果在 `Gateway Handler Mapping` 中找到与请求相匹配的路由，则将其发送到 Gateway Web Handler。Handler 再通过指定的过滤器链来将请求发送到我们实际的服务执行业务逻辑，然后返回。
过滤器之间用虚线分开是因为过滤器可能会在发送代理请求之前（“pre”）或之后（“post”）执行业务逻辑

## 示例

### 在配置文件 yml 中配置

1、在 `spring-cloud-app` 项目中创建 子module `gateway-cloud`：在 pom中添加如下依赖：

```xml
<dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-gateway</artifactId>
      <version>2.1.4.RELEASE</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

  </dependencies>
```

2、在 `gateway-cloud` 的 `application.yml` 添加 配置：

```yml
spring:
  application:
    name: service-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
        - id: add_request_header_route
          uri: http://jarvan-iv.github.io/2019/10/02/spring%20cloud%E6%9C%8D%E5%8A%A1%E9%97%B4%E8%B0%83%E7%94%A8/
          predicates:
          - Path=/consumer/**
          #filters:
          #  - AddRequestHeader=X-Request-red, blue

server:
  port: 9005
eureka:
  client:
    serviceUrl:
      # eureka-server地址
      defaultZone: http://127.0.0.1:8089/eureka/
```

> 在此之前，首先得启动 eureka-server

配置说明：

- `spring.cloud.gateway.discovery.locator.enabled`:是否与服务注册于发现组件进行结合，通过 `serviceId` 转发到具体的服务实例。默认为 `false`，设为 `true` 便开启通过服务中心的自动根据 `serviceId` 创建路由的功能。
- `id`：我们自定义的路由 ID，保持唯一。
- `uri`: 目标服务地址。
- `predicates`: 路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。
- `filters`: 过滤规则。

上面这段配置的意思是，配置了一个 id 为 `add_request_header_route` 的路由规则，当访问地址 `http://localhost:9005/consumer` 时会自动转发到地址：`https://jarvan-iv.github.io/2019/10/02/spring%20cloud%E6%9C%8D%E5%8A%A1%E9%97%B4%E8%B0%83%E7%94%A8/`。配置完成启动项目即可在浏览器访问进行测试可以正常跳转。

此外，启动 `eureka-server` 和 两个`service-provider`，联系访问 `http://localhost:9005/SERVICE-PROVIDER/xiapeng`,会得到如下结果：

```json
hello，xiapeng，这是provider 提供的信息
hello，xiapeng，这是provider-2 提供的信息
...
```

说明当 `spring.cloud.gateway.discovery.locator.enabled` 为 `true`时，`Spring Cloud Gateway` 会为我们注册到`eureka`的服务（`service-provider`等）自动创建了对应的路由，但是这里的 路径是大写的。通过 `http://localhost:9005/SERVICE-PROVIDER/xiapeng`就会匹配到 `SERVICE-PROVIDER`服务对应的 url(`/xiapeng`)上。

![eureka注册的服务.jpg](https://i.loli.net/2020/02/01/Ht2DThdIW81RfPU.jpg)

### 通过@Bean自定义 RouteLocator，在启动主类 Application 中配置

```java
@SpringBootApplication
public class GatewayCloudApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayCloudApplication.class, args);
    }

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                .route("add_request_header_route", r -> r.path("/consumer/**")
                        .uri("http://jarvan-iv.github.io/2019/10/02/spring%20cloud%E6%9C%8D%E5%8A%A1%E9%97%B4%E8%B0%83%E7%94%A8/"))
                .build();
    }

}
```

上述代码配置和 `application.yml` 作用等同。

以上介绍了 `Spring Cloud Gateway`的基本使用，下一篇介绍它强大的 `Route Predicate Factories和GatewayFilter Factories`

[示例github代码下载](https://github.com/Jarvan-IV/spring-cloud-demo)
