---
title: SpringBoot 集成ActiveMQ
date: 2017-05-24 20:36:47
urlname: SpringBoot-activeMQ
tags:
  - SpringBoot
  - ActiveMQ
categories: ActiveMQ
---

## JMS是什么
### JMS的基础

JMS是什么：JMS是Java提供的一套技术规范
JMS干什么用：用来异构系统 集成通信，缓解系统瓶颈，提高系统的伸缩性增强系统用户体验，使得系统模块化和组件化变得可行并更加灵活
通过什么方式：生产消费者模式（生产者、服务器、消费者）
![](1252cf72e1b24603988b216d0ac26b92.png)
jdk，kafka，activemq……

<!-- more -->
## JMS消息传输模型

点对点模式（一对一，消费者主动拉取数据，消息收到后消息清除）
点对点模型通常是一个基于拉取或者轮询的消息传送模型，这种模型从队列中请求信息，而不是将消息推送到客户端。这个模型的特点是发送到队列的消息被一个且只有一个接收者接收处理，即使有多个消息监听者也是如此。

发布/订阅模式（一对多，数据生产后，推送给所有订阅者）
发布订阅模型则是一个基于推送的消息传送模型。发布订阅模型可以有多种不同的订阅者，临时订阅者只在主动监听主题时才接收消息，而持久订阅者则监听主题的所有消息，即时当前订阅者不可用，处于离线状态。

![](beb8bbca2d284051931657cad03935a3.png)
queue.put（object）  数据生产
queue.take(object)    数据消费

## JMS核心组件
Destination：消息发送的目的地，也就是前面说的Queue和Topic。
Message ：从字面上就可以看出是被发送的消息。
Producer： 消息的生产者，要发送一个消息，必须通过这个生产者来发送。
MessageConsumer： 与生产者相对应，这是消息的消费者或接收者，通过它来接收一个消息。
![](1408d3c96baf4c16ae7936654bb1195a.png)

通过与ConnectionFactory可以获得一个connection
通过connection可以获得一个session会话。

## 常见的类JMS消息服务器

JMS消息服务器 ActiveMQ
分布式消息中间件 Metamorphosis
分布式消息中间件 RocketMQ

## 其他MQ
.NET消息中间件 DotNetMQ
基于HBase的消息队列 HQueue
Go 的 MQ 框架 KiteQ
AMQP消息服务器 RabbitMQ
MemcacheQ 是一个基于 MemcacheDB 的消息队列服务器。

## 为什么需要消息队列
消息系统的核心作用就是三点：解耦，异步和并行
以用户注册的案列来说明消息系统的作用

### 用户注册的一般流程
![](82d025b6ebc045f29fe366e488ef6123.jpg)
问题：随着后端流程越来越多，每步流程都需要额外的耗费很多时间，从而会导致用户更长的等待延迟

### 用户注册的并行执行
![](d4884748ecfc4e99a1db81ceb47ca160.jpg)
问题：系统并行的发起了4个请求，4个请求中，如果某一个环节执行1分钟，其他环节再快，用户也需要等待1分钟。如果其中一个环节异常之后，整个服务挂掉了。

### 用户注册的最终一致
![](f52ee2d994be4c7fa17a574876cc3d49.jpg)
保证主流程的正常执行、执行成功之后，发送MQ消息出去。
需要这个destination的其他系统通过消费数据再执行，最终一致。![](9f8ce850752742b5a62d9dd4d2efb361.png)

## ActiveMQ的安装
1.下载ActiveMQ
去官方网站下载：[http://activemq.apache.org/](http://activemq.apache.org/ "http://activemq.apache.org/")

2.运行ActiveMQ
解压缩apache-activemq-5.5.1-bin.zip，
修改配置文件activeMQ.xml，将0.0.0.0修改为localhost
```xml
<transportConnectors>
	<transportConnector name="openwire" uri="tcp://localhost:61616"></transportConnector>
	<transportConnector name="ssl"     uri="ssl://localhost:61617"></transportConnector>
	<transportConnector name="stomp"   uri="stomp://localhost:61613"></transportConnector>
	<transportConnector uri="http://localhost:8081"></transportConnector>
	<transportConnector uri="udp://localhost:61618"></transportConnector>
<transportConnectors>
```
然后双击apache-activemq-5.5.1\bin\activemq.bat运行ActiveMQ程序。
启动ActiveMQ以后，登陆：http://localhost:8161/admin/。
![](69742e33fb1d4bb18e40c8df24f50cf9.png)

## 和Spring Boot 集成
### pom文件
```xml
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.3.RELEASE</version>
    </parent>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.7</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- springboot整合activemq -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-activemq</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```
添加：spring-boot-starter-activemq依赖

### produce生产者编写
```java
@Service("registerMailboxProducer")
public class ActiveMQProducer {
    @Autowired
    private JmsMessagingTemplate jmsMessagingTemplate;

    public void send(Destination destination, String json){
        jmsMessagingTemplate.convertAndSend(destination, json);
    }
}

```

### cusumer消费者编写
```java
@Component
public class ActiveMQConsumer {
    @Autowired
    private ActiveMQProducer activeMQProducer;

    @JmsListener(destination = "my_queue")
    public void distribute(String json){
        System.out.println(json);
    }
}
```

### yml文件
```yaml
server:
  port: 8080
spring:
  activemq:
    in-memory: true
    broker-url: tcp://localhost:61616
    pool:
      enabled: false

messages:
  queue: my_queue

```

### 测试编写
```java
@SpringBootApplication
@RestController
public class ActiveMqServer {

    @Value("${messages.queue}")
    private String queue;

    @Autowired
    private ActiveMQProducer activeMQProducer;

    public static void main(String[] args) {
        SpringApplication.run(ActiveMqServer.class, args);
    }


    @GetMapping("/msg/{str}")
    public void set_queue(@PathVariable String str){
        Destination amq = new ActiveMQQueue(queue);
        activeMQProducer.send(amq, str);
    }
}
```

## 启动服务
访问localhost:8080/msg/hello-activeMQ,查看控制台打印。 Spring Boot 和ActiveMQ的集成就完成了。如果需要topic，只需要更换下，创建的API就可以了。用ActiveMQTopic。

源码地址： [**戳这里**](https://github.com/llss6887/springboot/tree/master/springbootactiveMq)














































