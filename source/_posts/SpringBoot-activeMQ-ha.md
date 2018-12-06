---
title: ActiveMQ 的高可用
date: 2017-05-24 20:36:47
urlname: activeMQ-ha
tags:
  - SpringBoot
  - ActiveMQ
categories: ActiveMQ
---

## 什么是高可用
高可用就是将各个系统通过网络或者其他手段组成一个群体，共同对外提供服务，消除单点故障，比如solr利用zookeeper进行主从节点的选举，选取leader对外提供服务。内部各节点进行通讯，当leader挂掉之后，会重新选取一个新的leader，并对外提供服务，当挂掉的leader恢复之后，自动加入到集群里面，保证整个服务能一直对外提供服务。
## 什么是负载均衡
将请求分摊到对个操作单元中进行执行，共同完成工作任务。比如nginx+tomcat的负载均衡，当访问量大时，将这些请求按照一定的规则分别分发的内部的tomcat服务，而不是单一的tomcat来响应这些请求。nginx的负载方式：
<!-- more -->
rr:  按照时间顺序
ip_hash: 按照访问用户的ip的hash值
fair: 按照响应时间，时间短的优先
url_hash: 根据请求的url固定到同一个服务

## ActiveMQ的高可用
### ShareNothing  Master/Slave
master和slave不共享存储
master接收到持久化消息后，会同步给slave
master出现故障，slave自己成为master或者关闭服务
如果master和slave之间通信故障，salve会认为master宕机，如果配置为自己成为master会出现脑裂
非持久的消息不会同步
slave只能同步连接后的消息，之前的无法同步，宕机之后的数据也不会自动同步，有丢失消息的风险
![](f242aa6ba66246ca989f70cc157ab31e.png)

### Shared Database Master/Slave
master和slave共享一份数据。
master和slave只有一个有写的权限。
master和slave之间不需要做数据同步。
master和slave通过竞争锁来竞争master。
![](a3390bbeb59e470f821eb0c7298f5dde.png)
在高可用环境中，用的最多的就是利用zookeeper来协调，zookeeper本身是高可用的架构，主备之间自动同步。zookeeper还提供全局的锁，多个服务在启动的时候在zookeeper创建一个带编号的临时文件，序号小的为master，并设置监听，当master和zookeeper断开时，zookeeper会删除掉master的临时文件，并通知其他服务，序号小的为master，当master重新连接，会重新向zookeeper注册。

## 搭建高可用环境

### 端口分配

| 服务名称  | web端口  | 消息服务端口  | 通信端口  |
| ------------ | ------------ | ------------ | ------------ |
| mq01  |  8161 | 60001  | 61600  |
| mq02  |  8162 |  60002 |  61600 |   
| mq03 |  8163 |  60003 |  61600  |   
注： web端口在jetty.xml文件中配置

### 配置XML

1、配置broker
```xml
<broker xmlns="http://activemq.apache.org/schema/core" brokerName="mqStore" dataDirectory="${activemq.data}">
```
brokerName一致的服务会加入到同一个集群中。
2、配置共享的db
```xml
<persistenceAdapter>
	<!--<kahaDB directory="${activemq.data}/kahadb"/>-->
	<replicatedLevelDB
		directory="${activemq.data}/storedb"
		replicas="3"  <!-- 副本数量 -->
		bind="tcp://127.0.0.1:61600"  <!-- 通信端口 -->
		zkAddress="182.61.34.13:2181" <!-- zk地址，多个逗号分隔 --> 
		hostname="mq01"     <!-- 对应各自的hostname -->
		zkPath="/activemq/mqstores"  <!-- 关于ActiveMQ的zk目录 -->
	/>
</persistenceAdapter>
```
### 启动每一个activeMQ
activemq start 
### 查看zookeeper中的文件
./zkCli.sh     get /activemq/mqstores
![](1af1bf4b7e4f4045a16ac96b21b54964.png)
出现上图，表示三台启动成功。numChildren = 3，表示有三台已经启动，并向zookeeper注册。
关掉一台ActiveMQ，显示只有两台启动。
![](f8a7be6245af41889a422ef04fdb8981.png)
### 测试

#### YML文件
```yaml
server:
  port: 8080
spring:
  activemq:
    in-memory: true
    broker-url: failover:(tcp://localhost:60001,tcp://localhost:60002,tcp://localhost:60002)
    pool:
      enabled: false

messages:
  queue: my_queue
```

#### 生产者
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
#### 消费者
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
![](94c2bebbfecb4ede962cb3093e6a1fc1.png)
#### 发送数据
![](23096f4a1ebc45cd8c14039fa014cd7f.png)
当前连接的是60002的activeMQ服务。现在关闭60002。
![](fa8b725997fa4a029ff9da43a42f1c9d.png)
重新选举出了60001为master，服务依然可用，现在关闭60001。
![](9fb9803776c8422b9aa9a6e4cd381df0.png)
60001关闭后，集群中只有一个节点，整个服务都不可用了。

## 总结
在一般公司中，做到高可用，基本就已经可以了，ActiveMQ占资源，如果有特别大的数据需求，Kafka,RocketMQ都适合大数据的场景，天然的集群分布式优势，实际中根据自己的需求来选择适合自己的队列。


源码地址：[ **戳这里**](https://github.com/llss6887/springboot/tree/master/springbootactiveMQ-HA " **戳这里**")














































