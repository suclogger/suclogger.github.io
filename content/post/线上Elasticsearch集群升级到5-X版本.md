---
title: 线上Elasticsearch集群升级到5.X版本
comments: true
toc: true
categories:
  - 搜索
tags:
  - elasticsearch
date: 2016-12-22 01:04:10
---
本文记录了将Elasticserach集群从`2.3.3`升级到`5.0.2`过程当中的一些坑。
<!-- more -->
## Elasticsearch
官方提供了升级Elasticsearch的详细[文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html)：
![](/image/2016-12-22-01-10-21.jpg)
集群的升级可以细化到每个节点的升级过程，也有相关的[帮助文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/rolling-upgrades.html#upgrade-node)。
官方还非常周到的提供了一个迁移助手[elasticsearch-migration](https://github.com/elastic/elasticsearch-migration/tree/2.x)，以插件的形式检测升级过程中需要注意的事项。
首先通过插件来做一些准备工作：
![](/image/2016-12-08-14-58-42.jpg)
主要包含了：Cluster Checkup、Reindex Helper、Deprecation Logging 3个工具，分别点击可以查看对应的需要处理的事项。
### 配置项变更
配置项的变化非常大，我主要涉及到下面几点：
* 索引相关属性都不再在集群范围定义，而是细化到各个索引当中，涵盖了分词器：`index`、`index.analysis.analyzer.default.type`、`index.analysis.analyzer.default.tokenizer`、`index.analysis.analyzer.default.filter`和分片信息：`index.number_of_shards`、`index.number_of_replicas`等；
* 网络层的配置更加考虑安全性，需要通过`http.cors.enabled: true`和`http.cors.allow-origin: *“`来启用非本地请求，`network.host`也移除了` _non_loopback_`，需要详细配置允许的域名列表；
* deprecate了groovy语言支持，推荐painless。
* max_map_count需要262144：
    * 写入`vm.max_map_count=262144`到`/etc/sysctl.conf`后执行`sysctl -p`
* max_file limit需要至少65536：
    * 修改`/etc/security/limits.conf`中的相应值为65536

新的配置项准备完成之后就可以开始升级了。

### 禁用自动分片
通过执行：
```
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.enable": "none"
  }
}
```
禁用自动部署分片到节点上。

### 将内存数据同步到磁盘(可选)
执行：
```
POST _flush/synced
```
### 关闭节点、执行升级
**如果你现有使用的jdk版本非1.8，需要升级jdk版本**
我是通过修改符号链接指向的方式指定elasticsearch版本的：
![](/image/2016-12-22-01-34-30.jpg)
将符号链接指向新的tar包解压文件，将老的`config`和`data`目录也同步过去，进行必要的配置项变更即可。

### 启用节点，开启自动分片
由于新老节点版本不同，启动节点之后会出现以下报错提示，忽略即可：
![](/image/2016-12-22-01-40-19.jpg)
执行：
```
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.enable": "all"
  }
}
```

依次在每个节点执行以上步骤即可。
## elasticsearch-head
head本来是以插件形式嵌入到elasticsearch中，5.0之后只能通过独立方式运行，所以需要elasticsearch配置中添加：`http.cors.enabled: true`和`http.cors.allow-origin: *“`。
安装步骤为：
* 安装依赖：
    * 添加epel源：`sudo yum localinstall http://rpms.famillecollet.com/enterprise/remi-release-7.rpm`
    * 安装node：`sudo yum -y install nodejs npm --enablerepo=epel`
    * 可以通过命令：`PROJECT_NAME="node" PROJECT_URL="https://npm.taobao.org/mirrors/node/" n project stable`快速升级node
    * 可以通过命令：`PHANTOMJS_CDNURL=https://npm.taobao.org/dist/phantomjs npm install phantomjs --registry=https://registry.npm.taobao.org --no-proxy`加快PHANTOMJS安装
* 拉取代码：`git clone git://github.com/mobz/elasticsearch-head.git`
* `npm install`。可以通过`npm config set registry https://registry.npm.taobao.org`添加淘宝npm源加快速度。
* `grunt server`即可运行。如果需要在断开ssh链接后继续运行，需要通过`tmux`。

## logstash
logstash主要用于同步数据库数据到搜索，详见[MySQL到Elasticsearch的同步之路](http://suclogger.me/MySQL%E5%88%B0Elasticsearch%E7%9A%84%E5%90%8C%E6%AD%A5%E4%B9%8B%E8%B7%AF/)。
安装的时候遇到的问题就是无法安装插件，表现为执行`./logstash-plugin install logstash-input-jdbc`时提示：
> Could not find gem 'logstash-plugin (>= 0) java' in any of the gem sources listed in your Gemfile or installed on this machine

尝试通过其他装有插件的机器执行`./logstash-plugin pack --tgz`导出gem文件，在需要安装的节点执行`./logstash-plugin install --local logstash-input-jdbc`依然失败。

这似乎是logstash的一个bug，可以参见这个[issue](https://github.com/elastic/logstash/issues/5721)

纠结了好久之后，直接从其他装有插件的机器上把logstash整个文件夹迁移了过来。

## java-api
java-api 也又很大变化，主要有：
* 抽离出transtaparent client的独立jar包
* netty升级到4
* 传入的日期字符串需要通过jodatime来格式化
* function score 函数需要组装为数组

需要注意的就是项目出现：
```
java.lang.NoClassDefFoundError: org/jboss/netty/channel/ReceiveBufferSizePredictorFactory
```
等相关的`NoClassDefFoundError`异常，大都是jar包版本冲突导致的，在idea中可以通过`Maven Helper`插件快速查看。

![](/image/2016-12-22-02-19-38.jpg)

