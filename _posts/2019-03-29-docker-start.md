---
layout: post
title: 'Docker启动小玩意'
date: 2019-03-29
author: 邬晨
color: rgb(238,232,170)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-03-29/th1.jpg'
tags: Docker
---





# Docker启动小玩意



> 开发环境快速部署 docker起一下完事
>
> 后续有需求再更新

## MySQL

```shell
docker run --name mysql -p 3306:3306 -v /opt/data/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=qwfakjklk123456 -d mysql:5.7
```

## Redis

```shell
docker run --name redis -p 6379:6379 -v /opt/data/redis:/data -d --restart=always redis:latest redis-server --appendonly yes --requirepass "redis123456qwe"
```

## memcache

```shell
docker run --name memcache -p 11211:11211 -d --restart=always memcached memcached -m 512
```

## consul
```shell
docker run -d --net=host --name=test-consul01 -v /opt/data/consul/:/data/consul -e CONSUL_BIND_INTERFACE=ens160 consul
```