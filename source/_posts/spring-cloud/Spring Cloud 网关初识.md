---
title: spring cloud 网关初识
date: 2020-01-29 19:40
author: xp
categories:
- Java
- Spring Cloud
tags:
- Java
- Spring Cloud
- 微服务
- 网关
---

# spring cloud 网关初识

## 什么是网关

*网关*是一个*抽象层*，出现的原因是*微服务架构*的出现，不同的微服务一般会有不同的网络地址，而外部客户端可能需要调用多个服务的接口才能完成一个业务需求，如果让客户端直接与各个微服务通信，会有以下的问题：

- 客户端会多次请求不同的微服务，增加了客户端的复杂性。
- 存在跨域请求，在一定场景下处理相对复杂。
- 认证复杂，每个服务都需要独立认证。
- 难以重构，随着项目的迭代，可能需要重新划分微服务。例如，可能将多个服务合并成一个或者将一个服务拆分成多个。如果客户端直接与微服务通信，那么重构将会很难实施。
- 某些微服务可能使用了防火墙/浏览器不友好的协议，直接访问会有一定的困难。
- 不利于负载均衡实现。

![网关.png](https://i.loli.net/2020/01/31/2bSEAm1ZQUpt4JF.png)

使用 API 网关后的优点如下：

- 简化客户端调用复杂度。
- 易于监控。可以在网关收集监控数据并将其推送到外部系统进行分析。
- 易于认证。可以在网关上进行认证，然后再将请求转发到后端的微服务，而无须在每个微服务中进行认证。
- 减少了客户端与各个微服务之间的直接交互。

> 在Spring Cloud体系中，网关的实现主要有 Netflix的Zuul和 spring cloud 的gateway，主要功能就是提供负载均衡、反向代理、权限认证等。

## Spring Cloud Netflix Zuul

Spring Cloud Netflix Zuul是由Netflix开源的API网关，在微服务架构下，网关作为对外的门户，实现动态路由、监控、授权、安全、调度等功能。

1、在 `spring-cloud-app` 项目中创建 子module `gateway-zuul`：在 pom中添加如下依赖：

```xml
<dependencies>
      <!--<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zuul</artifactId>
        <version>1.4.7.RELEASE</version>
      </dependency>-->
    <!--spring-cloud-starter-netflix-zuul 即为以前的 spring-cloud-starter-zuul-->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
      <version>2.1.1.RELEASE</version>
    </dependency>

  </dependencies>
```

2、在 `gateway-zuul` 的 `application.yml` 添加 配置：

```yml
spring:
  application:
    name: service-gateway
server:
  port: 9005
eureka:
  client:
    serviceUrl:
      # eureka-server地址
      defaultZone: http://127.0.0.1:8089/eureka/
zuul:
  routes:
    #这里的配置表示，访问/consumer/** 直接重定向到 http://jarvan-iv.github.io/2019/10/02/spring%20cloud%E6%9C%8D%E5%8A%A1%E9%97%B4%E8%B0%83%E7%94%A8/
    sn:
      path: /consumer/**
      # 跳转到指定 url
      url: http://jarvan-iv.github.io/2019/10/02/spring%20cloud%E6%9C%8D%E5%8A%A1%E9%97%B4%E8%B0%83%E7%94%A8/
```

3、启动类

```java
@EnableZuulProxy
@SpringBootApplication
public class GatewayZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayZuulApplication.class, args);
    }

}
```

4、测试

通过浏览器访问 `http://localhost:9005/consumer`,发现访问已经被重定向到：`https://jarvan-iv.github.io/2019/10/02/spring%20cloud%E6%9C%8D%E5%8A%A1%E9%97%B4%E8%B0%83%E7%94%A8/`

5、服务化

通过url映射的方式来实现zull的转发有局限性，比如每增加一个服务就需要配置一条内容，另外后端的服务如果是动态来提供，就不能采用这种方案来配置了。实际上在实现微服务架构时，服务名与服务实例地址的关系在eureka server中已经存在了，所以只需要将Zuul注册到eureka server上去发现其他服务，就可以实现对serviceId的映射。

在 pom 中添加依赖：

```xml
<dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
      <!--<version>2.1.4.RELEASE</version>-->
    </dependency>
```

然后在 `application.yml` 添加 配置：

```bash
zuul.routes.xp.path=/producer/**
#serviceId 为 eureka 注册的 服务名
zuul.routes.xp.serviceId=service-provider
```

添加后的zuul栏的配置为：

```yml
zuul:
  routes:
    #这里的配置表示，访问/consumer/** 直接重定向到 http://jarvan-iv.github.io/2019/10/02/spring%20cloud%E6%9C%8D%E5%8A%A1%E9%97%B4%E8%B0%83%E7%94%A8/
    sn:
      path: /consumer/**
      # 跳转到指定 url
      url: http://jarvan-iv.github.io/2019/10/02/spring%20cloud%E6%9C%8D%E5%8A%A1%E9%97%B4%E8%B0%83%E7%94%A8/
    #这里的配置表示，访问/producer/** 直接重定向到 服务service-provider对应的url上
    xp:
      path: /producer/**
      #serviceId 为 eureka 注册的 服务名
      serviceId: service-provider
```

在此之前，你需要启动 `eureka-server` 和 `service-provider`。

通过浏览器多次访问 `http://localhost:9005/producer/xiapeng/`,请求会被转发到服务 `service-provider` 对应的url 上，页面返回：

```json
hello，xiapeng，这是provider 提供的信息
hello，xiapeng，这是provider-2 提供的信息
hello，xiapeng，这是provider 提供的信息
hello，xiapeng，这是provider-2 提供的信息
```

说明通过zuul成功调用了 `service-provider` 服务并且做了均衡负载。

[示例代码下载](https://github.com/Jarvan-IV/spring-cloud-demo)
