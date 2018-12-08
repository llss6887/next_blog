---
title: RocketMQ 的使用
date: 2017-06-16 23:59:47
urlname: RocketMQ
tags:
  - RocketMQ
categories: RocketMQ
---
## 为什么选择RcoketMQ

支持严格的消息顺序
支持Topic与Queue两种模式
亿级消息堆积能力
比较友好的分布式特性
同时支持Push与Pull方式消费消息
历经多次天猫双十一海量消息考验
RocketMQ是纯java编写，基于通信框架Netty。


## 核心概念

它由四部分组成：name servers, brokers, producers 和 consumers。它们中的每一个都可以水平扩展而没有单一的故障点。如下面的截图所示。

![](rmq-basic-arc.png)
<!--more-->
### Producer（生产者）
生产者将业务应用程序系统生成的消息发送给代理。RocketMQ提供多种发送范例：同步，异步和单向。

### Producer Group（生产组）
具有相同角色的生产者组合在一起。如果原始生产者在事务之后崩溃，则代理可以联系同一生产者组的不同生产者实例以提交或回滚事务。

***注意：考虑到提供的生产者在发送消息方面足够强大，每个生产者组只允许一个实例，以避免不必要的生成器实例初始化。***

### Consumer（消费者）
消费者从Broker那里获取消息并将其提供给应用程序。从用户应用的角度来看，提供了两种类型的消费者：

#### PullConsumer（主动拉取得消费者）
拉动消费者积极地从Broker那里获取消息。一旦提取了批量消息，用户应用程序就会启动消费过程。

#### PushConsumer（通过监听获取消息的消费者）
推送消费者封装消息提取，消费进度并维护其他内部工作，为最终用户留下回调接口以实现将在消息到达时执行。

### Consumer Group（消费者组）
与之前提到的生产者组类似，完全相同角色的消费者被组合在一起并命名为消费者组。
消费者群体是一个很好的概念，在消息消费方面实现负载平衡和容错目标非常容易。

***注意：使用者组的使用者实例必须具有完全相同的主题订阅。***

### Topic（主题）
生产者传递消息和消费者提取消息的类别。主题与生产者和消费者的关系非常松散。具体来说，一个主题可能有零个，一个或多个生成器向它发送消息; 相反，Broker可以发送不同主题的消息。从消费者的角度来看，主题可以由零个，一个或多个消费者群体订阅。类似地，消费者组可以订阅一个或多个主题，只要该组的实例保持其订阅一致即可。

### Message（消息）
消息是要传递的信息。消息必须有一个主题，可以将其解释为您要发送给的邮件地址。消息还可以具有可选标记和额外的键 - 值对。

### Message Queue（消息队列）
主题被划分为一个或多个子主题“消息队列”。

### Tag(标签)
标记，换句话说，子主题，为用户提供了额外的灵活性。对于标记，来自同一业务模块的具有不同目的的消息可以具有相同的主题和不同的标记。标签有助于保持代码的清晰和连贯，而标签也可以方便RocketMQ提供的查询系统。

### Broker
RocketMQ系统的主要组成部分。它接收从生产者发送的消息，存储它们并准备处理来自消费者的拉取请求。它还存储与消息相关的元数据，包括消费者组，消耗进度偏移和主题/队列信息。

### Name Server
充当路由信息提供者。生产者/消费者客户查找主题以查找相应的Broker列表。相当于kafka中zookeeper的作用。

## 部署方式
### 单Master模式
只有一个 Master节点
优点：配置简单，方便部署
缺点：这种方式风险较大，一旦Broker重启或者宕机时，会导致整个服务不可用，不建议线上环境使用

### 多Master模式
一个集群无 Slave，全是 Master，例如 2 个 Master 或者 3 个 Master
优点：配置简单，单个Master 宕机或重启维护对应用无影响，在磁盘配置为RAID10 时，即使机器宕机不可恢复情况下，由与 RAID10磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢）。性能最高。多 Master 多 Slave 模式，异步复制
缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到受到影响

### 多Master多Slave模式（异步复制）
每个 Master 配置一个 Slave，有多对Master-Slave， HA，采用异步复制方式，主备有短暂消息延迟，毫秒级。
优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，因为Master 宕机后，消费者仍然可以从 Slave消费，此过程对应用透明。不需要人工干预。性能同多 Master 模式几乎一样。
缺点： Master 宕机，磁盘损坏情况，会丢失少量消息。

### 多Master多Slave模式（同步双写）
每个 Master 配置一个 Slave，有多对Master-Slave， HA采用同步双写方式，主备都写成功，向应用返回成功。
优点：数据与服务都无单点， Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高
缺点：性能比异步复制模式略低，大约低 10%左右，发送单个消息的 RT会略高。目前主宕机后，备机不能自动切换为主机，后续会支持自动切换功能

