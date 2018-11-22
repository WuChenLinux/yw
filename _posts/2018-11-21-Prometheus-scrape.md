---
layout: post
title: 'Prometheus 服务发现'
date: 2018-11-21
author: 邬晨
color: rgb(255,210,32)
cover: 'https://open.saintic.com/api/bingPic/'
tags: Prometheus
---
# Prometheus 服务发现

scrape_configs是指定Prometheus要监控的部分，可以说是最复杂的地方了

scrape_config中每个监控目标是一个job

job的类型有很多种，最简单的是static_configs，静态指定每一个目标  具体见[scrape_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)

```yaml
scrape_configs:

  - job_name: 'prometheus'

    # 默认采集路径 '/metrics'
    # 默认请求协议 'http'.

    static_configs:
      - targets: ['localhost:9090']
```

也可以使用服务发现的方式，动态发现目标，例如将kubernetes中的endpoints作为监控目标:

```yaml
  - job_name: 'kubernetes-service-endpoints'

    kubernetes_sd_configs:
    - role: endpoints

    relabel_configs:
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
      action: replace
      target_label: __scheme__
      regex: (https?)
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
    - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
      action: replace
      target_label: __address__
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
    - action: labelmap
      regex: __meta_kubernetes_service_label_(.+)
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: kubernetes_namespace
    - source_labels: [__meta_kubernetes_service_name]
      action: replace
      target_label: kubernetes_name    
```



首先要明白 Prometheus来获取数据，是根据url来进行获取的，而url是标签来拼接完成的

```yaml
#有内置意义的三个标签
__scheme__          : http、https等
__address__         : 检测目标的地址 
__metrics_path__    : 获取指标的路径

#   __scheme__://__address__/__metrics_path__   来进行访问
```

然后来看relabel_configs 每个选项的意思

你要先知道[可用元数据标签](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config) ，relabel_configs也是根据元标签来进行修改的

```yaml
source_labels: 元数据标签
#表现形式为 [__address__,__meta_kubernetes_service_annotation_prometheus_io_port]
#这也是后面替换的[$1,$2]
 
regex: 任何有效的RE2正则表达式   # default = (.*)

action: 动作   #分为replace(默认)，keep，drop，labelmap，labeldrop和labelkeep

#replace: 如果正则表达式与source_labels匹配。则将target_label和replacement替换为source_labels的值！！！($1,$2,...),如果正则表达式不匹配, 则不会进行替换
#keep: 如果正则能匹配上，则只！！！保留source_labels值！！！ 不匹配的都会被丢弃
#drop: 如果正则能匹配上，则删除source_labels值
#labelmap: 正则匹配标签名称。然后复制标签的值来标记给replacement（$1,$2,...）


#labeldrop: 匹配regex所有标签名称。匹配的任何标签都将从标签集中删除   匹配上正则，标签将被删除
#labelkeep: 匹配regex所有标签名称。任何不匹配的标签都将从标签集中删除  匹配不上正则，标签将被删除
#必须注意labeldrop 和 labelkeep要确保一旦标签被删除，度量标准仍然有唯一标记的。


target_label: 打标签
replacement: 替换内容 <string>类型      # default = $1  也可以 $1:$2  $1:222  等等
```



当我们了解了上面每个选项的意思之后，再来看relabel_configs 就很简单了

例如这个就是把正则比配到的地址+端口 重新赋予给 \__address__标签

内容就是  \__address__="10.10.0.144:9153"

```yaml
    - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
      action: replace
      target_label: __address__
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
```



这个可能表达的不是很清晰，有什么疑问可以联系我~
