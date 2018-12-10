---
title: SpringBoot集成RecoketMQ
date: 2017-06-18 23:59:47
urlname: SpringBoot_RecoketMQ
tags:
  - RocketMQ
  - SpringBoot
categories: RocketMQ
---

## Rocket的安装

### 下载

官方地址：[http://rocketmq.apache.org/dowloading/releases/](http://rocketmq.apache.org/dowloading/releases/)

### 部署Rocket

将下载的二进制包解压
![](1.png)
打开 runbroker.cmd和runserver.cmd 修改运行参数
<!--more-->
![](2.png)
更改参数主要是防止内存过大导致的内存不足等问题，你要是，内存够大，请忽略。
![](3.png)
![](4.png)

### 启动
#### 启动namesrv
![](5.png)
#### 启动broker
![](6.png)

出现以上 说明启动完成。

## 与SpringBoot的集成（原始的API）
这是基于原始的API，没有封装，不通用，很简单，加个配置文件就可以了。
### 目录结构
![](7.png)
### pom文件

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
    <!-- springboot整合recoketmq -->
    <dependency>
        <groupId>com.alibaba.rocketmq</groupId>
        <artifactId>rocketmq-client</artifactId>
        <version>3.2.6</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.44</version>
    </dependency>
</dependencies>
```
### rocketMQ的配置文件

```xml
com.liss.producer.group=liss_producer
com.liss.namesrvAddr=192.168.25.7:9876

com.liss.consumer.group=liss_consumer

com.liss.topic=liss_topic

com.liss.tag=liss_tag
```
### 消息实体

```java
@Getter
@Setter
@AllArgsConstructor
@EqualsAndHashCode
@ToString
public class DemoEvent {

    private String name;

    private String message;
}
```
### 生产者

```java
@PropertySource(value = "classpath:recoketMQ.properties")
@Component
@Slf4j
public class DemoProducer {

    @Value("${com.liss.producer.group}")
    private String producer_group;

    @Value("${com.liss.namesrvAddr}")
    private String namesrvAddr;

    @Value("${com.liss.topic}")
    private String topic;

    @Value("${com.liss.tag}")
    private String tag;


    //对象初始化就执行 相当于servlet的init
    @PostConstruct
    public void demoProducer(){
        DefaultMQProducer producer = new DefaultMQProducer(producer_group);
        producer.setNamesrvAddr(namesrvAddr);
        producer.setRetryTimesWhenSendFailed(3);
        try{
            producer.start();
            for (int i = 0; i < 50; i++) {
                DemoEvent event = new DemoEvent("lisi"+i, "这是一个message"+i);
                Message message = new Message(topic, tag, event.getName(), event.getMessage().getBytes("UTF-8"));
                SendResult sendResult = producer.send(message);
                System.err.println("生产者在生产："+sendResult.getMsgId());
            }
        }catch (Exception e){
            e.printStackTrace();
            log.error(e.getMessage());
        }finally {
            producer.shutdown();
        }
    }
```

### 消费者
```java
@PropertySource(value = "classpath:recoketMQ.properties")
@Component
@Slf4j
public class DemoConsumer {
    @Value("${com.liss.consumer.group}")
    private String consumer_group;

    @Value("${com.liss.namesrvAddr}")
    private String namesrvAddr;

    @Value("${com.liss.topic}")
    private String topic;

    @Value("${com.liss.tag}")
    private String tag;

    @PostConstruct
    public void demoConsumer(){
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(consumer_group);
        consumer.setNamesrvAddr(namesrvAddr);
        try{
            consumer.subscribe(topic, tag);
            consumer.registerMessageListener(new MessageListenerConcurrently() {
                @Override
                public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                    try {
                        list.forEach(messageExt -> {
                            System.err.println("消费者消息：" + new String(messageExt.getBody()));
                        });
                    }catch (Exception e){
                        log.error(e.getMessage());
                        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                    }
                    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                }
            });
            consumer.start();
        }catch (Exception e){
            e.printStackTrace();
            log.error(e.getMessage());
        }
    }
```
### 启动类
```java
@SpringBootApplication
public class RecoketMQApplication {
    public static void main(String[] args) {
        SpringApplication.run(RecoketMQApplication.class, args);
    }
}
```

### 启动查看
这个加入了@PostConstruct，是在项目一启动，生产者消费者就开始生产和消费。
![](8.png)

上图看，有生产有消费。一个简单的基于原始API的Rocket就搭建完成了。如果实际中，会出现很多重复代码，没啥通用性。

## 自定义封装RocketMQ
###目录结构
![](9.png)
### pom文件
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

    <!-- springboot整合recoketmq -->
    <dependency>
        <groupId>com.alibaba.rocketmq</groupId>
        <artifactId>rocketmq-client</artifactId>
        <version>3.2.6</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.44</version>
    </dependency>
</dependencies>
```
### 配置文件
```xml
# name server的地址
com.liss.rocketMQ.namesrvAddr=192.168.25.7:9876

#生产者组名称
com.liss.rocketMQ.producerGroupName=liss_producer_group

#事务生产组
com.liss.rocketMQ.transactionProducerGroupName=transaction_producer_name

#消费者组
com.liss.rocketMQ.consumerGroupName=liss_consumer_group

#生产者实例名称
com.liss.rocketMQ.producerInstanceName=liss_producer_instance

#消费者实例名称
com.liss.rocketMQ.consumerInstanceName=liss_consumer_instance

#事务生产者实例名称
com.liss.rocketMQ.producerTranInstanceName=liss_producer_transacition

#一次最大消费多少数量消息
com.liss.rocketMQ.consumerBatchMaxSize=1

#广播消费
com.liss.rocketMQ.consumerBroadcasting=false

#消费的topic：tag
com.liss.rocketMQ.subscribe[0]=liss_topic:first

#启动的时候是否消费历史记录
com.liss.rocketMQ.enableHistoryConsumer=false

#启动顺序消费
com.liss.rocketMQ.enableOrderConsumer=false
```
### 读取配置文件
```java
@Configuration
//加载配置文件到RocketPro中
@EnableConfigurationProperties(RocketPro.class)
@Slf4j
public class RocketMQConfiguration {

    @Autowired
    private RocketPro rocketPro;

    @Autowired
    private ApplicationEventPublisher publisher = null;

    /**
     * 初始化打印配置信息
     */
    @PostConstruct
    public void init(){
        System.err.println(rocketPro.getNamesrvAddr());
        System.err.println(rocketPro.getProducerGroupName());
        System.err.println(rocketPro.getConsumerBatchMaxSize());
        System.err.println(rocketPro.getConsumerGroupName());
        System.err.println(rocketPro.getConsumerInstanceName());
        System.err.println(rocketPro.getProducerInstanceName());
        System.err.println(rocketPro.getProducerTranInstanceName());
        System.err.println(rocketPro.getTransactionProducerGroupName());
        System.err.println(rocketPro.isConsumerBroadcasting());
        System.err.println(rocketPro.isEnableHistoryConsumer());
        System.err.println(rocketPro.isEnableOrderConsumer());
        System.out.println(rocketPro.getSubscribe().get(0));
    }

    /**
     * 生产者
     * @return
     * @throws MQClientException
     */
    @Bean
    @Qualifier("defaultMQProducer")
    public DefaultMQProducer defaultMQProducer() throws MQClientException {
        DefaultMQProducer producer = new DefaultMQProducer(rocketPro.getProducerGroupName());
        producer.setNamesrvAddr(rocketPro.getNamesrvAddr());
        producer.setRetryTimesWhenSendFailed(5);
        producer.setInstanceName(rocketPro.getProducerInstanceName());
        producer.start();
        log.info("生产者启动");
        return producer;
    }

    /**
     * 事务生产者
     * @return
     * @throws MQClientException
     */
    @Bean
    @Qualifier("transactionMQProducer")
    public TransactionMQProducer transactionMQProducer() throws MQClientException {
        TransactionMQProducer producer = new TransactionMQProducer(rocketPro.getTransactionProducerGroupName());
        producer.setNamesrvAddr(rocketPro.getNamesrvAddr());
        producer.setRetryTimesWhenSendFailed(5);
        producer.setInstanceName(rocketPro.getProducerTranInstanceName());
        producer.setCheckThreadPoolMinSize(2);
        producer.setCheckThreadPoolMaxSize(10);
        producer.setCheckRequestHoldMax(100);
        //3.2.6版本已经没有了，可以外部通过定时扫描解决, 当为COMMIT_MESSAGE 是消费者才能收到消息
        producer.setTransactionCheckListener(new TransactionCheckListener() {
            @Override
            public LocalTransactionState checkLocalTransactionState(MessageExt messageExt) {
                System.out.println(messageExt.getMsgId());
                return LocalTransactionState.ROLLBACK_MESSAGE;
            }
        });
        producer.start();
        log.info("事务生产者启动");
        return producer;
    }

    /**
     * 消费者
     * @return
     * @throws Exception
     */
    @Bean
    public DefaultMQPushConsumer defaultMQPushConsumer() throws Exception{
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(rocketPro.getConsumerGroupName());
        consumer.setNamesrvAddr(rocketPro.getNamesrvAddr());
        consumer.setConsumerGroup(rocketPro.getConsumerGroupName());
        consumer.setInstanceName(rocketPro.getConsumerInstanceName());
        consumer.setPullBatchSize(rocketPro.getConsumerBatchMaxSize());
        //判断是否是广播消息
        if(rocketPro.isConsumerBroadcasting()){
            consumer.setMessageModel(MessageModel.BROADCASTING);
        }
        //设置批量
        consumer.setConsumeMessageBatchMaxSize(rocketPro.getConsumerBatchMaxSize() == 0?1:rocketPro.getConsumerBatchMaxSize());
        rocketPro.getSubscribe().forEach(str->{
            String[] topic_tag = str.split(":");
            try {
                consumer.subscribe(topic_tag[0], topic_tag[1]);
            } catch (MQClientException e) {
                e.printStackTrace();
                log.error(e.getMessage());
            }
        });
        //顺序消费
        if(rocketPro.isEnableOrderConsumer()){
            consumer.registerMessageListener(new MessageListenerOrderly() {
                @Override
                public ConsumeOrderlyStatus consumeMessage(List<MessageExt> list, ConsumeOrderlyContext consumeOrderlyContext) {
                    consumeOrderlyContext.setAutoCommit(true);
                    try {
                        publisher.publishEvent(new MessageEvent(list, consumer));
                    } catch (Exception e) {
                        e.printStackTrace();
                        return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                    }
                    return ConsumeOrderlyStatus.SUCCESS;
                }
            });
        }else{
            consumer.registerMessageListener(new MessageListenerConcurrently() {
                @Override
                public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                    try {
                        //用spring的监听，将消息放到监听中
                        publisher.publishEvent(new MessageEvent(list, consumer));
                    } catch (Exception e) {
                        e.printStackTrace();
                        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                    }
                    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                }
            });
        }
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    consumer.start();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();
        return consumer;
    }
}

```
### 自定义消息
```java
public class MessageEvent extends ApplicationEvent {

    private DefaultMQPushConsumer consumer;
    private List<MessageExt> msgs;

    public MessageEvent(List<MessageExt> msgs, DefaultMQPushConsumer consumer) throws Exception {
        super(msgs);
        this.consumer = consumer;
        this.setMsgs(msgs);
    }
    public DefaultMQPushConsumer getConsumer() {
        return consumer;
    }

    public void setConsumer(DefaultMQPushConsumer consumer) {
        this.consumer = consumer;
    }

    public List<MessageExt> getMsgs() {
        return msgs;
    }

    public void setMsgs(List<MessageExt> msgs) {
        this.msgs = msgs;
    }
}
```
### 接受消息
```
@Component
public class ConsumerServiceAnnoTion {
    @EventListener(condition = "#event.msgs[0].topic=='liss_topic' && #event.msgs[0].tags=='first'")
    public void listenerEvent(MessageEvent event){
        event.getMsgs().forEach(messageExt -> {
            System.err.println("消费者的消息：" + new String(messageExt.getBody()));
        });
    }
}
```

### 创建消息发送
```
@RestController
public class ProducerController {

    @Autowired
    @Qualifier("defaultMQProducer")
    private DefaultMQProducer defaultMQProducer;

    @Autowired
    @Qualifier("transactionMQProducer")
    private TransactionMQProducer transactionMQProducer;

    @RequestMapping("/sendMessage")
    public void sendMessage() throws Exception {
        for (int i = 0; i < 20; i++) {
            Message message = new Message("liss_topic", "first", ("message " + i).getBytes());
            SendResult send = defaultMQProducer.send(message);
            System.out.println("发送状态："+send.getSendStatus());
        }
        defaultMQProducer.shutdown();
    }

    @RequestMapping("/sendTransactionMess")
    public void sendTransactionMess() throws Exception {
        SendResult sendResult = null;
        try {
            // a,b,c三个值对应三个不同的状态
            String ms = "c";
            Message msg = new Message("liss_topic","first",ms.getBytes());
            // 发送事务消息
            sendResult = transactionMQProducer.sendMessageInTransaction(msg, (Message msg1, Object arg) -> {
                String value = "";
                if (arg instanceof String) {
                    value = (String) arg;
                }
                if (value == "") {
                    throw new RuntimeException("发送消息不能为空...");
                } else if (value =="a") {
                    return LocalTransactionState.ROLLBACK_MESSAGE;
                } else if (value =="b") {
                    return LocalTransactionState.COMMIT_MESSAGE;
                }
                return LocalTransactionState.ROLLBACK_MESSAGE;
            }, 4);
            System.out.println("事务生生产者状态：" + sendResult.getSendStatus());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @RequestMapping("/orderSendMessage")
    public void orderSendMessage(){
        for(int i=0;i<100;i++) {
            Message message = new Message("liss_topic", "first", ("order_message"+i).getBytes());
            try {
                SendResult sendResult = defaultMQProducer.send(message, new MessageQueueSelector() {
                    @Override
                    public MessageQueue select(List<MessageQueue> list, Message message, Object o) {
                        int index = ((Integer) o) % list.size();
                        return list.get(index);
                    }
                }, i);
                System.out.println("生产的消息："+sendResult.getSendStatus());
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```
### 启动类

```java
@SpringBootApplication
@ComponentScan(basePackages = {"com.liss"})
public class RocketApplication {
    public static void main(String[] args) {
        SpringApplication.run(RocketApplication.class, args);
    }
}
```

## 测试

### 发送普通消息
![](10.png)

### 发送order消息
![](11.png)

### 发送事务消息
![](12.png)

到此，生产者和消费者都可以发送和接受消息了。