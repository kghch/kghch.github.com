---
layout: post
title: Is Kafka a Database 笔记
category: tech
---
DDIA的作者Martin Kleppmann去年在Kafka Summit做的演讲“ Is Kafka a Database？”， 时长28分钟： https://www.youtube.com/watch?v=v2RJQELoM6Y

（标题吸睛，其实讲的是kafka如何实现ACID能力）

是什么让DB之所以成为DB的？ ACID ，从这个角度看
    * A, all or nothing，描述的是错误处理能力。2PC 两阶段提交 能保证，但并非所有系统都能兼容2PC。
        * 譬如，更新一条记录 同时到 ： web server, redis, mysql . 
        * kafka如何做到？ 生成一个描述update的事件发往kafka，下游web server, redis, mysql 隶属于3个consumer group分别监听
    * C （C有特殊性，它更应该是应用层的特性，而非DB的特性）
    * I, 描述的是并发处理能力。最高的I能力保证是“序列化”，多线程执行就仿佛单线程执行一般。mysql 直接用隔离级别/事务来保证
        * 譬如，两个用户同时抢注册jane这个用户名
        * kafka如何做到？ 想要注册的用户生成注册事件发往kafka，下游消费并执行真正的检测和插入。 并发问题通过kafka通道转化成串行问题
        * 实时性，Kafka引入带来的时延是个额外的问题. 并且没法同步得到结果
    * D，持久化，不只持久化到非易失性存储（如硬盘），还要持久化到多个机器（非单机）
        * It’s really easy for kafka.  既有单机持久化到磁盘，又有partition的冗余

**By Kafka, transactions are broken down into multi-stage stream pipelines.**

从ACID角度看，kafka是database，但当然kafka不支持像DB的查询。“横看成岭侧成峰”，当从不同角度看一个事务时，能得出不同的认知。

BTW，温习下mysql的四种事务隔离级别

    * 读未提交。 事务A和事务B， 事务A读到了事务B未提交的数据
    * 读已提交。事务A先读取了数据（res 1)，事务B修改了数据，事务A又读取了数据（res 2).    同一个事务内读取多次结果不一致
        * 大多数据库的默认隔离级别
    * 可重复读。   （这个级别会出现幻读，但其实这个推导是有问题的。  可以从下面角度理解：不可重复读侧重表达 读-读，幻读侧重表达 读-写 （其实写之前也隐式地读了一次）
        * MVCC 做到
        * Mysql的默认隔离级别
    * 可串行读。


