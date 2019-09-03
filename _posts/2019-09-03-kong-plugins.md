---
layout: post
title: 'kong 插件开发'
date: 2019-09-02
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-09-03/th.jpg'
tags: kong
---



# kong 插件开发

> 本来想使用kong的serverless 函数的  结果发现有些库引用很麻烦，索性就写个插件了
>
> 背景是给客户端写一个接口 来控制下灰度升级
>
> 算法是取巧了些，利用了概率，然后概率的范围控制了下，反正要么升级，要么不升级 嗯 伯努利概型

### 环境

> 本地开发调试比较简单
>
> 插件目录在/usr/local/share/lua/5.1/kong/plugins/

```shell
cat /usr/local/share/lua/5.1/kong/constants.lua
# plugins里新加一个http-service的插件模块
```



```shell
ll /usr/local/share/lua/5.1/kong/plugins/http-service/
handler.lua    #必须
schema.lua     #必须
```



### 模块开发

> 直接上代码最爽快
>
> 根据权重返回给客户端v1或者v2
>
> 根据imei进行去重，多次请求返回相同结果，key存活时间1天 足够灰度测试
>
> 又分了下各个渠道app请求
>
> 把东西基本都提成了配置，这样不用更新比较方便

```lua
cat handler.lua

local BasePlugin = require "kong.plugins.base_plugin"
local LuckService = BasePlugin:extend()

local redis = require "resty.redis"

LuckService.PRIORITY = 99
LuckService.VERSION = "0.1.0"

function LuckService:new()
    LuckService.super.new(self, "LuckPlugin")
end


function LuckService:access(conf)
    LuckService.super.access(self)

    local imei = kong.request.get_header("imei")
    local channel = kong.request.get_header("channel")
    local rt_version = "v1"

    local cjson = require("cjson")


    LuckTable = {
        Share = 1,
        Fengyun = 2,
        Easypay = 3,
        Share_channel = Share,
        Fengyun_channel = Fengyun,
        Easypay_channel = Easypay,
        v2_Share_weight = (conf.v2_Share_weight),
        v2_Fengyun_weight = (conf.v2_Fengyun_weight),
        v2_Easypay_weight = (conf.v2_Easypay_weight),
        Share_rt_json_v1 = cjson.decode((conf.Share_rt_json_v1)),
        Share_rt_json_v2 = cjson.decode((conf.Share_rt_json_v2)),
        Fengyun_rt_json_v1 = cjson.decode((conf.Fengyun_rt_json_v1)),
        Fengyun_rt_json_v2 = cjson.decode((conf.Fengyun_rt_json_v2)),
        Fengyun_rt_json_v1 = cjson.decode((conf.Easypay_rt_json_v1)),
        Easypay_rt_json_v2 = cjson.decode((conf.Easypay_rt_json_v2)),
    }


    if(imei == nil) then
        ngx.status = 200
        ngx.say("Please give me a imei !") 
        return ngx.exit(200)
    end

    local red = redis:new()
    local ok,err = red:connect(conf.redis_host,conf.redis_port)
    if not ok then
        ngx.log(ngx.ERR,"failed to connect: ",err)
        return ngx.exit(502)
    end
    local res,err = red:auth(conf.redis_password)
    if not res then
        ngx.log(ngx.ERR,"failed to authenticate: ",err)
        return ngx.exit(502)
    end

    kong.log("imei is : ", imei, " Channel is : ", channel)
    kong.log("redis db is : ", LuckTable[(channel)])
    red:select(LuckTable[(channel)])
    res, err = red:get(imei)

    kong.log("imei The version that exists is : ", res)
    ngx.status = 200
    ngx.header["Content-Type"] = "application/json; charset=utf-8"
    if(res == "v2") then
        rt_version = "v2"
        ngx.say(cjson.encode(LuckTable[(channel).."_rt_json_v2"]))
        return ngx.exit(200)
    elseif (res == "v1") then
        rt_version = "v1"
        ngx.say(cjson.encode(LuckTable[(channel).."_rt_json_v1"]))
        return ngx.exit(200)
    else
        math.randomseed(os.time())
        local luck = math.random(0,100)
        kong.log("luck is : ", luck)
        local weight = LuckTable["v2_"..(channel).."_weight"]
        kong.log("channel weight is : ", channel, ":",weight)

        if luck <= weight then     
            rt_version = "v2"
            red:set(imei,"v2")
            kong.log("imei version is : ", imei," , ",rt_version)
            red:expire(imei,5184000)
            ngx.say(cjson.encode(LuckTable[(channel).."_rt_json_v2"]))
            return ngx.exit(200)
        else
            rt_version = "v1"
            red:set(imei,"v1")
            red:expire(imei,5184000)
            kong.log("imei version is : ", imei," , ",rt_version)
            ngx.say(cjson.encode(LuckTable[(channel).."_rt_json_v1"]))
            return ngx.exit(200)
        end
    end
end

return LuckService

```



```lua
cat schema.lua

local typedefs = require "kong.db.schema.typedefs"

return {
  name = "http-service",
  fields = {
    { consumer=typedefs.no_consumer },
    { protocols = typedefs.protocols_http },
    { config = {
        type = "record",
        fields = {
          { policy = {
              type = "string",
              default = "redis",
              len_min = 0,
              one_of = {"redis" },
          }, },
          { v2_Share_weight =  { type = "number", default = 10, },  },
          { v2_Fengyun_weight =  { type = "number", default = 10, },  },
          { v2_Easypay_weight =  { type = "number", default = 10, },  },
          { Share_rt_json_v1 = { type = "string", len_min = 0 }, },
          { Share_rt_json_v2 = { type = "string", len_min = 0 }, },
          { Fengyun_rt_json_v1 = { type = "string", len_min = 0 }, },
          { Fengyun_rt_json_v2 = { type = "string", len_min = 0 }, },
          { Easypay_rt_json_v1 = { type = "string", len_min = 0 }, },
          { Easypay_rt_json_v2 = { type = "string", len_min = 0 }, },
          { redis_host = typedefs.host },
          { redis_port = typedefs.port({ default = 6379 }), },
          { redis_password = { type = "string", len_min = 0 }, },
          { redis_timeout = { type = "number", default = 2000, }, },
        },
        custom_validator = validate_periods_order,
      },
    },
  },
  entity_checks = {
    { conditional = {
      if_field = "config.policy", if_match = { eq = "redis" },
      then_field = "config.redis_host", then_match = { required = true },
    } },
    { conditional = {
      if_field = "config.policy", if_match = { eq = "redis" },
      then_field = "config.redis_port", then_match = { required = true },
    } },
    { conditional = {
      if_field = "config.policy", if_match = { eq = "redis" },
      then_field = "config.redis_timeout", then_match = { required = true },
    } },
  },
}

```

### 效果

> 配置使用konga， 然后测试的话 就多测就完事了

![1567509146890](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-09-03/1567509146890.png)
