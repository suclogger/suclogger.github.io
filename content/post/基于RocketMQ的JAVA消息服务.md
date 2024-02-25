---
title: 基于RocketMQ的JAVA消息服务
comments: true
toc: true
categories:
  - 编程
tags:
  - java
  - RocketMQ
date: 2017-08-10 11:07:33
---
基于RocketMQ的JAVA消息服务
<!-- more -->

## Java消息服务（ JMS）基础

>Java消息服务（Java Message Service，JMS）应用程序接口是一个Java平台中关于面向消息中间件（MOM）的API，用于在两个应用程序之间，或分布式系统中发送消息，进行异步通信。 Java消息服务是一个与具体平台无关的API，绝大多数MOM提供商都对JMS提供支持。
Java消息服务- 维基百科，自由的百科全书
https://zh.wikipedia.org/zh-hans/Java消息服务

### 异构平台的通信和集成
使用消息传送机制，可以向在完全不同的平台上实现的应用程序和系统请求调用服务。

![](/image/2017-08-10-11-34-18.jpg)

### 提升吞吐量，缓解系统瓶颈
与一个同步组件处理众多请求时，众多请求一个接一个地积聚阻塞不同，这时候请求会发送到一个消息传送系统， 该系统将该请求分发给多个消息侦听器组件。如此一来，就缓解了单独采用点对点同步连接带来的系统瓶颈，在某些情况下，甚至可以完全消除这些瓶颈。

![](/image/2017-08-10-11-38-27.jpg)

### 异步
当执行长时间运行请求时，最终用户可以在系统上继续做其他工作。一旦该请求处理完毕，就立即告知最终用户，并将处理结果回传给最终用户。通过使用消息传送机制，最终用户就能够以更短的等待时间来完成更多的工作，使得最终用户拥有更高的生产率。

### 敏捷和解耦
体系结构敏捷性是对不断变化的环境快速响应的能力。通过使用消息传送机制来抽象和去 耦组件，就能够快速地响应软件、硬件，甚至是业务的变化。


## 术语
### 通用术语
* **Message**： 消息，包含消息头和消息体。`A message basically has two parts: a header and a payload. The header is comprised of special fields that are used to identify the message, declare attributes of the message, and provide information for routing.`
* **MessageProducer** ： 生产者，`The MessageProducer is the base interface for the TopicPublisher and the QueueSender.`
* **MessageConsumer**： 消费者， `The MessageConsumer is the base interface for the TopicSubscriber and the QueueReceiver.`
* **Topic**： Topic用于标识消息服务上的一个物理主题，主题是消息发布和订阅的通道，`The Topic is an administered object that acts as a handle or identifier for an actual topic, called a physical topic, on the messaging server. A physical topic is a channel to which many clients can subscribe and publish. When a JMS client delivers a Message object to a topic, all the clients subscribed to that topic receive the Message.`
* **Queue**：Queue用于标识消息服务上的一个物理队列，物理队列是消息发送和接收的通道，`The Queue is an administered object that acts as a handle or identifier for an actual queue, called a physical queue , on the messaging server. A physical queue is a channel through which many clients can receive and send messages.`

### RocketMQ概念术语

* **Producer Group**：标识发送同一类消息的Producer，通常发送逻辑一致。发送普通消息的时候，仅标识使用，并无特别用处
* **Consumer Group**：标识一类Consumer的集合名称，这类Consumer通常消费一类消息，且消费逻辑一致。同一个Consumer Group下的各个实例将共同消费topic的消息，起到负载均衡的作用。RocketMQ要求同一个Consumer Group的消费者必须要拥有相同的注册信息，即必须要听一样的topic(并且tag也一样)。
* **Tag**：消息的二级分类，同一个topic的消息虽然逻辑管理是一样的。但是消费topic1的时候，如果你订阅的时候指定的是tagA，那么tagB的消息将不会投递。
* **Offset**：消息的消费进度。RocketMQ中，有很多offset的概念。但通常我们只关心暴露到客户端的offset。一般我们不特指的话，就是指逻辑Message Queue下面的offset。消费进度以Consumer Group为粒度管理，不同Consumer Group之间消费进度彼此不受影响，即消息A被Consumer Group1消费过，也会再给Consumer Group2消费。每次消息消费成功后，这个offset在会先更新到内存，而后定时持久化。在集群消费模式下，会同步持久化到broker，而在广播模式下，则会持久化到本地文件。

### 结构术语
* **Broker**：消息中转角色，负责存储消息，转发消息，一般也称为 Server。在 JMS 规范中称为 Provider。`A message server, also called a message router or broker, is responsible for delivering messages from one messaging client to other messaging clients.`
* **NameServer**：寻址服务。用于把Broker的路由信息做聚合。客户端依靠Name Server决定去获取对应topic的路由信息，从而决定对哪些Broker做连接。


