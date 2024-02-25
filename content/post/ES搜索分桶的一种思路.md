---
title: ES搜索分桶的一种思路
comments: true
toc: true
categories:
  - 搜索
tags:
  - elasticsearch
date: 2017-09-16 22:16:33
---
一个简化版的有赞搜索实践。
<!-- more -->
感谢@有赞 提供的思路，可以去看这篇文章：[有赞搜索引擎实践](http://tech.youzan.com/you_zan_searchengine2/)

>实际上想达到店铺去重的效果通过分桶搜索是很容易做的事情. 我们假设每页搜索20个结果, 我们把索引库分成4个桶, 每个商品对桶数取模得到所在桶的编号. 这样可以保证同一店铺的商品仅在一个桶里面.

刚好我们也想做一件类似的事情，优化之前同一个优质商家的多个相关商品就是霸占搜索结果的前几页，分桶可以做到每个商家固定的出几个商品。

## 索引数据进入不同分桶

之前的文章有谈到我们还在用logstash将数据从mysql同步到es，具体可以参见：[MySQL到Elasticsearch的同步之路](http://suclogger.me/MySQL到Elasticsearch的同步之路/)

而logstash恰恰提供了很多强大的插件功能，这里主要用到了其中的[`ruby`插件](https://www.elastic.co/guide/en/logstash/current/plugins-filters-ruby.html)，这个插件logstash自带，不需要额外安装。

数据流是： MySQL -> Ruby Filter -> ES

在`Ruby Filter`这一层我们可以根据数据的属性做不同的路由，如：

```
filter
{
    ruby
    {
        code => '\''event.set("bulk_no", (event.get("[hash_key]") % 10)'\''
    }
}
```

这个filter根据索引的`hash_key`对10取模，而10就是总的分桶数，得到的`bulk_no`可以用于后续`output`中用于指定桶编号，如：

```
output {
    elasticsearch {
        index => "sku_bulk_%{[bulk_no]}"
        document_type => "sku"
        document_id => "%{sku_id}"
        hosts => ["127.0.0.1:9200"]
    }
}
```

## 从各分桶获取数据

>搜索的过程每个桶平均分摊搜索任务的25%, 并根据静态分合并成一页的结果. 这样同一保证结果的相对顺序, 又达到了店铺去重的目的.

以10个桶为例，分桶之后一次搜索请求分割为了10个到分桶的搜索请求，而且为了保证聚合结果，需要额外在全量桶中搜索一次，等于一次请求分裂为11次。
 为了保证响应速度不会显著下降，可以考虑并发请求各个分桶。


```
CountDownLatch count = new CountDownLatch(10);
for (int i = 0; i < 10; i++) {
    searchCondition.setIndexName("sku_bulk_" + i);
    res.add(ThreadPoolFactory.getThreadPool().submit(new RequestBulk(builder(searchCondition), count)));
}
count.await(1000, TimeUnit.MILLISECONDS);
res.foreach(fu->fu.get(1000, TimeUnit.MILLISECONDS));
```


## 没解决的问题

通过这种简易的方式，从各个分桶取出的结果条数不可控，就导致了每次搜索结果条数的不可控。

