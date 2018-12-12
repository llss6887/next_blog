---
title: SpringCloud-Eureka
date: 2017-08-02 23:59:47
urlname: springcloud-eureka
tags:
  - eureka
categories: SpringCloud
---

## Spring Cloud 简介
Spring Cloud为开发人员提供了快速构建分布式系统中一些常见模式的工具（例如配置管理，服务发现，断路器，智能路由，微代理，控制总线）。分布式系统的协调导致了样板模式, 使用Spring Cloud开发人员可以快速地支持实现这些模式的服务和应用程序。他们将在任何分布式环境中运行良好，包括开发人员自己的笔记本电脑，裸机数据中心，以及Cloud Foundry等托管平台。

## 特性
Spring Cloud专注于提供良好的开箱即用经验的典型用例和可扩展性机制覆盖。
分布式/版本化配置
服务注册和发现
路由
service to service调用
负载均衡
断路器
分布式消息传递
<!--more-->

## 注册中心 Eureka

### 简介

Eureka是Spring Cloud Netflix微服务套件中的一部分，可以与Springboot构建的微服务很容易的整合起来。
Eureka包含了服务器端和客户端组件。服务器端，也被称作是服务注册中心，用于提供服务的注册与发现。Eureka支持高可用的配置，当集群中有分片出现故障时，Eureka就会转入自动保护模式，它允许分片故障期间继续提供服务的发现和注册，当故障分片恢复正常时，集群中其他分片会把他们的状态再次同步回来。
客户端组件包含服务消费者与服务生产者。在应用程序运行时，Eureka客户端向注册中心注册自身提供的服务并周期性的发送心跳来更新它的服务租约。同时也可以从服务端查询当前注册的服务信息并把他们缓存到本地并周期性的刷新服务状态。
和dubbo很相似，都是为了治理，分布式服务者中的远程调用问题。只是Eureka的支持功能比dubbo更强大，比如熔断，分布式配置，服务网关等等，这些都是dubbo不具备的。

### 架构图
![](1.png)
- 服务提供者将服务注册到服务注册中心。
- 消费者从服务注册中心订阅到自己需要的服务。
- 注册中心返回消费者服务提供者的地址信息。
- 消费者调用提供者的服务。

### Eureka实践
#### POM 文件
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.3.RELEASE</version>
</parent>
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
</properties>
<dependencies>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-feign</artifactId>
    </dependency>

</dependencies>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Dalston.RC1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
<repositories>
```

#### yml配置文件
```xml
spring:
  application:
    name: springcloud-eureka-server
server:
  port: 8080

eureka:
  instance:
    hostname: localhost
  client:
    fetch-registry: false
    register-with-eureka: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

```
- eureka.client.register-with-eureka ：表示是否将自己注册到Eureka Server，默认为true。
- eureka.client.fetch-registry ：表示是否从Eureka Server获取注册信息，默认为true。
- eureka.client.serviceUrl.defaultZone ：设置与Eureka Server交互的地址，查询服务和注册服务都需要依赖这个地址。默认是http://localhost:8761/eureka ；多个地址可使用 , 分隔。
- 
#### 启动类
```java
@EnableEurekaServer   //开启enreka服务
@SpringBootApplication
public class EurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class, args);
    }
}
```
启动工程访问  locahost:8080
![](2.png)
看到上图，还没有服务注册。

#### 启动双节点
修改yml配置文件如下：
```xml
spring:
  application:
    name: springcloud-eureka-server

---
server:
  port: 8080

spring:
  profiles: eureka1
eureka:
  instance:
    hostname: eureka1
  client:
    fetch-registry: false
    register-with-eureka: false
    service-url:
      defaultZone: http://eureka2:8081/eureka/
---
server:
  port: 8081

spring:
  profiles: eureka2
eureka:
  instance:
    hostname: eureka2
  client:
    fetch-registry: false
    register-with-eureka: false
    service-url:
      defaultZone: http://eureka1:8080/eureka/