如果无法容忍消息丢失，建议部署SYNC_MASTER并为其附加SLAVE。如果对丢失感到满意，但希望Broker始终可用，则可以使用SLAVE部署ASYNC_MASTER。如果只是想让它变得简单，可能只需要一个没有SLAVE的ASYNC_MASTER。

这些部署方式。在官方的包中都有默认配置。

## 实例：

使用RocketMQ以三种方式发送消息：可靠的同步，可靠的异步和单向传输

### maven依赖
```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>3.2.6</version>
</dependency>
```
### 同步消息
主要用在重要的通知消息，短信通知，短信营销系统等。
```java
public class SyncProducer {
    public static void main(String[] args) {
        //创建生产者，生产者组需要写
        DefaultMQProducer producer = new DefaultMQProducer("first_producer_group");
        //设置nameserver，多个用逗号分隔
        producer.setNamesrvAddr("localhost:7986,localhost:9875");
        for (int i = 0; i <10 ; i++) {
            try {
                //创建消息，
                // public Message(String topic, byte[] body)
                // public Message(String topic, String tags, byte[] body)
                // public Message(String topic, String tags, String keys, byte[] body)
                //public Message(String topic, String tags, String keys, int flag, byte[] body, boolean waitStoreMsgOK)
                Message message = new Message("test_topic", "*", ("RockMQ" + i).getBytes("UTF-8"));
                SendResult result = producer.send(message);
                System.out.println(result);
            }catch (Exception e){
                e.printStackTrace();
            }
        }
        //关闭
        producer.shutdown();
    }
}
```

### 异步消息

```java
public class AsyncProducer {
    public static void main(String[] args) {
        //创建生产者，生产者组需要写
        DefaultMQProducer producer = new DefaultMQProducer("first_producer_group");
        producer.setNamesrvAddr("localhost:7986,localhost:9875");
        //重试
        producer.setRetryTimesWhenSendFailed(0);
        try {
        producer.start();
            for (int i = 0; i <10 ; i++) {
                Message message = new Message("test_topic", "*", ("RockMQ" + i).getBytes("UTF-8"));
                producer.send(message, new SendCallback() {
                    //成功
                    @Override
                    public void onSuccess(SendResult sendResult) {
                        System.out.println(sendResult.getMsgId());
                    }
                    //失败
                    @Override
                    public void onException(Throwable throwable) {
                        throwable.printStackTrace();
                    }
                });
            }
        }catch (Exception e){
            e.printStackTrace();
        }
        //关闭
        producer.shutdown();
    }
}
```
### 单向消息，没有返回

```java
public class OneWayProducer {
    public static void main(String[] args) {
        //创建生产者，生产者组需要写
        DefaultMQProducer producer = new DefaultMQProducer("first_producer_group");
        //设置nameserver，多个用逗号分隔
        producer.setNamesrvAddr("localhost:7986,localhost:9875");
        for (int i = 0; i <10 ; i++) {
            try {
                //创建消息，
               Message message = new Message("test_topic", "*", ("RockMQ" + i).getBytes("UTF-8"));
                producer.sendOneway(message);
            }catch (Exception e){
                e.printStackTrace();
            }
        }
        //关闭
        producer.shutdown();
    }
}
```
### 消费者
```java
public class Consumer {
    public static void main(String[] args) throws MQClientException {
        //创建消费者
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_group");
        //第一个参数  为topic的名称，第二个为 标签，*代表所有
        consumer.subscribe("test_topic","*");
        //注册监听
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                //消费消息
                for (MessageExt messageExt : list) {
                    System.out.printf(String.valueOf(messageExt.getBody()));
                }
                //返回消费状态
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        //启动
        consumer.start();
    }
}
```

### 顺序消息

#### 全局排序
全局排序的时候，只要设置只有一个topic

#### 分区排序
一个订单的顺序流程是：创建、付款、推送、完成。订单号相同的消息会被先后发送到同一个队列中，消费时，同一个OrderId获取到的肯定是同一个队列。

```java
private static void orderProduce() throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("order_produce");
        producer.setNamesrvAddr("localhost:7986,localhost:9875");
        producer.start();
        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 100; i++) {
            //模拟订单ID
            int orderId = i % 10;
            Message msg = new Message("TopicTestjjj",tags[i % tags.length], "KEY" + i,
                    ("Hello RocketMQ " + i).getBytes("UTF-8"));
            SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
                @Override
                public MessageQueue select(List<MessageQueue> list, Message message, Object o) {
                    Integer id = (Integer)o;
                    int index = id % list.size();
                    return list.get(index);
                }
            }, orderId);
        }
    }
```
使用这种方式MessageListenerConcurrently无法保证有序的，需要用MessageListenerOrderly

