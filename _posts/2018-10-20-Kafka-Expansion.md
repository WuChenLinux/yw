---
layout: post
title: 'Kafka扩容'
date: 2018-10-20
author: 邬晨
color: rgb(255,210,32)
cover: 'https://bing.ioliu.cn/v1/rand?w=900&h=300'
tags: Kafka
---

# Kafka的扩容



扩容：

1）先部署好新节点环境，并根据上文"配置修改"修改配置，然后启动集群，确保新节点为可用状态。

2）分区重新分配：在原来机器上的主题分区不会自动均衡到新的机器，需要使用分区重新分配工具来均衡均衡

3）“高级命令“的[Expanding your cluster](http://kafka.apache.org/documentation/#basic_ops_cluster_expansion)小节介绍了扩容的基本方法：

1. 生成扩容使用的json文件：

    

   ```shell
   cat  topics-to-move.json
   
   {"topics": [{"topic": "foo1"},     #foo1,2..是要手动指定的topic名称
               {"topic": "foo2"}],
    "version":1
    }
   ```

2. 通过上一步写好的json文件，使用kafka命令生成数据迁移配置

    > 这个时候，迁移还没有开始，它只是告诉你当前分配和新的分配规则，当前分配规则用来回滚，新的分配规则保存在json文件

    ```shell
    bin/kafka-reassign-partitions.sh --topics-to-move-json-file topics-to-move.json --zookeeper 127.0.0.1:2181 --broker-list "0,1,2,3,4" --generate   #01234是指定数据迁移到那些broker。
    {"version":1,
    "partitions":[{"topic":"foo1","partition":2,"replicas":[1,2]},
                  {"topic":"foo1","partition":0,"replicas":[0,1]},
                  {"topic":"foo2","partition":2,"replicas":[1,2]},
                  {"topic":"foo2","partition":0,"replicas":[0,2]},
                  {"topic":"foo1","partition":1,"replicas":[2,1]},
                  {"topic":"foo2","partition":1,"replicas":[1,0]}]
    }
     
    Proposed partition reassignment configuration
     
    {"version":1,
    "partitions":[{"topic":"foo1","partition":2,"replicas":[0,1,2,3,4]},
                  {"topic":"foo1","partition":0,"replicas":[0,1,2,3,4]},
                  {"topic":"foo2","partition":2,"replicas":[0,1,2,3,4]},
                  {"topic":"foo2","partition":0,"replicas":[0,1,2,3,4]},
                  {"topic":"foo1","partition":1,"replicas":[0,1,2,3,4]},
                  {"topic":"foo2","partition":1,"replicas":[0,1,2,3,4]}]
    ```

3. 将第一部分保存留作回退备份(即Proposed partition reassignment configuration上面的json串)，下面json串为扩容将要使用的到的配置，将其保存为expand-cluster-reassignment.json

    ```shell
    cat expand-cluster-reassignment.json
    {"version":1,
    "partitions":[{"topic":"foo1","partition":2,"replicas":[0,1,2,3,4]},
                  {"topic":"foo1","partition":0,"replicas":[0,1,2,3,4]},
                  {"topic":"foo2","partition":2,"replicas":[0,1,2,3,4]},
                  {"topic":"foo2","partition":0,"replicas":[0,1,2,3,4]},
                  {"topic":"foo1","partition":1,"replicas":[0,1,2,3,4]},
                  {"topic":"foo2","partition":1,"replicas":[0,1,2,3,4]}]
    ```

4. 执行扩容命令：

    ```shell
    bin/kafka-reassign-partitions.sh --zookeeper 127.0.0.1:2181 --reassignment-json-file expand-cluster-reassignment.json --execute
    ```

      正常执行的话会生成同上图类似的json串，表示原始状态和目标状态

5. 查询执行状态： 

   ```shell
   bin/kafka-reassign-partitions.sh --zookeeper 127.0.0.1:2181 --reassignment-json-file expand-cluster-reassignment.json --verify
   ```

     正常执行后会返回当前数据迁移的不用partion的，信息状态类似下面

   ```shell
   Reassignment of partition [foo1,0] completed successfully   #移动成功
   Reassignment of partition [foo1,1] is in progress           #这行代表数据在移动中
   Reassignment of partition [foo1,2] is in progress
   Reassignment of partition [foo2,0] completed successfully
   Reassignment of partition [foo2,1] completed successfully
   Reassignment of partition [foo2,2] completed successfully
   ```

6. 数据迁移一旦开始无法停止，也不要强行停止集群，这样会造成数据不一致，带来无法挽回的后果。

7. 注意：kafka数据迁移的原理是先拷贝数据到目标节点，然后再删除原节点的数据。这样的话如果集群原节点空间不足，不要继续指定其为迁移broker，这样将造成原节点空间用尽，例如原节点是broker为0，1，2，3，4就不要这样指定 --broker-list "**0,1,2,3,4**"，应该这样 --broker-list "**5,6**"。
    另外数据迁移也可以通过手工定制。