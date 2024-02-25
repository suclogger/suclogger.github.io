---
title: 代码重构之：Mybatis批量写入
comments: true
toc: true
categories:
  - 编程
tags:
  - java
date: 2017-06-22 23:28:28
---
今天做了一点代码重构，以此为记。
<!-- more -->
业务场景是作为dubbo服务提供方，先到一张表写入数据后取出主键，以此主键为外键到另一张表写入数据。
因为处理时间过长，导致一系列的dubbo，nginx超时。

遍历记录列表，2张表每次各写入一条记录，写入2000条耗时约100s，日志输出：
```
2017-06-22 22:34:13:2017-06-22 22:34:44.849  INFO 177 --- [:23889-thread-9] c.c.c.s.impl.OOXXServiceImpl     :...
2017-06-22 22:35:51:2017-06-22 22:36:22.261  INFO 177 --- [:23889-thread-9] c.c.c.s.impl.OOXXServiceImpl     : finish insert
```

**要支持批量写入并且回传ID，需要Mybatis版本大于3.3.1，详情见：[github issue](https://github.com/mybatis/mybatis-3/pull/547)**

重构代码添加mapper层方法：
```
<insert id="batchInsert" useGeneratedKeys="true" keyProperty="someId">
        INSERT INTO some_table
        <trim prefix="(" suffix=")" suffixOverrides=",">
            <if test="list[0].otherId != null">
                otherId,
            </if>
        </trim>
        values
        <foreach collection="list" item="someDO" index="index" separator="," >
            <trim prefix="(" suffix=")" suffixOverrides=",">
                <if test="someDO.otherId != null">
                    #{someDO.otherId,jdbcType=BIGINT},
                </if>
            </trim>
        </foreach>
    </insert>
```

主键会回写到传入的DO中。
两张表各写入2000，共计耗时1s，提升100倍，日志输出：
```
2017-06-22 22:57:13:2017-06-22 22:57:29.377  INFO 178 --- [23889-thread-51] c.c.c.s.impl.OOXXServiceImpl     :  ...)
2017-06-22 22:57:13:2017-06-22 22:57:30.283  INFO 178 --- [23889-thread-51] com.ooxx.core.ao.impl...     : batch insert OO count : 2000
2017-06-22 22:57:13:2017-06-22 22:57:30.786  INFO 178 --- [23889-thread-51] c.c.core.ao.impl....      : batch insert XX , count 2000
```