```java

private static void orderConsumer() throws Exception {
        //创建消费者
        final DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("order_consumer");
        //第一个参数  为topic的名称，第二个为 标签，取出TagA和TagB
        consumer.subscribe("order_topic","TagA || TagB");
		consumer.setNamesrvAddr("localhost:7986,localhost:9875");
        consumer.registerMessageListener(new MessageListenerOrderly() {
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> list, ConsumeOrderlyContext consumeOrderlyContext) {
                consumeOrderlyContext.setAutoCommit(false);
                for (MessageExt messageExt : list) {
                    System.out.printf(String.valueOf(messageExt.getBody()));
                }
                //返回消费状态
                return ConsumeOrderlyStatus.SUCCESS;
            }
        });
    }
```
### 广播模式
广播模式中，生产者没有变化。消费者代码如下：
```java
public static void main(String[] args) throws MQClientException {
        //创建消费者
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_group");
        consumer.setNamesrvAddr("localhost:7986,localhost:9875");
        //第一个参数  为topic的名称，第二个为 标签，*代表所有
        consumer.subscribe("test_topic","*");
        //设置从什么地方开始
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        //设置广播方式
        consumer.setMessageModel(MessageModel.BROADCASTING);
        //注册监听
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                //消费消息
                for (MessageExt messageExt : list) {
                    System.out.printf(String.valueOf(messageExt.getBody()));
                }
                //返回消费状态
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        //启动
        consumer.start();
    }
```
### 延时消息
延时消息，消费者没变化，生产者代码如下：
```java
public class ScheduledMessage {
    public static void main(String[] args) {
        //创建生产者，生产者组需要写
        DefaultMQProducer producer = new DefaultMQProducer("first_producer_group");
        //设置nameserver，多个用逗号分隔
        producer.setNamesrvAddr("localhost:7986,localhost:9875");
        for (int i = 0; i <10 ; i++) {
            try {
                Message message = new Message("test_topic", "*", ("RockMQ" + i).getBytes("UTF-8"));、
                //这里设置  1代表延迟1s 2代表延迟5秒，3代表延迟10s 以此类推
                message.setDelayTimeLevel(3);
                SendResult result = producer.send(message);
                System.out.println(result);
            }catch (Exception e){
                e.printStackTrace();
            }
        }
        //关闭
        producer.shutdown();
    }
}
```
### 批量发送

如果一次发送的不超过1M，代码如下：
```java
String topic = "BatchTest";
List<Message> messages = new ArrayList<>();
messages.add(new Message(topic, "TagA", "OrderID001", "Hello world 0".getBytes()));
messages.add(new Message(topic, "TagA", "OrderID002", "Hello world 1".getBytes()));
messages.add(new Message(topic, "TagA", "OrderID003", "Hello world 2".getBytes()));
try {
    producer.send(messages);
} catch (Exception e) {
    e.printStackTrace();
}
```
只有在发送大批量时，不确定它是否超出了大小限制,此时拆分列表，官方demo如下：
```java
public class ListSplitter implements Iterator<List<Message>> {
    private final int SIZE_LIMIT = 1000 * 1000;
    private final List<Message> messages;
    private int currIndex;
    public ListSplitter(List<Message> messages) {
            this.messages = messages;
    }
    @Override public boolean hasNext() {
        return currIndex < messages.size();
    }
    @Override public List<Message> next() {
        int nextIndex = currIndex;
        int totalSize = 0;
        for (; nextIndex < messages.size(); nextIndex++) {
            Message message = messages.get(nextIndex);
            int tmpSize = message.getTopic().length() + message.getBody().length;
            Map<String, String> properties = message.getProperties();
            for (Map.Entry<String, String> entry : properties.entrySet()) {
                tmpSize += entry.getKey().length() + entry.getValue().length();
            }
            tmpSize = tmpSize + 20; //for log overhead
            if (tmpSize > SIZE_LIMIT) {
                //it is unexpected that single message exceeds the SIZE_LIMIT
                //here just let it go, otherwise it will block the splitting process
                if (nextIndex - currIndex == 0) {
                   //if the next sublist has no element, add this one and then break, otherwise just break
                   nextIndex++;  
                }
                break;
            }
            if (tmpSize + totalSize > SIZE_LIMIT) {
                break;
            } else {
                totalSize += tmpSize;
            }
    
        }
        List<Message> subList = messages.subList(currIndex, nextIndex);
        currIndex = nextIndex;
        return subList;
    }
}
//then you could split the large list into small ones:
ListSplitter splitter = new ListSplitter(messages);
while (splitter.hasNext()) {
   try {
       List<Message>  listItem = splitter.next();
       producer.send(listItem);
   } catch (Exception e) {
       e.printStackTrace();
       //handle the error
   }
}
```
### 过滤器
在大多数情况下，tag是一种简单而有用的设计，用于选择所需的消息。例如：
```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("CID_EXAMPLE");
consumer.subscribe("TOPIC", "TAGA || TAGB || TAGC");
```
