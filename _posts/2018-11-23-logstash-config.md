---
layout: post
title: 'logstash 简单用法'
date: 2018-11-23
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/11-23/VarennaSnow_ZH-CN7673479242_1920x1080.jpg'
tags: ELK
---

# logstash 简单用法

> 其实配置之前，去读一遍官网挺好的
>
> 今天被人问了一天的logstash的问题- -  
>
> 写一下把，一些简单的配置



官网：[filebeat添加字段](https://www.elastic.co/guide/en/beats/filebeat/6.5/configuration-general-options.html#libbeat-configuration-fields)

有不同的设置，配置多个输入就好了。6版本写法如下：

```shell
filebeat.prospectors:
- type: log
  enabled: true
  paths:
    - /clzdata/logs/jarslogs/*/execute_time*.log
  fields:                           #添加的自定义字段
    log_topics: execute-time
  multiline:
    pattern: ^\d{4}-
    negate: true
    match: after
    timeout: 2s
  tail_files: true
- type: log
  enabled: true
  paths:
    - /clzdata/logs/jarslogs/*/*.log
  fields:
    log_topics: ca-pro
  multiline:
    pattern: ^\d{4}-
    negate: true
    match: after
    timeout: 2s
  tail_files: true

#
output.kafka:
  enabled: true
  hosts: ["172.16.225.23:9092"]
  topic: '%{[fields][log_topics]}'   #创建的topic可以写死，也可以写变量的形式，取上面的字段
  worker: 2
  max_retries: 3
  bulk_max_size: 2048
  timeout: 30s
  broker_timeout: 10s
  channel_buffer_size: 256
  keep_alive: 60
  compression: gzip
  max_message_bytes: 1000000
  required_acks: 0

```



logstash：[官方if示例](https://www.elastic.co/guide/en/elastic-stack-get-started/6.5/get-started-elastic-stack.html#logstash-filter)

**你如果实在不知道自己可以判断什么字段，你就什么都别加，**

**看logstash的日志输出也行**

**去看kibana的索引字段都有哪些也行**

```shell
#######################配置从Kafka输入###########################################
input {
  kafka {
    bootstrap_servers => "172.16.225.23:9092"
    group_id => "logstash"             #如果配置多个logstash，要保持一致，详见官网
    topics => ["ca-pro", "ca-pre", "execute-time"] #可以配置多个
    codec => "json"
    consumer_threads => "4"
    client_id =>  "logstash01"           #每个input不同
  }
}
########################配置过滤############################################
filter {
#这里是你要判断的字段，如果匹配则执行这个，否则跳过
#因为我这里环境问题，日志格式每个团队之间还没有统一，所以filter这里可以判断一下，分别过滤
  if [fields][log_topics] == "execute-time" {
    grok {
      patterns_dir => ["/clzdata/apps/logstash/patterns"] 
      match => {
        "message" => "%{DATETIME:time} %{SPACES:logleave} %{SPACES:drop} %{SPACES:drop} %{SPACES:drop} %{SPACES:drop} %{SPACES:drop} %{SPACES:drop} %{SPACES:drop} %{SPACES:Interface} %{SPACES:drop} %{SPACES:execution_time}"
      }
      overwrite = >["message"]
    }
    date {
      match => ["time", "yyyy-MM-dd HH:mm:ss.SSS"] 
      target = >"@timestamp"
    }
    mutate {
      convert => ["execution_time", "float"] 
      remove_field => "drop"
      remove_field => "message"
    }
  }

  #同上
  if [fields][log_topics] == "ca-pro" {
    grok {
    patterns_dir => ["/clzdata/apps/logstash/patterns"] 
    match => {
        "message" => "%{DATETIME:datetime}%{SPACE:drop}%{LOGLEVEL:level}%{SPACE:drop}%{HOSTNAME:hostname}%{SPACE:drop}%{POSINT:pid}%{SPACE:drop}---%{SPACE:drop}%{BBBBB:thread}%{SPACE:drop}%{BBBBB:class}%{SPACE:drop}%{BBBBB:Line}%{SPACE:drop}:%{SPACE:drop}%{LOGMESSAGE:message}"
      }
      overwrite => ["message"]
    }
    grok {
    patterns_dir => ["/clzdata/apps/logstash/patterns"] 
    match => {
        "source" => "/%{WORD:drop}/%{WORD:drop}/%{WORD:drop}/%{WORD:drop}/%{HOSTNAME:servername}/%{TIP:logname}"
      }
    }
    date {
      match => ["datetime", "yyyy-MM-dd HH:mm:ss.SSS"] 
      target => "@timestamp"
    }
    mutate {
      remove_field => "[beat][hostname]"
      remove_field => "[beat][name]"
      remove_field => "[input][type]"
      #remove_field => "source"
      remove_field => "[prospector][type]"
      remove_field => "stream"
      remove_field => "tags"
      remove_field => "drop"
    }
  }
}

######################配置输出#################################################
output {
  #同理，只有匹配到的才会输出
  #因为历史遗留原因 有filebeat5和6版本，每个版本都有些细微的变动
  #就在这里加了个or，哈哈哈可能是我比较懒- -
  if [fields][type] == "ca-pro" or [fields][log_topics] == "ca-pro" {
    elasticsearch {
      hosts => ["10.46.226.77:9200", "10.26.91.126:9200", "10.27.0.39:9200"] 
      index =>"ca-pro-%{+YYYY-MM-dd}"
    }
  }
  if [fields][type] == "ca-pre" or [fields][log_topics] == "ca-pre" {
    elasticsearch {
      hosts => ["10.46.226.77:9200", "10.26.91.126:9200", "10.27.0.39:9200"] 
      index =>"ca-pre-%{+YYYY-MM-dd}"
    }
  }
  if [fields][log_topics] == "execute-time" {
    elasticsearch {
      hosts => ["10.46.226.77:9200", "10.26.91.126:9200", "10.27.0.39:9200"] 
      index =>"execute-time-%{+YYYY-MM-dd}"
    }
  }
}
```



PS1: logstash是可以写多个文件的，还是建议写在多个文件中，比较清晰

PS2: 正则的效率非常低

PS3: 请结合实际环境
