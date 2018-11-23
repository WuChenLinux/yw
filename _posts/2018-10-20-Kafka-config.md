---
layout: post
title: 'Kafka 安装与配置解读'
date: 2018-10-20
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/10-20-Prometheus-rules/ChiribiqueteNP_ZH-CN10719426351_1920x1080.jpg'
tags: Kafka
---

# Kafka集群的安装与配置文件解读

## 目录

- [环境准备](#环境准备)
  - [系统环境](#系统环境)
- [环境部署](#环境部署)

- [基础命令](#基础命令)

## 环境准备

### 系统环境

| 主机名  | 系统版本 | 内网IP       | 安装软件                      | 安装路径                    |
| ------- | -------- | ------------ | ----------------------------- | --------------------------- |
| Kafka01 | centos7  | 172.16.21.81 | kafka(自带的zookeeper),JDK1.8 | /usr/local/kafka_2.12-1.0.1 |
| Kafka02 | centos7  | 172.16.21.83 | kafka(自带的zookeeper),JDK1.8 | /usr/local/kafka_2.12-1.0.1 |
| Kafka03 | centos7  | 172.16.21.82 | kafka(自带的zookeeper),JDK1.8 | /usr/local/kafka_2.12-1.0.1 |

## 环境部署

> 官网下载[Kafka软件包](https://kafka.apache.org/downloads),[JDK软件包](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) 上传到服务器上

1. 首先安装JDK，三台机器都要装

```shell
tar -xf jdk-8u162-linux-x64.tar.gz -C /usr/local
tar -xf kafka_2.12-1.0.1.tgz -C /usr/local

vim /etc/profile                                      #添加JDK的环境变量
export JAVA_HOME=/usr/local/jdk1.8.0_162
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

java -version                                        #验证JDK是否安装成功
```

2. 配置三台zookeeper并且启动，下面是一个示例

```shell
cd /usr/local/kafka_2.12-1.0.1/config/
vim zookeeper.properties                      
dataDir=/clzdata/zookeeper/data/
dataLogDir=/clzdata/zookeeper/logs
clientPort=2181
maxClientCnxns=0
tickTime=2000
initLimit=10
syncLimit=5
server.1=172.16.21.81:2887:3887
server.2=172.16.21.83:2888:3888
server.3=172.16.21.82:2889:3889

mkdir /clzdata/zookeeper/{data,logs} -p
echo '1' > /clzdata/zookeeper/data/myid      #每台机器和server.id保持一致
cd /usr/local/kafka_2.12-1.0.1/bin/
./zookeeper-server-start.sh -daemon ../config/zookeeper.properties
tail -f ../logs/zookeeper.out          #查看三台的zk启动日志，要三台都启动才报错，基本zk就完成了
```

3. 配置kafka，并启动

```shell
cd /usr/local/kafka_2.12-1.0.1/config/
vim server.properties 
broker.id=1                                   #brokerID要不一样
delete.topic.enable=true                      #开启删除topic的开关
listeners=PLAINTEXT://172.16.21.81:9092       #每台机器的监听地址
num.network.threads=3                      #服务器用于接收来自网络的请求并向网络发送响应的线程数
num.io.threads=8                           #服务器用于处理请求的线程数，包括磁盘I/O
socket.send.buffer.bytes=102400            #socket服务器发送缓冲区
socket.receive.buffer.bytes=102400         #socket服务器接收缓冲区
socket.request.max.bytes=104857600         #socket服务器接收请求的最大值(防止OOM)
log.dirs=/clzdata/kafka/logs               #用于存储日志文件的目录，可以多个'，'分离
num.partitions=9                           #每个topic的分区数
num.recovery.threads.per.data.dir=1        #在启动时用于日志恢复的每个数据目录的线程数, 在关闭时刷新，默认值为1启用

#保证数据可靠性
#默认1 生产者在ISR的leader已收到消息的确认后发送下一条，如果leader宕机了，则会丢失数据
#0 不管收不收到确认，都会发送下一条信息，可靠性最低
#-1 生产者需要等待ISR中的所有follower都确认收到消息后才算一次发送成功，可靠性最高
request.required.acks=-1
#ISR中最小的follower个数。默认值为1，当且仅当request.required.acks参数设置为-1才会生效
#如果ISR中的副本数少于min.insync.replicas配置的数量时，客户端会返回异常：org.apache.kafka.common.errors.NotEnoughReplicasExceptoin: Messages are rejected since there are fewer in-sync replicas than required
min.insync.replicas=1

#副本数对Kafka的吞吐率是有一定的影响，但极大的增强了可用性。默认情况下Kafka的replica数量为1，但是为了消息的可靠性，将大小设置为大于1
#topic : __consumer_offsets    记录偏移量
offsets.topic.replication.factor=3         #用于于配置offset记录的topic的的副本个数
offsets.topic.num.partitions=50            #用于于配置offset记录的topic的分区数，默认50
transaction.state.log.replication.factor=3 #事务主题的副本个数
transaction.state.log.num.partitions=50    #事务主题的分区个数，默认50
transaction.state.log.min.isr=1            #覆盖min.insync.replicas的参数

log.retention.hours=1                      #Kafka数据在被消费后一小时就可以被删除了
log.retention.bytes=1073741824             #日志数据存储的最大字节数,超过就会根据policy处理
log.segment.bytes=1073741824               #日志文件的最大大小，达到后创建新文件
log.retention.check.interval.ms=300000     #检查日志的时间，是否可以根据规则删除
log.cleanup.policy=delete                  #日志清理策略
log.cleaner.enable=false                   #false时，一旦日志的保存时间或者大小达到上限时，就会被删除，如果设置为true，则当保存属性达到上限时，就会进行日志压缩

zookeeper.connect=10.46.226.77:2181,10.26.91.126:2181,10.27.0.39:2181 #zk地址
zookeeper.connection.timeout.ms=6000       #连接zk超时时间设置
auto.leader.rebalance.enable=true          #自动平衡leader
controlled.shutdown.enable=true            #支持更加优雅的关机

group.initial.rebalance.delay.ms=6         #为了缩短多消费者首次平衡的时间，这段延时期间允许更多的消费者加入组

#三台机器启动Kafka
mkdir /clzdata/kafka/logs
cd /usr/local/kafka_2.12-1.0.1/bin/
./kafka-server-start.sh -daemon ../config/server.properties
tail -f ../logs/server.log                 #查看下启动日志~
```



## 基础命令

```shell
#创建topic，设置分区和副本数
./kafka-topics.sh --create --zookeeper 127.0.0.1:2181 --partitions 15 --replication-factor 2 --topic topic_can

#修改分区数，暂时不支持减少分区
./kafka-topics.sh  --alter --zookeeper 127.0.0.1:2181 --topic  "topic_can" --partitions 10

#列出topic
./kafka-topics.sh  --zookeeper 127.0.0.1:2181 --list

#查看消费者组
./kafka-consumer-groups.sh --bootstrap-server 172.16.21.81:9092 --list

#查看topic的详细信息
./kafka-topics.sh  --zookeeper 127.0.0.1:2181 --topic  "topic_can" --describe

#查看消费者组的消费情况
./kafka-consumer-groups.sh --bootstrap-server 172.16.21.81:9092 --describe --group test-consumer-group

./kafka-consumer-groups.sh --zookeeper 127.0.0.1:2181 --describe --group test-consumer-group

#删除某个消费者组，可以多个
./kafka-consumer-groups.sh --bootstrap-server 172.16.21.81:9092 --delete --group test-consumer-group --group my-group

#手动平衡leader
./kafka-preferred-replica-election.sh --zookeeper 127.0.0.1:2181

#手动消费
./kafka-console-consumer.sh --bootstrap-server 10.28.38.197:9092  --topic topic_can --from-beginning

#查看某个topic生产了多少消息
./kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list 172.16.21.81:9092 --topic topic_can --time -1| awk -F ':' '{print $3}' | awk '{sum += $1};END {print sum}'

#Kafka性能测试
./kafka-producer-perf-test.sh --topic topic_can --num-records 10000000 --throughput 200000 --producer-props bootstrap.servers=172.16.21.81:9092,172.16.21.82:9092,172.16.21.83:9092 --record-size 1000
# --topic 为此topic生产消息
# --num-records 要生产的消息数
# --throughput 最大消息吞吐量限制
# --producer-props kaafka集群相关的配置属性，如bootstrap.servers，client.id等
# --record-size 消息大小 --record-size或--payload-file必须存在且不能同时存在
#  --payload-file 文本文件
```

