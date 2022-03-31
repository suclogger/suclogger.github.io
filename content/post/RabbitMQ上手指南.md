---
title: RabbitMQ上手指南
comments: true
toc: true
categories:
  - 编程
tags:
  - 消息队列
date: 2016-11-07 23:20:27
---
本文记录了在macOS Sierra 10.12.2环境下搭建RabbitMQ的过程。
<!-- more -->
## 安装RabbitMQ
通过homebrew安装：
```
brew install rabbitmq
```
完成后如图：
![](/image/2016-11-07-23-21-57.jpg)
默认安装路径为：`/usr/local/sbin`，通过给`~/.zshrc`（shell使用zsh）添加：
```
export PATH=$PATH:/usr/local/sbin
```
后执行：
```
source ~/.zshrc
```
添加到默认路径。
## 启动RabbitMQ
执行：
```
rabbitmq-server
```
显示：

![](/image/2016-11-07-23-29-16.jpg)
即启动成功，可以通过命令：
```
rabbitmqctl status
```
查看启动状态：

![](/image/2016-11-07-23-30-07.jpg)

## Hello Workd
先通过以下命令安装Python下的AMQP库：
```
pip install pika
```
生产者：
```
# -*- coding: utf-8 -*-
import pika
import sys

credentials = pika.PlainCredentials("guest", "guest")
# 默认使用5672端口
conn_params = pika.ConnectionParameters("localhost", credentials=credentials)
# 建立到代理服务器的连接
conn_broker = pika.BlockingConnection(conn_params)
# 获得信道
channel = conn_broker.channel()
# 声明交换器
channel.exchange_declare(exchange="hello-exchange",
                         type="direct",
                         passive=False,
                         durable=True,
                         auto_delete=False)

msg = sys.argv[1]
msg_props = pika.BasicProperties()
# 创建纯文本消息
msg_props.content_type = "text/plain"
# 发布消息
channel.basic_publish(body=msg,
                      exchange="hello-exchange",
                      properties=msg_props,
                      routing_key="hola")



```
消费者：
```
# -*- coding: utf-8 -*-
import pika

credentials = pika.PlainCredentials("guest", "guest")
# 默认使用5672端口
conn_params = pika.ConnectionParameters("localhost", credentials=credentials)
# 建立到代理服务器的连接
conn_broker = pika.BlockingConnection(conn_params)
# 获得信道
channel = conn_broker.channel()
# 声明交换器
channel.exchange_declare(exchange="hello-exchange",
                         type="direct",
                         passive=False,
                         durable=True,
                         auto_delete=False)

# 声明队列
channel.queue_declare(queue="hello-queue")
# 通过routing_key绑定队列到交换器
channel.queue_bind(queue="hello-queue",
                   exchange="hello-exchange",
                   routing_key="hola")


def msg_consumer(channel, method, header, body):
    # 消息确认
    channel.basic_ack(delivery_tag=method.delivery_tag)
    if body == "quit":
        # 停止消费并退出
        channel.basic_cancel(consumer_tag="hello-consumer")
        channel.stop_consuming
    else:
        print body
    return

# 订阅消费者
channel.basic_consume(msg_consumer,
                      queue="hello-queue",
                      consumer_tag="hello-consumer")
channel.start_consuming()

```

provider:
![](/image/2016-11-07-23-51-07.jpg)
consumer:
![](/image/2016-11-07-23-51-20.jpg)
