---
title: solr中文全词匹配的问题
comments: true
toc: true
categories:
  - 搜索
tags:
  - solr
date: 2016-05-3 22:36:13
---
<!-- abstract -->

<!-- 开始正文 -->

正常情况下的全词匹配可以用双引号括住需要检索的关键字,比如: q=name:"keyword"
但是如果某个字段用配置了中文分词器,用双引号括住无法查找到对应记录,只能用分词的结果的一部分做模糊查询,比如要精确查询 "美丽人生" ,用 q=name:"美丽人生" 是无法匹配到的,只能用 q=name:"美丽" ,或者 q=name:"人生" 才能匹配到.
中文分词配置如下
```
<field name="name" type="textComplex" indexed="true" stored="true"/>

...

<fieldType name="textComplex" class="solr.TextField" positionIncrementGap="100">
    <analyzer type="index">
            <tokenizer class="com.chenlb.mmseg4j.solr.MMSegTokenizerFactory" mode="complex"/>
            <filter class="solr.WordDelimiterFilterFactory"
            splitOnNumerics="0"
            generateWordParts="1"
            generateNumberParts="1"
            catenateWords="0"
            catenateNumbers="0"
            catenateAll="0"
            preserveOriginal="1"
            />
            <filter class="solr.LowerCaseFilterFactory"/>
            <filter class="solr.EdgeNGramFilterFactory" minGramSize="1" maxGramSize="10"/>
    </analyzer>
    <analyzer type="query">
<tokenizer class="solr.StandardTokenizerFactory"/>
 <filter class="solr.LowerCaseFilterFactory"/>
    </analyzer>
</fieldType>

```

解决的思路就是索引两个字段，一个分词，一个不分词，但是他们都索引了，在查询时，同时查询这两个字段，分词的字段：模糊查，不分词的字段：精确查，这样以来既能保证召全率，也能保证查准率,配置如下:
```
<!-- 这个字段用于模糊匹配 -->
<field name="name" type="textComplex" indexed="true" stored="true"/>

<!-- 这个字段用于精确匹配 -->
<field name="exactName" type="exactText" indexed="true" stored="true"/>

<!--保持这两个字段相等 -->
<copyField source="name" dest="exactName"/>

```



