---
title: RocketMQ
comments: true
toc: true
categories:
  - 编程
tags:
  - rocketmq
date: 2017-07-03 22:10:45
---
介绍一下近期做的RocketMQ接入的工作。
<!-- more -->
## docker
官方在external中提供了namesrv和brokr的标准镜像：[rocketmq-docker](http://https://github.com/apache/incubator-rocketmq-externals/tree/master/rocketmq-docker)
但是标准镜像的jvm启动参数是写死在`runserver.sh`和`runbroker.sh`中的：

```
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:PermSize=128m -XX:MaxPermSize=320m"
```

定义在`JAVA_OPT`中会被后续的`-Xms4g -Xmx4g -Xmn2g`覆盖。所以需要修改一下Dockfile，使用`sed`命令替换：

```
sed -i 's/-Xm[sxn]..//g' runbroker.sh \
```

因为是集群部署，还需要写入相关配置并将brokerId和brokerName透到参数中，以下为双主集群配置，其他配置同理：

```
        && echo -e "brokerClusddrName=camaro\nbrokerId=0\ndeleddWhen=04\nfileReservedTime=48\nbrokerRole=ASYNC_MASTER\nflushDiskType=ASYNC_FLUSH" > broker.properties \
        && echo -e "\nbrokerName=$BROKER_NAME" >> broker.properties \
        && echo -e "\nbrokerIP1=$BROKER_IP" >> broker.properties \
```

如果需要自动创建Topic，还需要添加启动参数：

```
autoCreateTopicEnable=true
```

可以使用compose组装nameserver和broker：

```
version: '2'
services:
  rocketmq-namesrv:
    image: daocloud.io/maihaoche/rocketmq-namesrv:4.0.1
    restart: always
    network_mode: host
    ports:
    - 9876:9876
    volumes:
    - /root/logs:/opt/logs
    - /root/store:/opt/store
  rocketmq-broker:
    image: daocloud.io/maihaoche/rocketmq-broker:4.0.1
    restart: always
    network_mode: host
    ports:
    - 10911:10911
    - 10909:10909
    volumes:
    - /root/logs:/opt/logs
    - /root/store:/opt/store
    environment:
    - BROKER_IP=172.21.10.101
    - NAMESRV_ADDR=172.21.10.101:9876
    - BROKER_NAME=broker-101

```

不同的机器可以通过修改compose文件中的`environment`参数进行部署。
务必配置网络类型为host，否则会导致nameserver返回broker的docker0对应IP使客户端无法连接。

## benchmark
Broker和namesrv的jvm参数配置为`-Xms2g -Xmx2g -Xmn1g`，两台双主集群下跑benchmark。
benchmark使用consumer和provider的jvm参数：
```
-server -Xms1g -Xmx1g -Xmn256m -XX:PermSize=128m -XX:MaxPermSize=320m -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8 -XX:+DisableExplicitGC -verbose:gc -Xloggc:/dev/shm/rmq_srv_gc.log -XX:+PrintGCDetails -XX:-OmitStackTraceInFastThrow -XX:-UseLargePages -XX:+PerfDisableSharedMem
```
producer 64线程，消息体10K，TPS峰值为5,000左右：
![](/image/benchmark-producer.png)

consumer 64 线程，消息体10K，TPS峰值为28,000：
![](/image/benchmark-consumer.png)

## starter
代码已经开源：[github](https://github.com/maihaoche/rocketmq-spring-boot-starter)
推荐姿势：
### producer 



#### pruducer
我们将producer设计成了单例，你可以全局只继承一次AbstractMQProducer，然后用这个producer发送不同topic消息：

```
AbstractMQProducer.synSend(String topic, Object msgObj)
```

当然，你可以定义多个类继承`AbstractMQProducer`，每个类都重写父类中的`getTopic()`方法，这样注入对应的bean就可以省去topic直接发送：

```
AbstractMQProducer.sendMessage(Object msgObj)
```


#### pruducerGroup
因为producer的单例设计，我们推荐同一个应用所有消息采用同一个pruducerGroup，放在配置文件中。

```
camaro:
  mq:
    name-server-address: 172.21.10.101:9876;172.21.10.111:9876
    producer-group: local_pufang_producer
```

### consumer

不同consumerGroup都有自己的topic消费位点记录，所以同一条消息在不同consumerGroup中都可以被消费一次，相应的，同一条消息在同一个consumerGroup中只能消费一次。

在某些极端情况下，同一条消息在同一个consumerGroup中会被多次消费，所以消费代码要做到幂等。

## console

我们还基于开源的 [console](https://github.com/apache/incubator-rocketmq-externals/tree/master/rocketmq-console) 项目做了一层kotlin封装，展示效果采用[muu](https://github.com/maihaoche/muu)：

![](/image/2017-07-04-00-24-34.jpg)

## why
到最后说几句为什么做这件事情。

### why not ONS
平时在用ONS的过程中都遇到过难以调试的困扰，消息不知道发了没有，消息不知道被谁消费了等等问题，归根结底主要有以下原因：
1. 由于我们采用的都是公网的ons，消息服务之间无法做到物理隔离;
2. 问题排查，topic维护需要繁琐的子账号授权，大量的人肉运维；
3. ons是根据topic来计费，我们单topic的消息量是很少的，使用ons非常的不经济。


### why RocketMQ
直接促成我们使用RocketMQ的是上上周的OSC上RocketMQ的宣讲，主要有以下优点：
1. 国产
2. 支持消息追溯
3. 支持广播
4. 有较好的console和运维终端
5. 可以比较好的保留我们现有的ons使用习惯




