---
layout: post
title: 'Kafka'
date: 2018-10-20
author: 邬晨
color: rgb(255,210,32)
cover: 'https://bing.ioliu.cn/v1/rand?w=900&h=300'
tags: Kafka
---

# Kafka的优化

> 平时用Kafka也比较多~~~

## 基础名词

1. Topic：          用于划分Message的逻辑概念，一个Topic可以分布在多个Broker上。
 2. Partition：   是Kafka中横向扩展和一切并行化的基础，每个Topic都至少被切分为1个Partition。
 3. Offset：        消息在Partition中的编号，编号顺序不跨Partition。
 4. Consumer： 用于从Broker中取出/消费Message。
 5. Producer：    用于往Broker中发送/生产Message。
 6. Replication：Kafka支持以Partition为单位对Message进行冗余备份，每个Partition都可以配置至少1个Replication(当仅1个Replication时即仅该Partition本身)。
 7. Leader：每个Replication集合中的Partition都会选出一个唯一的Leader，所有的读写请求都由Leader处理。其他Replicas从Leader处把数据更新同步到本地，过程类似大家熟悉的MySQL中的Binlog同步。
 8. Broker：Kafka中使用Broker来接受Producer和Consumer的请求，并把Message持久化到本地磁盘。每个Cluster当中会选举出一个Broker来担任Controller，负责处理Partition的Leader选举，协调Partition迁移等工作。
 9. ISR(In-Sync Replica)：是Replicas的一个子集，表示目前Alive且与Leader能够“Catch-up”的Replicas集合。由于读写都是首先落到Leader上，所以一般来说通过同步机制从Leader上拉取数据的Replica都会和Leader有一些延迟(包括了延迟时间和延迟条数两个维度)，任意一个超过阈值都会把该Replica踢出ISR。每个Partition都有它自己独立的ISR

## 流程总结

1. 数据存储被抽象为topic，topic表明了不同数据类型，在broker中可以有很多个topic

2. producer发出消息给broker，consumer订阅一个或者多个topic，从broker拿数据。从broker拿数据和存数据都需要编码和解码，只有数据特殊时，才需要自己的解码器。
3. consumer订阅了topic之后，它可以有很多的分组，sparkStreaming采用迭代器进行处理。生产者发布消息时，会具体到topic的分区中，broker会在分区的后面追加，所以就有时间的概念，当发布的消息达成一定阀值后写入磁盘，写完后消费者就可以收到这个消息了
4. kafka里没有消息的id，只有offset，而且kafka本身是无状态的，offset只对consumer有意义

## 简单归纳

Partition 越多,吞吐量越大,在集群数越少时稳定性越差,效率降低

Partition 越多,并发写入硬盘的数据越多如果在同一块硬盘上,会导致数据连续性下降

提前规划好Partition数量通过横向增加集群提高系统稳定性和性能

pagecache 是linux内核的低优先级缓存，在内存空间富裕的情况下才能获得较大的空间。并且kafka不自建缓存（零拷贝，直接内核处理写入磁盘，不进行应用程序的缓存），堆空间需求也比较小。我个人认为Kafka只要能启动起来就好了，留更大的内存空间给系统，以便保证pagecache的分配



**/proc/sys/vm/dirty_ratio** 
 这个参数控制文件系统的文件系统写缓冲区的大小，单位是百分比，表示系统内存的百分比，表示当写缓冲使用到系统内存多少的时候，开始向磁盘写出数据。增大 之会使用更多系统内存用于磁盘写缓冲，也可以极大提高系统的写性能。但是，当你需要持续、恒定的写入场合时，应该降低其数值，：

```shell
 echo '1' > /proc/sys/vm/dirty_ratio
```



**/proc/sys/vm/dirty_background_ratio**
 这个参数控制文件系统的pdflush进程，在何时刷新磁盘。单位是百分比，表示系统内存的百分比，意思是当写缓冲使用到系统内存多少的时 候，pdflush开始向磁盘写出数据。增大之会使用更多系统内存用于磁盘写缓冲，也可以极大提高系统的写性能。但是，当你需要持续、恒定的写入场合时， 应该降低其数值，：

```shell
 echo '1' > /proc/sys/vm/dirty_background_ratio
```

## 压测

2C4G的kafka集群  JVM配置为1G

使用官方自带的压测工具压测

./kafka-producer-perf-test.sh --topic topic_gb_error --num-records 10000000 --throughput 2000 --producer-props bootstrap.servers=10.142.168.150:9092 --record-size 3000

发送10000000条字节大小为3000的数据  每次发送2000条   CPU为 35.6%

发送10000000条字节大小为3000的数据  每次发送20000条   CPU为 104%

发送10000000条字节大小为30000的数据  每次发送2000条   CPU为 61%

发送10000000条字节大小为3000的数据  峰值大概为3W   CPU为 66%  瓶颈在于内存和硬盘读写

## 结论

系统层面:
 1.横向添加机器
 2.横向增加硬盘（能用ssd尽量用ssd）
 3.内存设置为1G，尽可能的留给系统
 4.使用i/o优化实例
 5.文件系统格式官方目前推荐XFS，禁用noatime
 6.EXT4格式 调整/proc/sys/vm/dirty_background_ratio和/proc/sys/vm/dirty_ratio

kafka配置:
 待完善