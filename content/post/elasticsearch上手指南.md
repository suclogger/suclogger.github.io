---
title: elasticsearch上手指南
comments: true
toc: true
categories:
  - 搜索
tags:
  - elasticsearch
date: 2016-06-10 19:09:26
---
<!-- abstract -->
<!-- 开始正文 -->
## CentOS上安装elasticsearch
首先需要JAVA环境，这里不再详述。
在[官网下载页面](https://www.elastic.co/downloads/elasticsearch)找到RMP包的下载路径，通过wget命令下载安装包到本地：
```
wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/rpm/elasticsearch/2.3.3/elasticsearch-2.3.3.rpm
```
执行安装
```
sudo rpm -ivh elasticsearch-2.3.3.rpm
```
默认安装到`/usr/share/elasticsearch/`，配置文件在`/etc/elasticsearch`，init脚本在`/etc/init.d/elasticsearch`。
添加为服务：
```
sudo systemctl enable elasticsearch.service
```
启动：
```
sudo /etc/init.d/elasticsearch start
```
查看：
```
curl http://localhost:9200
```
## 创建一个文稿
```
curl -XPUT 'localhost:9200/get-together/group/1?pretty' -d '{ "name": "Elasticsearch Denver", "organizer": "Lee" }'
```
可以看到返回信息：
```
{
  "_index" : "get-together",
  "_type" : "group",
  "_id" : "1",
  "_version" : 1,
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "created" : true
}
```
Elasticsearch自动添加了`get-together`索引，并创建了一个到类型`group`的映射。
当然你也可以手动创建一个索引：
```
curl -XPUT 'localhost:9200/new-index'
```
Elasticsearch自动将`name`和`organizer`检测为`string`类型，如果你又添加了一个文稿，包含除了`name`和 `organizer`的其他字段，Elasticsearch也会自动检测它的类型并添加到映射。
可以查看这个映射的详细信息：
```
curl 'localhost:9200/get-together/_mapping/group?pretty
{
  "get-together" : {
    "mappings" : {
      "group" : {
        "properties" : {
          "name" : {
            "type" : "string"
          },
          "organizer" : {
            "type" : "string"
          }
        }
      }
    }
  }
}
```
## 开始搜索
添加一些示例文稿：
```
git clone git@github.com:suclogger/elasticsearch-in-action.git
sh elasticsearch-in-action/populate.sh
```
这个脚本会重建我们之前创建的`get-together`索引。
一个简单的搜索：
```
curl "localhost:9200/get-together/group/_search?q=elasticsearch&fields=name,location&size=1&pretty"
```
curl请求的链接中，`get-together/group`指定了范围：`get-together`索引下的`group`类型。
常见的关键词格式是：`q=name:elasticsearch`，这里没有指定group下的某个参数，elasticsearch会默认指定为`q=_all:elasticsearch`来搜索所有参数字段。
也可以通过`,`分隔来同时搜索多个类型，如：`curl "localhost:9200/get-together/group,event/_search?q=elasticsearch&fields=name,location&size=1&pretty"`，或者完全忽略类型部分来搜索所有类型：`curl "localhost:9200/get-together/_search?q=elasticsearch&fields=name,location&size=1&pretty"`。
类似的，也可以通过`,`分隔来同时搜索多个索引，如：`curl "localhost:9200/get-together,other-index/_search\ ?q=elasticsearch&pretty"`，当然这个请求会失败，因为我们还没创建`other-index`索引。也可以指定`_all`来搜索所有索引。
返回的结果中：
```
{
  "took" : 5,
  "timed_out" : false,
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : 0.83339834,
    "hits" : [ {
      "_index" : "get-together",
      "_type" : "group",
      "_id" : "2",
      "_score" : 0.83339834,
      "fields" : {
        "name" : [ "Elasticsearch Denver" ]
      }
    } ]
  }
}
```
`took`表示查询耗时5ms，`timed_out`为false表示查询未超时（默认下查询永远不会超时，可以在curl链接后附带`&timeout=3s`来指定超时时间为3s）












