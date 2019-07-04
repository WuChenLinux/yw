---
layout: post
title: 'Elasticsearch字段太多'
date: 2019-06-12
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-06-12/th.jpg'
tags: ELK
---



# Elasticsearch字段太多

> 某一天，es的日志突然报Limit of total fields [1000] in index 然后发现是json那个索引，然后询问了开发人员，是每个接口都打印出了json ，所以就会出现超过字段数量的事情了，那就我们这边来更改设置

## 设置索引的字段数量

> 一个索引中能定义的字段的最大数量，默认是 1000，一般是够用的，但是有些场景解析json会暴增

```shell
#可以在kibana的Dev Tools请求
PUT */_settings
{
  "index.mapping.total_fields.limit": 2000
}
#或者服务器#
curl -XPUT 10.1.1.47:9200/*/_settings -d '{"index.mapping.total_fields.limit": 2000}'
```



因为我们的索引设置一般是通过时间来的，所以要更改下模板设置

```shell
curl -XPUT '10.1.1.47:9200/_template/all ' -d '
{
  "template": "*",
  "settings": {
    "index.mapping.total_fields.limit": 3000,
    "refresh_interval": "30s"
  }
}'
```



### 查看是否修改成功

```shell
curl -XGET 10.1.1.47:9200/pro-json-finance-2019.06.12/_settings?pretty
```

![1560305530244](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-06-12/1560305530244.png)



PS: 解析json一定要事先和开发人员确认好字段类型，不然坑太大了

查看集群中不同节点，不同索引的状态
```shell
GET _cat/shards?h=index,shard,prirep,state,unassigned.reason
```
副本置为1
```shell
curl -XPUT "http://localhost:9200/_settings" -d' { "number_of_replicas" : 0 } '
```
