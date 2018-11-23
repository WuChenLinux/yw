---
layout: post
title: 'logstash 配置'
date: 2018-10-22
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/10-22/SpiritBearSleeps_ZH-CN7690026884_1920x1080.jpg'
tags: ELK
---

# logstash-k8s

> logstash是一个数据分析软件，主要目的是分析log日志，可以进行过滤和格式化
>
> 这里主要贴一下k8s的logstash配置，镜像使用的官方镜像~
>
> 把配置挂载到一个地方，然后启动的时候命令指一下就好了
>
> ```yaml
> spec:
>   containers:
>     - args:
>         - '-f'
>         - /etc/logstash/conf.d/logstash.conf
>       command:
>         - logstash
> ```



贴一下日志格式，放一个[日志切割网站](https://grokdebug.herokuapp.com/)

```shell
%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level ${PID:- } --- [%15.15t] : [%-40.40logger{39}][Line:%-3L] : %msg%n%wEx
```

嗯。。正则匹配效率很低


```yaml
input {                       #filebeat输出到5044端口
   beats {
    port => 5044
      }
   }
filter {
  grok {
# 我这里将自定义的pattern放在Logstash目录下的patterns文件夹内
    patterns_dir => ["/etc/logstash/patterns/"]
    match => { "message" => "%{DATETIME:datetime}%{SPACE:drop}%{LOGLEVEL:level}%{SPACE:drop}%{POSINT:pid}%{SPACE:drop}---%{SPACE:drop}%{BBBBB:}%{SPACE:drop}%{BBBBB:class}%{SPACE:drop}%{BBBBB:Line}%{SPACE:drop}:%{SPACE:drop}%{LOGMESSAGE:message}" }
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
    remove_field => "source"
    remove_field => "[prospector][type]"
    remove_field => "stream"
    remove_field => "tags"
    remove_field => "drop"
  }
}
   output {
     if [kubernetes][namespace] == "test-t3" {
    elasticsearch {
      hosts => ["elasticsearch-svc:9200"]
      index => "t3-%{type}-%{+YYYY.MM.dd}"
      document_type => "%{type}"
      flush_size => 20000
      idle_flush_time => 10
      template_overwrite => true
    }
    #stdout {
    #    codec => rubydebug
    #} 
  }
}
```



自定义正则文件

```shell
#贴一下所用到的正则，写的很垃圾- -
cat /usr/local/elk/logstash/patterns/test

BBBBB \[[^\[\]]+\]
DATETIME \d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2}.\d{3}
LOGLEVEL ([A-a]lert|ALERT|[T|t]race|TRACE|[D|d]ebug|DEBUG|[N|n]otice|NOTICE|[I|i]nfo|INFO|[W|w]arn?(?:ing)?|WARN?(?:ING)?|[E|e]rr?(?:or)?|ERR?(?:OR)?|[C|c]rit?(?:ical)?|CRIT?(?:ICAL)?|[F|f]atal|FATAL|[S|s]evere|SEVERE|EMERG(?:ENCY)?|[Ee]merg(?:ency)?)
POSINT \b(?:[1-9][0-9]*)\b
LOGMESSAGE [\w|\W]+
SPACE \s*
```

