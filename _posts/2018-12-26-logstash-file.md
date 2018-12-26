---
layout: post
title: 'logstash 输出到文件'
date: 2018-12-26
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2018-12-26/OxfordBoxing_ZH-CN2854964515_1920x1080.jpg'
tags: ELK
---



# logstash 输出到文件

今天遇到客户的一个需求,需要把每个服务的日志,集中到一台服务器上查看(嗯，喜欢大黑屏，不喜欢kibana)

嗯，客户至上！然后想了一下，去看了看logstash的官网，发现有这个output，就搞了一下

主要利用了Kafka的特性


```shell
input {
    kafka  {
      bootstrap_servers => "172.16.225.23:9092"
      group_id => "test01"                    #kafka的消费者，一定不能和你之前的消费者一样
      topics => ["ca-pre","ca-pro"]
      codec => "json"
    }
}

#emmmmmm 保证源日志的完整性，就没有做切割
output {
if [fields][type] == "ca-pre" or [fields][log_topics] == "ca-pre" {
file {
   path => "/clzdata/logs/pre/%{[beat][name]}/%{+YYYY-MM-dd}.log"  #字段可以为变量
   codec => line { format => "%{message}"}          #输出message字段
        }
    }
if [fields][type] == "ca-pro" or [fields][log_topics] == "ca-pro" {
file {
   path => "/clzdata/logs/pro/%{[beat][name]}/%{+YYYY-MM-dd}.log"
   codec => line { format => "%{message}"}
        }
    }

}

```

真的，这个需求我也是- - - - - -- - - - --  -- --- -- -  -- - -- - -- 。

![1545827712483](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2018-12-26/1545827712483.png)

完成！自己看去吧!