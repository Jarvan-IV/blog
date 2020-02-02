---
title: 注册中心Eureka
date: 2019-10-02 18:58:00
author: xp
categories:
- Java
- Spring Cloud
tags:
- Java
- Spring Cloud
- 微服务
---

# 注册中心Eureka

## 简介

> Eureka是Netflix开发的基于Http协议的轻量级服务治理中间件，分为服务端和客户端两个部分，服务端提供服务注册与发现能力，客户端SDK提供给微服务使用，简化与服务端的交互。Eureka拥有很多强大的特性，比如动态更新路由配置、集群高可用、分区容错、保证最终一致性等等，基于Http协议的服务器端接口也容易与客户端交互，是一款实现微服务注册中心的理想的中间件。

### 没有它

在数年前的SOA时代，服务之间调用通过直接配置IP地址+端口实现，后来服务多了形成了集群，则针对每个服务集群配置一个LVS硬负载，就在服务之间就通过配置硬负载的代理地址+端口实现相互调用。显而易见，这种方式带来的麻烦之处就是随着服务数量的庞大，配置维护工作也越来越麻烦，运维人员每次都得停止服务，修改每个服务实例的配置文件，而后重启服务，工作效率低下且容易出错；如果集群中某些服务是跨机房部署，更是容易受到忽视，从而花费开发和测试人员大量的排查精力。

### 拥有他

Eureka中间件介入之后，就取代了原来服务集群中维护服务路由列表以及选择负载的相关功能，硬负载将被Eureka客户端SDK中软负载算法所取代，而服务的路由配置都将自动注册在Eureka服务端中。

## 示例

> 项目基于较新的 spring boot 2.x

新建一个项目 `spring-cloud-app`,这是一个父项目，主要的`pom` 内容信息如下：

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <cloud.version>Greenwich.SR4</cloud.version>
    <boot.version>2.1.11.RELEASE</boot.version>
</properties>
<dependencyManagement>
    <!--这儿使用的 spring boot 版本为 2.1.11.RELEASE-->
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
        <version>${boot.version}</version>
        <exclusions>
          <exclusion>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
          </exclusion>
        </exclusions>
      </dependency>
      <dependency>
        <groupId>com.google.code.gson</groupId>
        <artifactId>gson</artifactId>
        <version>2.6</version>
      </dependency>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>2.1.11.RELEASE</version>
      </dependency>
      <!--这儿使用的 spring-cloud-starter-netflix-eureka 版本为 2.1.4.RELEASE-->
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        <version>2.1.4.RELEASE</version>
      </dependency>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        <version>2.1.4.RELEASE</version>
      </dependency>
      <!--cloud 版本为 Greenwich.SR4-->
      <!--<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>${cloud.version}</version>
        <type>pom</type>
        <scope>runtime</scope>
      </dependency>-->
    </dependencies>
  </dependencyManagement>
  <repositories>
    <repository>
      <id>spring-snapshots</id>
      <name>Spring Snapshots</name>
      <url>https://repo.spring.io/snapshot</url>
      <snapshots>
        <enabled>true</enabled>
      </snapshots>
    </repository>
    <repository>
      <id>spring-milestones</id>
      <name>Spring Milestones</name>
      <url>https://repo.spring.io/milestone</url>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
    </repository>
  </repositories>
```

### eureka-server

在 `spring-cloud-app` 中新建 `Module` 为 `eureka-server`，`pom` 依赖：

```xml
<dependencies>
    <!--eureka-server-->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
      <!--<version>2.1.4.RELEASE</version>-->
    </dependency>
  </dependencies>
```

`application.yml`:

```yml
spring:
  application:
    name: eureka-server
server:
  port: 8089
eureka:
  client:
    #表示是否将自己注册到Eureka Server，默认为true。
    register-with-eureka: false
    #表示是否从Eureka Server获取注册信息，默认为true。
    fetch-registry: false
    serviceUrl:
      #设置与Eureka Server交互的地址，查询服务和注册服务都需要依赖这个地址。默认是http://localhost:8761/eureka ；多个地址可使用 , 分隔。
      defaultZone: http://localhost:${server.port}/eureka/
