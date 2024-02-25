---
title: UNIX网络编程学习笔记（二）
comments: true
toc: true
categories:
  - 编程
tags:
  - UNIX
date: 2020-05-11 03:45:52
---
UNIX网络编程学习笔记（二）
<!-- more -->
## 大纲
![](/image/2020-05-11/第二章.png)
## 习题
### “2.1　我们已经提到IPv4（IP版本4）和IPv6（版本6）。IP版本5情况如何，IP版本0、1、2和3又是什么？”

参考：
https://www.heficed.com/blog/ip-address-evolution-ipv4-vs-ipv6-has-ipv5-gone-missing
https://blog.alertlogic.com/blog/where-is-ipv1,-2,-3,and-5/

其余都是实验的或者被证明失败的版本，并未被正式推广

### “2.2 你从哪里可以找到有关IP版本5的信息？”
IP版本5从未正式发布。参考：https://en.wikipedia.org/wiki/Internet_Stream_Protocol

### “2.3　在讲解图2-15时我们说过，如果没收到来自对端的MSS选项，本端TCP就采用536这个MSS值。为什么使用这个值？”

536 = （最小重组缓冲区大小576）- （TCP标准头 20）- （TCP标准头选项20）

### “2.4　给在第1章中讲解的时间获取客户/服务器应用画出类似于图2-5的分组交换过程，假设服务器在单个TCP分节中返回26个字节的完整数据。”
本地通过第一章中的程序访问：128.138.140.44 交互报文如下：
![](/image/2020-05-11/2020-05-13-01-47-29.jpg)
可以得出交互图如下
![](/image/2020-05-11/TCP-Based DateTime.png)
同时DayTime协议也支持UDP，这里就略过了。

### “2.5　在一个以太网上的主机和一个令牌环网上的主机之间建立一个连接，其中以太网上主机的TCP通告的MSS为1460，令牌环网上主机的TCP通告的MSS为4096。两个主机都没有实现路径MTU发现功能。观察分组，我们在两个相反方向上都找不到大于1460字节的数据，为什么？”

我的理解是，若观察分组是从TCP报文分组的视角，是因为TCP报文需要基于IP报文的上限MSU拆分成不大于1460的数据报文段，才支持基于IP协议向下传递。

### “2.6　在讲解图2-19时我们说过OSPF直接使用IP。承载OSPF数据报的IPv4首部（见图A-1）的协议字段是什么值？”
可以参考IINA或者[维基页面](https://en.wikipedia.org/wiki/List_of_IP_protocol_numbers)
给OSPF分配的协议号为89。
附OSPF报文的结构：
![](/image/2020-05-11/2020-05-13-02-18-05.jpg)

### “2.7　在讨论SCTP输出时我们说过，SCTP发送端必须等待累积确认点超过已发送的数据，才可以从套接字缓冲区中释放该数据。假设某个选择性确认（SACK）表明累积确认点之后的数据也得到了确认，这样的数据为什么却不能被释放呢？”

SCTP内容暂时忽略。


