---
layout: post
title: 'alertmanager 沉默规则'
date: 2018-11-22
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/11-22-alertmanager-amtool/EibseeHerbst_ZH-CN9383344658_1920x1080.jpg'
tags: Prometheus
---

# alertmanager 沉默规则

> 今天用到了这个沉默规则，记录下 
>
> 使用的是官方docker镜像：https://hub.docker.com/r/prom/alertmanager/
>
> 官方github地址:  https://github.com/prometheus/alertmanager   

docker hub 上的架构图哈哈哈哈哈

![1542879471951](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/11-22-alertmanager-amtool/1542879471951.png)





容器启动之后，进入到容器中

```shell
#最重要的东西！！！！！！！！！！！！！
#之前只是简单的 amtool -h了，哎心疼自己
amtool --help-long
```

配置文件地址

```shell
cat /etc/amtool/config.yml   

#alertmanager地址,这里定义了之后命令行中可以不带 --alertmanager.url="http://127.0.0.1:9093"
alertmanager.url: "http://localhost:9093"

#提交人
author: wuchen

#是否需要备注才能提交，默认true：必须要备注
comment_required: false

#设置默认输出格式
output: extended

#默认接收器，测试时候可以用
receiver: webhook
```




```shell
#查看当前已触发的报警
amtool alert --alertmanager.url="http://127.0.0.1:9093"

#查看所有沉默
amtool silence query --alertmanager.url="http://127.0.0.1:9093"

#查看匹配的沉默
amtool silence query alertname=PodMemory  --alertmanager.url="http://127.0.0.1:9093"


#沉默添加，如果comment_required设置为true，不写comment的话会提交不上
#alertname就是Prometheus那里设置的alert
#--start 生效时间
#--end   结束时间
amtool silence add alertname="PodMemory" container_name="elasticsearch" --start="2019-01-02T15:04:05+08:00" --end="2019-01-02T15:05:05+08:00" --comment="test"  -a "wuchen" --alertmanager.url="http://127.0.0.1:9093"

#docker官方镜像，默认数据源保存在/alertmanager，所以要挂载数据盘，不然容器重启一切都白做

#因为查看沉默列表的时候，生效时间输出的不完整，所以可以
amtool silence query -o json --alertmanager.url="http://127.0.0.1:9093"
#还可以导出json数据
amtool silence query -o json PodMemory > 1.json --alertmanager.url="http://127.0.0.1:9093"

#沉默解除
amtool silence expire 8188b168-1332-4398-83a5-a9df4263c60d --alertmanager.url="http://127.0.0.1:9093"

#解除所有沉默
amtool silence expire $(amtool silence query -q --alertmanager.url="http://127.0.0.1:9093") --alertmanager.url="http://127.0.0.1:9093"


```

官网镜像是把数据源设置到了/alertmanager，我们挂载一个数据盘到这个目录就好了，不然容器重启后，一切操作都没了。

嗯，就记录下。