```

在hosts文件中增加：

    127.0.0.1 eureka1
    
    127.0.0.1 eureka2

分别以eureka1和eureka2为参数启动：

    java -jar spring-cloud-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=eureka1
    
    java -jar spring-cloud-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=eureka2
分别输入localhost:8080和localhost:8081查看控制台。可以看到eureka1注册到了eureka2中，eureka2注册到了eureka1中
![](3.png)
![](4.png)

### 集群的使用
配置原理和上面双节点是一样的，将注册中心指向其他的注册中心，修改defaultZone为出了自己的所有节点。
然后依次启动每个节点，就完成了集群的部署。

## 服务的提供和调用

上面完成了注册中心的搭建，还有服务提供者和调用者。

### 服务提供者
#### POM依赖
```xml
<dependencies>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-eureka</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>
```

#### yml配置文件
```xml
server:
  port: 8082
eureka:
  client:
    service-url:
      defaultZone: http://eureka1:8080/eureka/
spring:
  application:
    name: eureka-producer
```

#### 服务类
```java
@RestController
public class service {
    @RequestMapping("/hello")
    public String hello(String name){
        return "hello this is first Eureka app   " + name;
    }
}
```

#### 启动类
在启动类上加@EnableDiscoveryClient之后就可以向eureka注册服务了。
```java
@EnableDiscoveryClient
@SpringBootApplication
public class ProducerApp {
    public static void main(String[] args) {
        SpringApplication.run(ProducerApp.class, args);
    }
}
```
启动项目之后访问，localhost:8080和8081.查看是否已经注册成功。
![](5.png)
![](6.png)
到此，服务提供者已经注册成功。

### 服务调用
#### 提供两种调用方式：

##### 客户端负载平衡器：Ribbon：

Ribbon是一个客户端负载均衡器，它可以很好地控制HTTP和TCP客户端的行为。Feign已经使用Ribbon。
Ribbon中的中心概念是指定客户端的概念。每个负载平衡器是组合的组合的一部分，它们一起工作以根据需要联系远程服务器，并且集合具有您将其作为应用程序开发人员（例如使用@FeignClient注释）的名称。Spring Cloud使用RibbonClientConfiguration为每个命名的客户端根据需要创建一个新的合奏作为ApplicationContext。

##### 声明性REST客户端：Feign：

Feign是一个声明式的Web服务客户端。这使得Web服务客户端的写入更加方便 要使用Feign创建一个界面并对其进行注释。它具有可插入注释支持，包括Feign注释和JAX-RS注释。Feign还支持可插拔编码器和解码器。Spring Cloud增加了对Spring MVC注释的支持，并使用Spring Web中默认使用的HttpMessageConverters。Spring Cloud集成Ribbon和Eureka以在使用Feign时提供负载均衡的http客户端。

#### 与Feign的集成
##### POM依赖
和服务提供者一致

##### 启动类
```java
@EnableDiscoveryClient
@EnableFeignClients
@SpringBootApplication
public class EurekaConsumer {
    public static void main(String[] args) {
        SpringApplication.run(EurekaConsumer.class, args);
    }
}
```

##### 定义远程接口
```java
@FeignClient(name = "eureka-producer")
public interface HelloRemote {
    @RequestMapping("/hello")
    public String hello(@RequestParam(value = "name") String name);
}
```

##### 本地controller
```java
@RestController
public class RemoteController {
    @Autowired
    private HelloRemote helloRemote;

    @GetMapping("/remote/{name}")
    public String remoteHello(@PathVariable("name") String name){
        return helloRemote.hello(name);
    }
}
```

##### yml配置文件
```xml
spring:
  application:
    name: eureka-consumer
server:
  port: 8083
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8080/eureka/
```

#### 启动服务
访问localhost:8083/remote/{name}，看页面返回值,完成远程的调用。
![](7.png)

## 服务提供者的负载均衡
### 修改返回值
```java
@RestController
public class service {
    @RequestMapping("/hello")
    public String hello(String name){
        return "hello this is first Eureka app  2 " + name;
    }
}
```

### 启动两个服务提供者
![](8.png)
发现两个都注册到了注册中心。

### 再次访问
访问localhost:8083/remote/{name}  观察
![](9.gif)
不断测试下去，两种结果交替出现，说明注册中心自动实现了服务提供者的负载均衡，如果服务提供者增多，结果也是一样。请求会自动轮询到每个服务来处理。


源码地址：[**戳这里**](https://github.com/llss6887/springcloud/tree/master/eurekacloud)