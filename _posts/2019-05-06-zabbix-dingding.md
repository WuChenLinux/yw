---
layout: post
title: 'zabbix钉钉通知'
date: 2019-05-06
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-05-06/th.jpg'
tags: Zabbix
---





# zabbix 钉钉通知

zabbix 动作

```markdown
#### 告警主机: {HOST.NAME}  

####  IP地址: {HOST.IP}

#### 开始时间: {EVENT.DATE} {EVENT.TIME}

#### 当前状态: {ITEM.VALUE1}



#### 恢复主机: {HOST.NAME}  

####  IP地址: {HOST.IP}

#### 恢复时间: {EVENT.DATE} {EVENT.TIME}

#### 当前状态: {ITEM.VALUE1}
```



## python版本

> python2.7.5

```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-

import urllib
import urllib2
import json
import sys

#获取变量
user = sys.argv[1]
title = sys.argv[2]
text = sys.argv[3]

#填写钉钉机器人url
requrl = "https://oapi.dingtalk.com/robot/send?access_token=XXXXXXXX"

message = {
     'msgtype': 'markdown',
     'markdown': {
                  'title': title,
                  'text':'%s \n\n %s' %(title,text)
                 }
          }

header_dict = {'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; Trident/7.0; rv:11.0) like Gecko',
                   "Content-Type": "application/json"}
json_data = json.dumps(message).encode(encoding='utf-8')
req = urllib2.Request(url = requrl,data = json_data,headers = header_dict)

res_data = urllib2.urlopen(req)
res = res_data.read()

```

## golang版本

> 人生苦短

```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
	"strings"

	jsoniter "github.com/json-iterator/go"
)

type Message struct {
	Msgtype  string `json:"msgtype"`
	Markdown struct {
		Title string `json:"title"`
		Text  string `json:"text"`
	} `json:"markdown"`
	At struct {
		AtMobiles []string `json:"atMobiles"`
		IsAtAll   bool     `json:"isAtAll"`
	} `json:"at"`
}

func main() {
	Post()
}

func DingData() string {
    // 获取变量
	//  user := os.Args[1]
	Title := os.Args[2]
	Text := os.Args[3]

	// 为了换行符
	lines := []string{
		Title,
		Text,
	}
	//fmt.Println(strings.Join(lines, "\r\n"))
	message := strings.Join(lines, "\r\n")

	p := &Message{}
	p.Msgtype = "markdown"
	p.Markdown.Title = Title
	p.Markdown.Text = message
	p.At.AtMobiles = []string{"1xxxxxx0234"}
	p.At.IsAtAll = false
	jsondata, _ := jsoniter.Marshal(p)
	fmt.Println(string(jsondata))
	data := string(jsondata)
	return data
}

func Post() {
	data := DingData()
	dingdingURL := "https://oapi.dingtalk.com/robot/send?access_token=xxxxxxx"
	request, _ := http.NewRequest("POST", dingdingURL, strings.NewReader(data))
	request.Header.Set("Content-Type", "application/json")
	resp, err := http.DefaultClient.Do(request)
	if err != nil {
		fmt.Printf("post data error:%v\n", err)
	} else {
		fmt.Println("post a data successful.")
		respBody, _ := ioutil.ReadAll(resp.Body)
		fmt.Printf("response data:%v\n", string(respBody))
	}
}

```



## 效果

> 在考虑加颜色了

![1557122286814](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-05-06/1557122286814.png)