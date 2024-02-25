---
title: Elasticsearch中文社区20160618杭州线下聚会纪要
comments: true
toc: true
categories:
  - 搜索
tags:
  - elasticsearch
date: 2016-06-18 22:56:21
---
<!-- abstract -->
<!-- 开始正文 -->
## 汪兴－Elasticsearch结合hbase的应用
介绍中使用的存储介质是HBase，我司目前还未达到迁移到NoSQL的数据量级，而且后面的分享中大都采用hadoop（hive，spark）作为mysql数据的全量备份，目前看来参考的价值不大。
不过后面也有一些具有参考意义的，比如提到的可以将若干条件作为一组进行分组条件逻辑查询，还涉及到备份方式的讨论，脑裂问题的解决。

## 万斌－ElasticSearch在gooagoo的实践
算是对于搜索的另一个方面的应用：数据统计，将传统BI的工作用elasticsearch来实现。
提到了Sense插件，可以自动补全查询的json。
提到了terms  Aggregation无法精确统计的问题。

## 曾勇－Introduction to Beats and extending
作为中文鼎鼎大名IK分词器的作者，medcl给我们带来了一个新的产品线Beats系列，是将来能很大程度取代logstash的一个产品，大部分还在开发验证阶段。

## 洪斌－有赞搜索引擎实践
是今天干货最多的一个topic了，虽然在量级上不如淘宝，但是设计中需要考虑到的方面基本覆盖到了。
印象很深的地方就是很大程度依赖hadoop来全量备份线上数据和进行索引构建。
1. 通过监控mysql的binlog，产生消息到kafka，消费消息实时将mysql的事务同步到搜索中。![](/image/2016-06-19 00-25-41.jpg)
2. 商用搜索引擎的AdvancedSearch的概念，将反向代理和通用算法层抽离出来。
3. 提到了评分体系的静态分和动态分的概念，静态分需要稳定和线性，动态分体现搜索的相关度。
4. 商品去重的问题，使用spark进行矢量相似度计算，在搜索时有限搜索去重之后的主库，只在出现召回等情况时搜索辅库。![](/image/2016-06-19 00-24-10.jpg)
5. 店铺去重的问题，通过分bucket的方式，在匹配度高的店铺和匹配度低的店铺中做了折衷。![](/image/2016-06-19 00-22-44.jpg)
6. Query分析，通过分析搜索日志和购买日志画出query转化图，进行后续的同义词扩展，核心词提取和智能纠错方面的应用。
7. 将搜索输入的关键字转化为不同权重的同义词输入es查询。![](/image/2016-06-19 00-21-16.jpg)
8. 使用rolling技术替代性能很差的分页操作。
9. 优先采用filtered_query。
9. 使用用bulk。 

## 宁海元－基于ELK的云日志产品实践
主要介绍ELK的日志实现，技术细节不多。

## 卢栋－elasticsearch-jdbc介绍及基于binlog的增量同步方案
算是最贴近我司现有起步阶段的topic，介绍了elasticsearch-jdbc的使用。
提到了elasticsearch-jdbc使用中的一些痛点，如定时轮询的操作会影响mysql的性能，多数据源配置比较麻烦和sql的增量控制的时间条件不准确，从而提出了通过监听binlog来实现实时同步的方案简单有效，很受启发。
![](/image/2016-06-19 00-31-14.jpg)
还提到了通过alias不停机迁移来做定时的全量，也很受启发。

