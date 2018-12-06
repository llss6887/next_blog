---
title: SpringBoot 集成Dubbo
date: 2017-05-25 20:36:47
urlname: SpringBoot-dubbo
tags:
  - SpringBoot
  - Dubbo
categories: SpringBoot
---

## Dubbo是什么
Dubbo是阿里巴巴SOA服务化治理方案的核心框架，每天为2,000+个服务提供3,000,000,000+次访问量支持，并被广泛应用于阿里巴巴集团的各成员站点。

Dubbo是一个分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案，以及SOA服务治理方案。
<!-- more -->
### 其核心部分包含:
远程通讯: 提供对多种基于长连接的NIO框架抽象封装，包括多种线程模型，序列化，以及“请求-响应”模式的信息交换方式。
集群容错: 提供基于接口方法的透明远程过程调用，包括多协议支持，以及软负载均衡，失败容错，地址路由，动态配置等集群支持。
自动发现: 基于注册中心目录服务，使服务消费方能动态的查找服务提供方，使地址透明，使服务提供方可以平滑增加或减少机器。


## Dubbo能做什么？
透明化的远程方法调用，就像调用本地方法一样调用远程方法，只需简单配置，没有任何API侵入。
软负载均衡及容错机制，可在内网替代F5等硬件负载均衡器，降低成本，减少单点。
服务自动注册与发现，不再需要写死服务提供方地址，注册中心基于接口名查询服务提供者的IP地址，并且能够平滑添加或删除服务提供者。

![](90b90a9a66b74a8287a18f157b136731.jpg)
我的理解就是，服务生产者注册自己的服务，生产者消费自己所需要的服务，注册中心将服务提供者给消费者，如果调用失败，会再返回一台给消费者，实现负载均衡和高可用。

## 系统角色
Provider: 暴露服务的服务提供方。
Consumer: 调用远程服务的服务消费方。
Registry: 服务注册与发现的注册中心。
Monitor: 统计服务的调用次调和调用时间的监控中心。
Container: 服务运行容器。

## 注册中心
Multicast注册中心
Zookeeper注册中心
Redis注册中心
Simple注册中心

### Zookeeper注册中心
Zookeeper是一个分布式的服务框架，是树型的目录服务的数据存储，能做到集群管理数据 ，这里能很好的作为Dubbo服务的注册中心。

Dubbo能与Zookeeper做到集群部署，当提供者出现断电等异常停机时，Zookeeper注册中心能自动删除提供者信息，当提供者重启时，能自动恢复注册数据，以及订阅请求。

## 安装Dubbo

### 安装Zookeeper
zk下载地址：[https://www.apache.org/dyn/closer.cgi/zookeeper/](https://www.apache.org/dyn/closer.cgi/zookeeper/ "https://www.apache.org/dyn/closer.cgi/zookeeper/")

解压 zookeeper-3.4.5.tar.gz
进入到zookeeper-3.4.5/conf中将zoo_simple.cfg 复制一份为zoo.cfg
设置dataDir dataDir=/apps/zookeeper-3.4.5/data
设置dataLogDir dataLogDir=/apps/zookeeper-3.4.5/logs
设置server.1=localhost:2888:3888  第一个端口为通信端口，进行信息交换，第二个leader选举的端口。

注意：zookeeper只支持单数工作，不支持双数工作。

启动  ./zkServer.sh start

### dubbo-admin 
安装因为zookeeper只是一个黑框，我们无法看到是否存在了什么提供者或消费者，这时就要借助Dubbo-Admin管理平台来实时的查看，也可以通过这个平台来管理提者和消费者。
dubbo的所有源码可在 [https://github.com/alibaba/dubbo](https://github.com/alibaba/dubbo "https://github.com/alibaba/dubbo") 上下载
之后进入到dubbo-admin里面利用maven打包。
`mvn package -Dmaven.skip.test=true ` 
将打包好的war包放在tomcat下启动，注意修改![](bda3356bfed5414f8181aee7af6f16aa.png)
![](c95ae46b260241239368f988dc8d538c.png)
的zookeeper地址。
![](8f0ad31361304293a97406e58ed793c6.png)
当前没有服务。

## 服务提供者代码
### POM文件：
```xml
<dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
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
            <groupId>com.alibaba.spring.boot</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>2.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.4.8</version>
        </dependency>
        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.10</version>
        </dependency>
    </dependencies>
```

### yaml文件：
```yaml
server:
  port: 8100
spring:
  dubbo:
    application:
      id: liss
      name: liss
    registry:
      address: zookeeper://182.61.34.13:2181
    protocol:
      id: dubbo
      name: dubbo
      port: 20881
      status : server
    scan: com.lls.server.impl
```

### 启动类：
```java
@CrossOrigin
@SpringBootApplication
@EnableDubboConfiguration
public class ProduceServer {
    public static void main(String[] args) {
        SpringApplication.run(ProduceServer.class, args);
    }
}
```

### 服务接口：
```java
package com.lls.server;

public interface DubboServer {
    public String getName(String name);
}
```

### 服务实现：
@Service提供参数如下：
  version = "1.0.0",
  application = "${dubbo.application.id}",
  protocol = "${dubbo.protocol.id}",
  registry = "${dubbo.registry.id}"
  这里使用version，value自定义，用来区分服务。
```java
@Component("dubboServer")
@Service(version = "1.0.0", interfaceClass = DubboServer.class)
public class DubboServerImpl implements DubboServer {

    @Override
    public String getName(String name){
        return "Dubbo " + name;
    }
}
```

### 启动服务：
![](5fc9021345d244889361c0e930f588da.png)

![](af06c5f608d74f0c89f282c83e7e44e1.png)
服务已经注册！

## 调用者代码：

### YML文件：
和服务者一致。注意服务端口。

### POM 文件：
和服务者一致。

### 调用代码：
通过@Reference注解来调用。
```java
@Component
public class ClientServerImpl implements ClientServer {

    @Reference(version = "1.0.0")
    private DubboServer dubboServer;

    @Override
    public String conService(String name){
        return dubboServer.getName(name);
    }
}
```

### 启动调用者
打开 localhost:8111/dubbo/liss 返回 Dubbo liss.

搭建成功。

在上面的调用中，我们是在服务提供者和调用者里面都写了DubboServer这个接口，在实际开发中，我们是将需要提供的服务类的接口抽象成一个api，多个服务实现这个api，每个服务有自己的标示，在调用者使用的时候，根据标示来区分自己是需要哪一个服务，就调用哪一个服务。


源码地址：[**戳这里**](https://github.com/llss6887/springboot/tree/master/springbootdubbo "**戳这里**")















































