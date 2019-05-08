---
layout: post
title: 'sentinl插件报警'
date: 2019-03-15
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-03-15/th.jpg'
tags: ELK
---



# kibana sentinl插件报警



> 日志收集完，可以根据需求来进行日志的一个报警，这里用的kibana的插件sentinl


## 安装插件

GitHub地址：<https://github.com/sirensolutions/sentinl/>

根据自己的版本来进行安装插件，可能遇到网络因素，多试几次吧，安装好之后 重启服务

```shell
./kibana-plugin install https://github.com/sirensolutions/sentinl/releases/download/tag-6.6.0-0/sentinl-v6.6.1.zip
```



## 配置邮件告警smtp

```shell
vim /etc/mail.rc

set from=admin@blog-wuchen.cn
set smtp=smtp.blog-wuchen.cn
set smtp-auth-user=admin@blog-wuchen.cn
set smtp-auth-password=xxxxxxxx
set smtp-auth=login
```



```yml
vim kibana.yml   #在最后添加
sentinl:
  settings:
    email:
      active: true
      user: admin@blog-wuchen.cn
      password: xxxxxxx
      host: smtp服务器
      ssl: true
      port: xxx
      timeout: 10000
```

## 配置告警规则

> 这个插件有图形化UI 不过最后还是会转换为Advanced，所以还不如直接这样来的省事也容易理解
>
> 就根据自己的需求 写es的查询 然后进行匹配
>
> 这个就是查找ERROR字段里数据超过1 就发送

```json
{
  "actions": {
    "email_html_alarm_76b83c8f-0f4a-4db5-8a15-185933e17ca2": {
      "name": "日志异常邮件告警",
      "throttle_period": "2m",
      "email_html": {
        "stateless": false,
        "subject": "日志异常邮件告警",
        "priority": "medium",
        "html": "<h2>告警服务：{{#payload.hits.hits}}</h2><blockquote><p>{{_source.kubernetes.labels.app}}</p ><h2>告警时间:</h2><p>{{_source.datetime}}</p ><h2>告警类:</h2><p>{{_source.class}}</p ><h2>异常日志:</h2><p>{{_source.message}} {{/payload.hits.hits}} </p ></blockquote> <hr />",
        "to": "xxxx@blog-wuchen.cn,xxxx@blog-wuchen.cn",
        "from": "admin@blog-wuchen.cn"
      }
    },
    "Webhook_f3303006-a643-42f6-a2ff-8d4066d18c3a": {
      "name": "钉钉告警",
      "throttle_period": "2m",
      "webhook": {
        "priority": "medium",
        "stateless": false,
        "method": "POST",
        "host": "oapi.dingtalk.com",
        "port": "443",
        "path": "/robot/send?access_token=xxxxxxx",
        "body": "{\r\n    \"msgtype\": \"markdown\",\r\n    \"at\": {\r\n        \"isAtAll\": \"false\"\r\n    },\r\n    \"markdown\": {\r\n        \"title\": \"异常消息\",\r\n        \"text\": \" 异常日志: \\n {{#payload.hits.hits}} \\n 服务: {{kubernetes.labels.app}} \\n 时间: {{_source.datetime}} \\n  类: {{_source.class}} \\n 异常信息: {{_source.message}} {{/payload.hits.hits}}\"\r\n    }\r\n}",
        "params": {
          "watcher": "{{watcher.title}}",
          "payload_count": "{{payload.hits.total}}"
        },
        "headers": {
          "Content-Type": "application/json"
        },
        "message": "日志告警",
        "use_https": true
      }
    },
    "disable": false,
    "report": false,
    "title": "日志告警",
    "save_payload": false,
    "spy": false,
    "impersonate": false
  },
  "input": {
    "search": {
      "request": {
        "index": [
          "pro-finance-*"
        ],
        "body": {
          "query": {
            "bool": {
              "must": [
                {
                  "match": {
                    "level": {
                      "query": "ERROR"
                    }
                  }
                },
                {
                  "bool": {
                    "must_not": [
                      {
                        "match": {
                          "message": "ErrorScene^_^"
                        }
                      }
                    ]
                  }
                }
              ],
              "filter": {
                "range": {
                  "@timestamp": {
                    "gte": "now-2m/m",
                    "lte": "now/m",
                    "format": "epoch_millis"
                  }
                }
              }
            }
          },
          "size": 2,
          "aggs": {
            "dateAgg": {
              "date_histogram": {
                "field": "@timestamp",
                "time_zone": "Asia/Shanghai",
                "interval": "1m",
                "min_doc_count": 1
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "script": "payload.hits.total >= 1"
    }
  },
  "trigger": {
    "schedule": {
      "later": "every 2 minutes"
    }
  },
  "disable": false,
  "report": false,
  "title": "告警",
  "save_payload": false,
  "spy": false,
  "impersonate": false
}
```



然后效果

![1552623945403](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-03-15/1552623945403.png)