逻辑结构：
![](/image/2017-08-10-12-53-31.jpg)

物理部署结构：
![](/image/2017-08-10-12-54-59.jpg)

### 消费模式
* **集群消费CLUSTERING**：一个Consumer Group中的各个Consumer实例分摊去消费消息，即一条消息只会投递到一个Consumer Group下面的一个实例，每个Consumer是平均分摊Message Queue的去做拉取消费。
* **广播消费BROADCASTING**：消息将对一个Consumer Group下的各个Consumer实例都投递一遍。即即使这些 Consumer 属于同一个Consumer Group，消息也会被Consumer Group 中的每个Consumer都消费一次，一个消费组下的每个消费者实例都获取到了topic下面的每个Message Queue去拉取消费。

### 消息发送模式
* **同步阻塞**：消息到达broker之后才返回，对应`synSend`。
* **异步回调**：消息发送结束后触发回调，对应`asynSend`。
* **发送即忘**：只管发送，不管是否到达broker，对应`sendOneWay`。

## 使用场景
### 普通消息

Producer：

```
val message =  DemoMessage("suclogger", "plain_message", "syncSend")
demoProducer.synSend("suclogger", "", message)
```

Consumer:

```
@MQConsumer(consumerGroup = "camaro.mq.consumer", topic = "suclogger")
class DemoConsumer : AbstractMQPushConsumer<DemoMessage>() {

    override fun process(message:DemoMessage) : Boolean {
        println(message)
        return true
    }
}

@MQConsumer(consumerGroup = "consumer_withtag_suclogger_local", topic = "suclogger", tag = arrayOf("tagA", "tagB"))
class DemoConsumerWithTag : AbstractMQPushConsumer<DemoMessage>() {

    override fun process(message:DemoMessage) : Boolean {
        println(message)
        return true
    }
}
```

#### 消息消费失败后重试

消息消费返回false触发重试：

```
@MQConsumer(consumerGroup = "camaro.mq.consumer", topic = "suclogger")
class DemoConsumer : AbstractMQPushConsumer<DemoMessage>() {

    override fun process(message:DemoMessage) : Boolean {
        println(message)
        return false
    }
}
```

### 广播消息
Producer：
```
    @Bean
    fun broadcastMessage() = CommandLineRunner {
        val message =  DemoMessage("suclogger", "broadcast_message", "oneway")
        demoProducer.sendOneWay("suclogger-broadcast", "", message)
    }
```
Consumer：
```
@MQConsumer(topic = "suclogger-broadcast", consumerGroup = "consumer_broadcast_suclogger_local_b1", messageMode = "BROADCASTING")
class AnonmouseMessageConsumer : AbstractMQPushConsumer<Any>() {
    override fun process(message: Any?): Boolean {
        println(message.toString())
        return true
    }
}

@MQConsumer(topic = "suclogger-broadcast", consumerGroup = "consumer_broadcast_suclogger_local_b2", messageMode = "BROADCASTING")
class AnonmouseMessageConsumerOther : AbstractMQPushConsumer<Any>() {
    override fun process(message: Any?): Boolean {
        println(message.toString())
        return true
    }
}
```

### 有序消息
最简单的方式：创建Topic的时候只选择包含一个queue。
如果要增加吞吐量：发送消息的时候选择queue来实现部分有序。

Producer:

```
    @Bean
    fun orderMessage() = CommandLineRunner {
        demoProducer.sendOneWayOrderly("suclogger", "", "this is for 1", "1")
        demoProducer.sendOneWayOrderly("suclogger", "", "this is for 2", "2")
        demoProducer.sendOneWayOrderly("suclogger", "", "this is for 3", "3")
        demoProducer.sendOneWayOrderly("suclogger", "", "this is for 4", "4")
    }
```

Consumer:

```
@MQConsumer(consumerGroup = "consumer_order_suclogger_local", topic = "suclogger", consumeMode = "ORDERLY")
class DemoConsumerOther : AbstractMQPushConsumer<Any>() {
    override fun process(message:Any) : Boolean {
        println(message)
        return true
    }
}
```

### 事务消息
基于两阶段提交的思想：[分布式系统设计迷思](http://suclogger.me/分布式系统设计迷思/)


![](/image/2017-08-10-14-23-21.jpg)

* 事务回查逻辑不完整
* 下游事务不能失败

预期发布：[issur](https://issues.apache.org/jira/browse/ROCKETMQ-123)


## 部署和推荐姿势

[部署和推荐姿势](http://suclogger.me/RocketMQ/)






