---
layout: post
title: 'Elasticsearch缩减节点'
date: 2019-08-08
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-08-08/th.jpg'
tags: ELK
---





# ES缩减节点

官方传送门: https://www.elastic.co/guide/en/elasticsearch/reference/current/allocation-filtering.html

> 排除的时候，一定要一次性排除掉，不要先排除1再排除2 要不然第一次排除的还会再分配

```shell
# 排除某个节点
PUT _cluster/settings
{
  "transient" : {
    "cluster.routing.allocation.exclude._ip" : "10.1.1.251,10.1.1.252"
  }
}
```

查看监控，会发现排除节点的shards慢慢减少，当为0的时候，就可以把节点干掉了

![1565242433844](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-08-08/1565242433844.png)

green 多么美好的颜色~

![1565242486973](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-08-08/1565242486973.png)