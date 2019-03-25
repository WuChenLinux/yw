---
layout: post
title: 'kibana地图'
date: 2019-03-25
author: 邬晨
color: rgb(238,233,191)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-03-25/th.jpg'
tags: ELK
---




# kibana地图


## 安装geoip

```shell
cd /opt/apps/logstash-6.6.1/bin
./logstash-plugin install logstash-filter-geoip

yum install GeoIP-data -y
#手动指定geoip库
cd /opt/apps/logstash-6.6.1/config/
wget https://geolite.maxmind.com/download/geoip/database/GeoLite2-City.tar.gz
tar -xf GeoLite2-City.tar.gz
```



## logstash配置

> 凑活看

```shell
# Sample Logstash configuration for creating a simple
# Beats -> Logstash -> Elasticsearch pipeline.


input {
  kafka {
    bootstrap_servers => "10.1.1.47:9092"
    group_id => "logstash"
    client_id =>  "logstash06"
    topics => ["pro-nginx"]
    codec => "json"
    consumer_threads => "4"
  }
}

filter {

if [fields][log_topics] == "pro-nginx" {
#if [kubernetes][labels][app] == "eureka-server" {
  grok {
# 我这里将自定义的pattern放在Logstash目录下的patterns文件夹内
    patterns_dir => ["/etc/logstash/patterns/"]
    match => { "message" => "%{IPORHOST:clientip} - - \[%{HTTPDATE:timestamp}\] \"%{WORD:verb} %{URIPATHPARAM:request} HTTP/%{NUMBER:httpversion}\" %{NUMBER:response} (?:%{NUMBER:bytes}|-) (?:\"(?:%{URI:referrer}|-)\"|%{QS:referrer}) %{QS:agent} \"%{IP:xforwardedfor}\"" }
#    overwrite => [ "message" ]
  }

#source是切割好的IP地址
geoip {
     source => "xforwardedfor"
     target => "geoip"
     database => "/opt/apps/logstash-6.6.1/config/GeoLite2-City_20190319/GeoLite2-City.mmdb"
     add_field => ["[geoip][coordinates]","%{[geoip][longitude]}"]
     add_field => ["[geoip][coordinates]","%{[geoip][latitude]}"]
        }
      mutate {
        convert => [ "[geoip][coordinates]", "float"]       #转化经纬度的值为浮点数
          }

 date {
    match => [ "timestamp", "yyyy-MM-dd HH:mm:ss.SSS" ]
    target => "@timestamp"
 }
 
#删除无用字段 
#mutate {
#   remove_field => "[beat][hostname]"
#   remove_field => "[beat][name]"
#   remove_field => "[input][type]"
#   remove_field => "default"
#   remove_field => "source"
#   remove_field => "[prospector][type]"
#   remove_field => "stream"
#   remove_field => "tags"
#   remove_field => "pid"
#   remove_field => "[log][file][path]"
#   remove_field => "drop"
#   remove_field => "[log][flags]"
#  }
}
}


#报错成logstash是为了匹配一个模板 不然geoip.location的类型需要自己去改
output {
if [fields][log_topics] == "pro-nginx" {
  elasticsearch {
    hosts => ["http://10.1.1.47:9200"]
    index => "logstash-nginx-%{+YYYY.MM.dd}"
}
}
}

```



## kibana使用高德地图

> 设置完重启下就好

```shell
#最下面添加
vim kibana.yml
tilemap.url: 'http://webrd02.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=7&x={x}&y={y}&z={z}'
```



![1553502507459](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-03-25/1553502507459.png)

![1553502523730](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-03-25/1553502523730.png)

![1553502542649](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-03-25/1553502542649.png)

## 效果图

![1553502083108](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-03-25/1553502083108.png)
