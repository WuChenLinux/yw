---
layout: post
title: 'logstash-json'
date: 2019-04-25
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-04-25/th1.jpg'
tags: ELK
---





# logstash-json

> 场景就是message消息既有普通文本，也有json
>
> 哎  就非常蛋疼

```shell
# Sample Logstash configuration for creating a simple
# Beats -> Logstash -> Elasticsearch pipeline.


input {
  kafka {
    bootstrap_servers => "10.1.1.47:9092"
    group_id => "logstash"
    topics => ["pro-finance"]
    codec => "json"
    consumer_threads => "4"
    client_id =>  "logstash01"
  }
}

filter {

if [fields][log_topics] == "pro-finance" {
  grok {
# 我这里将自定义的pattern放在Logstash目录下的patterns文件夹内
    patterns_dir => ["/etc/logstash/patterns/"]
    match => { "message" => "%{DATETIME:datetime}%{SPACE:drop}%{LOGLEVEL:level}%{SPACE:drop}%{POSINT:pid}%{SPACE:drop}---%{SPACE:drop}%{BBBBB:thread}%{SPACE:drop}%{NOTSPACE:class}%{SPACE:drop}:%{SPACE:drop}%{LOGMESSAGE:message}" }
    match => { "message" => "%{DATETIME:datetime}%{SPACE:drop}%{WORD:default}%{SPACE:drop}%{BBBBB:thread}%{SPACE:drop}%{LOGLEVEL:level}%{SPACE:drop}%{NOTSPACE:class}%{SPACE:drop}-%{SPACE:drop}%{LOGMESSAGE:message}" }
    overwrite => [ "message" ]
  }
 date {
    match => [ "datetime", "yyyy-MM-dd HH:mm:ss.SSS" ]
    target => "@timestamp"
 }
mutate {
   remove_field => "[beat][hostname]"
   remove_field => "[beat][name]"
   remove_field => "[input][type]"
   remove_field => "default"
   remove_field => "source"
   remove_field => "[prospector][type]"
   remove_field => "stream"
   remove_field => "tags"
   remove_field => "pid"
   remove_field => "[log][file][path]"
   remove_field => "drop"
   remove_field => "[log][flags]"
  }
#kong这个标签的日志 不会被上面的匹配到的，所以还是json，所以就if一下然后json解析就好了
if [kubernetes][labels][app] == "kong" {
json {
   source => "message"
   target => "kong_message"
}
}
}
}


output {

if [kubernetes][namespace] == "prod-finance"  or [host][name] == "finance01" {
if [fields][log_topics] == "pro-finance" {
  elasticsearch {
    hosts => ["http://10.1.1.47:9200"]
    index => "pro-finance-%{+YYYY.MM.dd}"
}
}
}
}


```



效果如下 这里还保留了一下message

![1556173766927](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-04-25/1556173766927.png)