---
title: rocketMQ的分布式事务一致性
date: 2018-07-31 23:59:47
urlname: RocketMQ-transaction
tags:
  - RocketMQ
categories: RocketMQ
---

## 前言
在分布式事务中，通常有两种方式保持事务的一致性。
一种是通过MQ发送和接受消息来通知下一步的操作。
一种是通过上游执行完之后，将状态保存到外部存储中，系统定时扫描根据状态来决定下游的操作。
两种方法有利有弊，第一种如果消费失败，消息就没了，第二种频繁的扫描，对系统的性能是个影响，并且百分之九十是在做无用功。如果存在外部数据库中，性能也是个瓶颈。

在众多的MQ产品中，貌似只有rocketMQ是提供支持的。

## rocket的分布式事务

RocketMQ在 V3.1.5 开始，使用 数据库 实现【事务状态】的存储。但未开源，因为rocketmq阉割了对生产者的LocalTransactionState状态的回查机制，所以增加了生产端事务的复杂度。本来由RocketMQ中间件通过回查机制来让生产者知道事务信息发送成功，现在要生产者自己来确认。
<!--more-->

### rocketMQ的分布式事务原理
![](1.png)

因为rocketmq阉割了对LocalTransactionState状态的回查机制，所以生产者必须确认rocketMQ集群是否收到LocalTransactionState状态；需要自己动手实现一个回查。

思路：
1、在执行本地事务commit前向回查表插入消息的KEY值  key,status,count
2、设置一个定时任务
 2.1 从回查表中取出状态为未确认的记录。
 2.2 判断回查次数count的次数是否等于指定的次数，如果等于指定的次数，则说明消息已经失败，并且回查了指定的次数，这时候需要根据自身的业务，是同步重发此消息，还是通知人工来解决。如果count小于指定的次数，说明此消息并没有达到回查的阈值，这时候去MQ集群中查找该消息。
 2.3 查找到该消息之后，判断是否已经消费过，没有消费的话，修改回查表的count+1，如果该消息已经消费，修改回查表的status和count值。

### 消费者集群事务
1、消费者在执行的时候，会出现重复消费的情况，在外部建立去重表，消费的时候判断是否已经消费，如果已经消费，就忽略。
2、对于事务执行失败的情况，rocketMQ给的方案是 **人工干预**，如果一个系统中，如果牵扯到太多的话，回滚事务也是一个大工程，而且出现BUG的几率比消费失败大的多。没有必要去花大代价去解决一件出现概率极低的时间。

### rocketMQ4.3

在rocketMQ4.3版本中，重新支持了事务的回查。
#### 使用限制
1.没有时间表和批量。
2.为了避免单个消息被多次检查并导致半队列消息累积，单个消息的检查次数限制为15次，但是用户可以通过更改“transactionCheckMax”来更改此限制“代理配置中的参数，如果已经通过”transactionCheckMax“检查了一条消息，则代理将默认丢弃此消息并同时打印错误日志。可以通过覆盖“AbstractTransactionCheckListener”类来更改此行为。
3.在broker的配置中由参数“transactionTimeout”确定的一段时间之后将检查交易消息。也可以通过在发送事务消息时设置用户属性“CHECK_IMMUNITY_TIME_IN_SECONDS”来更改此限制，此参数优先于“transactionMsgTimeout”参数。
4.可以多次检查或消费交易消息。
5.对目标主题的已提交消息可能会失败。目前，它取决于日志记录。RocketMQ本身的高可用性机制确保了高可用性。如果要确保事务性消息不会丢失且事务完整性得到保证，建议使用同步双写。机制。
事务消息的生产者ID不能与其他类型消息的生产者ID共享。与其他类型的消息不同，事务性消息允许后向查询。
#### 事务状态
TransactionStatus.CommitTransaction: 提交事务，这意味着允许消费者使用此消息。
TransactionStatus.RollbackTransaction: 回滚事务，表示该消息将被删除而不允许使用。
TransactionStatus.Unknown: 中间状态，表示需要MQ检查以确定状态。
#### 生产者实例
```java
private static ConcurrentHashMap<String, Integer> localTrans = new ConcurrentHashMap<>();
TransactionMQProducer producer = new TransactionMQProducer("tran_group");
    producer.setNamesrvAddr("192.168.25.7:9876");
    producer.setTransactionListener(new TransactionListener() {
        //执行本地事务
        @Override
        public LocalTransactionState executeLocalTransaction(Message message, Object o) {
            //将message写入消息表或者其他存储中，并保证有唯一标示
            localTrans.put(message.getTransactionId(),"唯一标示");
            return LocalTransactionState.UNKNOW;
        }
        //事务回查 三种状态 UNKNOW COMMIT_MESSAGE ROLLBACK_MESSAGE
        @Override
        public LocalTransactionState checkLocalTransaction(MessageExt messageExt) {
            Integer status = localTrans.get(messageExt.getTransactionId());
            //根据本地事务执行的状态，决定是发给消费者还是不通知消费者，或者回滚
            switch (status){
                case 0:
                    return LocalTransactionState.UNKNOW;
                case 1:
                    return LocalTransactionState.COMMIT_MESSAGE;
                case 2:
                    return LocalTransactionState.ROLLBACK_MESSAGE;
            }
            return LocalTransactionState.COMMIT_MESSAGE;
        }
    });
    producer.start();
    producer.sendMessageInTransaction(new Message("tran_topic","tran","ceshi".getBytes()), null);

}
```