```

启动，只需要在启动类加上 `@EnableEurekaServer` 注解即可：

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

访问  `http://localhost:8089/` 即可看到以下页面：

![EurekaServer.png](https://i.loli.net/2019/10/07/6wlOhyFuEtBkNxI.png)

在 `Instances currently registered with Eureka` 栏下，可看到当前没有 `服务` 注册到 `eureka`

### service-provider

在 `spring-cloud-app` 中新建 `Module` 为 `service-provider`，这是模拟服务的提供方，`pom` 如下：

```xml
<dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
      <!--<version>2.1.4.RELEASE</version>-->
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
      <!--<version>2.1.11.RELEASE</version>-->
    </dependency>
  </dependencies>
```

`application.yml`:

```yml
spring:
  application:
    name: service-provider
server:
  port: 9010
eureka:
  client:
    serviceUrl:
      # eureka-server地址
      defaultZone: http://localhost:8089/eureka/
```

最后在启动类上加上注解 `@EnableDiscoveryClient`:

```java
@EnableDiscoveryClient
@SpringBootApplication
public class ServiceProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceProviderApplication.class, args);
    }
}

```

这时候查看注册中心：
![service-provider注册.png](https://i.loli.net/2019/10/07/GmLqkDfzcTVlO42.png)

会发现注册中心多了一个服务 `SERVICE-PROVIDER`。

### service-consumser

在 `spring-cloud-app` 中新建 `Module` 为 `service-consumser`，这是模拟服务的消费方，`pom` 和 `service-provider` 一样。

修改 `application.yml`:

```yml
spring:
  application:
    name: service-consumser
server:
  port: 9011
eureka:
  client:
    serviceUrl:
      # eureka-server地址
      defaultZone: http://localhost:8089/eureka/
```

启动的流程也和 `service-provider`一样：

```java
@EnableDiscoveryClient
@SpringBootApplication
public class ServiceConsumserApplication {

    /**
     * 查看注册到 eureka-server服务信息
     * @param dc dc
     * @return CommandLineRunner
     */
    @Bean
    CommandLineRunner runner(DiscoveryClient dc) {
        return args -> dc.getInstances("service-provider")
                .forEach(si -> System.out.println(String.format(
                        "Found %s %s:%s", si.getServiceId(), si.getHost(), si.getPort())));
    }
    public static void main(String[] args) {
        SpringApplication.run(ServiceConsumserApplication.class, args);
    }
}
```

同样，启动之后 访问 `http://localhost:8089/` 可以看到 `service-consumser`服务。

## eureka高可用

>注册中心这么关键的服务，如果是单点话，遇到故障就是毁灭性的。在一个分布式系统中，服务注册中心是最重要的基础部分，理应随时处于可以提供服务的状态。为了维持其可用性，使用集群是很好的解决方案。Eureka通过互相注册的方式来实现高可用的部署，所以我们只需要将Eureke Server配置其他可用的serviceUrl就能实现高可用部署。

### 搭建eureka集群

1. 只需将`eureka-server`分别又指向其它两个节点即可，并将 `register-with-eureka` 和 `fetch-registry`改为 `true`，修改后的`application.yml`如下：

```yml
---
spring:
  application:
    name: eureka-server
  profiles: node1
server:
  port: 8087
eureka:
  instance:
    hostname: node1
  client:
    #表示是否将自己注册到Eureka Server，默认为true。
    #register-with-eureka: false
    #表示是否从Eureka Server获取注册信息，默认为true。
    #fetch-registry: false
    serviceUrl:
      #设置与Eureka Server交互的地址，查询服务和注册服务都需要依赖这个地址。默认是http://localhost:8761/eureka ；多个地址可使用 , 分隔。
      defaultZone: http://node2:8088/eureka/,http://node3:8089/eureka/
---
spring:
  application:
    name: eureka-server
  profiles: node2
server:
  port: 8088
eureka:
  instance:
    hostname: node2
  client:
    #表示是否将自己注册到Eureka Server，默认为true。
    #register-with-eureka: false
    #表示是否从Eureka Server获取注册信息，默认为true。
    #fetch-registry: false
    serviceUrl:
      #设置与Eureka Server交互的地址，查询服务和注册服务都需要依赖这个地址。默认是http://localhost:8761/eureka ；多个地址可使用 , 分隔。
      defaultZone: http://node1:8087/eureka/,http://node3:8089/eureka/
---
spring:
  application:
    name: eureka-server
  profiles: node3
server:
  port: 8089
eureka:
  instance:
    hostname: node3
  client:
    #表示是否将自己注册到Eureka Server，默认为true。
    #register-with-eureka: false
    #表示是否从Eureka Server获取注册信息，默认为true。
    #fetch-registry: false
    serviceUrl:
      #设置与Eureka Server交互的地址，查询服务和注册服务都需要依赖这个地址。默认是http://localhost:8761/eureka ；多个地址可使用 , 分隔。
      defaultZone: http://node1:8087/eureka/,http://node2:8088/eureka/
---
```

2.在 `host` 中配置：

```json
127.0.0.1 node1
127.0.0.1 node2
127.0.0.1 node3
```

3.执行命令 `mvn package`，打包项目后，再运行

```bash
java -jar eureka-server.jar --spring.profiles.active=node1
java -jar eureka-server.jar --spring.profiles.active=node2
java -jar eureka-server.jar --spring.profiles.active=node3
```

4.访问 `http://localhost:8087/`,结果如图：

![image.png](https://i.loli.net/2019/10/07/Omug5XwsRqdHAY8.png)

> 再集群模式下，需要将 配置 `eureka.client.register-with-eureka` 和 `eureka.client.fetch-registry`都设置为 `true`,也就是默认值。否则这些`节点`会出现在界面 `General Info`的 `unavailable-replicas`中，也就是不可用！

## 工作流程

![34b90774918e4719826d6909b8a1adcd.png](https://i.loli.net/2019/10/07/MKoItcYUv3XzwDT.png)

上图更进一步的展示了3个角色之间的交互。

- Service Provider会向Eureka Server做Register（服务注册）、Renew（服务续约）、Cancel（服务下线）等操作。
- Eureka Server之间会做注册服务的同步，从而保证状态一致
- Service Consumer会向Eureka Server获取注册服务列表，并消费服务

### Register

Register（服务注册），这个接口会在Service Provider启动时被调用来实现服务注册。同时，当Service Provider的服务状态发生变化时（如自身检测认为Down的时候），也会调用来更新服务状态。

- ApplicationResource类接收Http服务请求，调用PeerAwareInstanceRegistryImpl的register方法

- PeerAwareInstanceRegistryImpl完成服务注册后，调用replicateToPeers向其它Eureka Server节点（Peer）做状态同步（异步操作）

### Renew

Renew（服务续约）操作由Service Provider定期调用，类似于heartbeat。主要是用来告诉Eureka Server Service Provider还活着，避免服务被剔除掉。重要参数：

- `instance.leaseRenewalIntervalInSeconds`，Renew频率。默认是30秒，也就是每30秒会向Eureka Server发起Renew操作。

- `instance.leaseExpirationDurationInSeconds`，服务失效时间。默认是90秒，也就是如果Eureka Server在90秒内没有接收到来自Service Provider的Renew操作，就会把Service Provider剔除。

### Cancel

Cancel（服务下线）一般在Service Provider shut down的时候调用，用来把自身的服务从Eureka Server中删除，以防客户端调用不存在的服务。

### Fetch Registries

Fetch Registries由Service Consumer调用，用来获取Eureka Server上注册的服务。
为了提高性能，服务列表在Eureka Server会缓存一份，同时每30秒更新一次。

### Eviction

Eviction（失效服务剔除）用来定期（默认为每60秒）在Eureka Server检测失效的服务，检测标准就是超过一定时间没有Renew的服务。

默认失效时间为90秒，也就是如果有服务超过90秒没有向Eureka Server发起Renew请求的话，就会被当做失效服务剔除掉。

失效时间可以通过`eureka.instance.leaseExpirationDurationInSeconds`进行配置，定期扫描时间可以通过`eureka.server.evictionIntervalTimerInMs`进行配置。

[示例github代码下载](https://github.com/Jarvan-IV/spring-cloud-demo)