---
title: spring cloud服务间调用
date: 2019-10-02 19:58:00
author: xp
categories:
- Java
- Spring Cloud
tags:
- Java
- Spring Cloud
- 微服务
---

# spring cloud服务间调用

上一篇文章我们介绍了eureka服务注册中心的搭建，这篇文章介绍一下如何利用服务注册中心（`eureka-server`），搭建一个简单的服务调用案例。

案例中有三个角色：服务注册中心（`eureka-server`）、服务提供者（`service-provider`）、服务消费者（`service-consumser`），其中服务注册中心用我们上一篇的`eureka单机版`启动既可，流程是首先启动 `eureka-server`，然后启动`service-provider`并注册到`eureka-server`中，`service-consumser`从`eureka-server`中获取服务并调用。

> 假设你已经读过了上一篇 `spring cloud注册中心Eureka`，并已经 搭建了 `eureka-server` 、`service-provider`、 `service-consumser` 工程。

## service-provider

在 `service-provider` 增加一个类 `UserController.java`:

```java

@RestController
public class UserController {

    @RequestMapping("/{name}")
    public String getUser(@PathVariable("name") String name) {
        return "hello，" + name + "，这是provider 1 提供的信息";
    }
}
```

## service-consumser

### 通过 RestTemplate 调用service-provider

> RestTemplate是一个http请求的客户端工具，它不是类似HttpClient的东东，也不是类似Jackson，jaxb等工具，但它封装了这些工具．是Spring 调用`http`的client端核心类．顾名思义，与其它template类如JdbcTemplate一样，它封装了与http server的通信的细节，使调用端无须关心http接口响应的消息是什么格式，统统使用Java pojo 来接收请求结果。

在 `service-consumser` 中 增加类：`ConsumerController.java`：

```java
@RestController
public class ConsumerController {

    @RequestMapping("test")
    public String getFromProvider(){
        return new RestTemplate().getForObject("http://localhost:9010/xp",String.class);
    }
}
```

访问 `http://localhost:9011/test`，会输出：

```json
hello，xp，这是provider 提供的信息
```

### 通过 Feign 调用

> Feign是一个声明式Web Service客户端。使用Feign能让编写Web Service客户端更加简单, 它的使用方法是定义一个接口，然后在上面添加注解，同时也支持JAX-RS标准的注解。Feign也支持可拔插式的编码器和解码器。Spring Cloud对Feign进行了封装，使其支持了Spring MVC标准注解和HttpMessageConverters。Feign可以与Eureka和Ribbon组合使用以支持负载均衡。

使用Fegin调用服务，需要在`service-consumser` 中修改以下几处：

1.在 `pom` 中增加 fegin 依赖：

```xml
    <!--fegin 调用-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
        <version>2.0.2.RELEASE</version>
    </dependency>
```

2.在启动类 `ServiceConsumserApplication.java` 增加 `@EnableFeignClients` 注解：

```java
@EnableDiscoveryClient
@SpringBootApplication
@EnableFeignClients
public class ServiceConsumserApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceConsumserApplication.class, args);
    }
}
```

2.编写 fegin client

增加一个接口 `FeginRemoteInterface.java`

```java
@FeignClient(name = "service-provider")
public interface FeginRemoteInterface {

    @RequestMapping("/{name}")
    String hello(@RequestParam(value = "name") String name);
}
```

3.在 `ConsumerController.java` 中注入`Fegin Client`：

```java
@RestController
public class ConsumerController {

    @Autowired
    private FeginRemoteInterface feginRemoteInterface;

    @RequestMapping("test")
    public String getFromProvider(){
        return new RestTemplate().getForObject("http://localhost:9010/xp",String.class);
    }

    @RequestMapping("/fegin/{name}")
    public String getUserByFegin(@PathVariable("name") String name) {
        return "这是 通过feigin 调用的接口：" + feginRemoteInterface.hello(name);
    }
}
```

#### fegin 负载均衡

1. 复制一份 `service-provider` 工程，命名为 `service-provider2`,修改 `application.yml`：中的 `server.port` 为 `9012`.

2. 修改 `service-provider2` 的 `UserController.java`：

```java
@RestController
public class UserController {

    @RequestMapping("/{name}")
    public String getUser(@PathVariable("name") String name) {
        return "hello，" + name + "，这是 provider2 提供的信息";
    }
}
```

3.启动 `service-provider2`。
此时通过 `http://localhost:8087/` 访问 `eureka-server`，会看到如图的效果：

![34b90774918e4719826d6909b8a1adcd.png](https://i.loli.net/2019/10/07/ZSDsmcEPwgRJfNe.png)

可以看到 `SERVICE-PROVIDER` 服务中有两个 `UP`（` xp:service-provider:9010 , xp:service-provider:9012`），表明注册中心有2个 `SERVICE-PROVIDER` 服务。

> 由于本人操作的时候，启动 `service-provider2` 的时候没有更改端口导致 `端口被占用` 而失败,所以引起了注册中心有 2个 `DOWN (1)`，不过我们看 注册服务的数量，只看 `UP`，至于很长时间之后，`eureka-server` 很长时间都会显示那两个`DOWN`，这就涉及到知识盲区了？

访问`http://localhost:9011/fegin/xp`4次数，得到的结果如下:

```json
这是 通过feigin 调用的接口：hello，xp，这是provider 提供的信息
这是 通过feigin 调用的接口：hello，xp，这是 provider2 提供的信息
这是 通过feigin 调用的接口：hello，xp，这是provider 提供的信息
这是 通过feigin 调用的接口：hello，xp，这是 provider2 提供的信息
```

不断的进行测试下去会发现两种结果交替出现，说明两个服务中心自动提供了`服务均衡负载`的功能。如果我们将服务提供者的数量在提高为N个，测试结果一样，请求会`自动轮询`到每个服务端来处理。
