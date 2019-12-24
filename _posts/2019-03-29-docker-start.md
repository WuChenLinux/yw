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
docker run --name mysql --cap-add=SYS_TIME -p 3306:3306 -v /opt/data/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=qwfakjklk123456 -d mysql:5.7
```

## Redis

```shell
docker run --name redis -p 6379:6379 -v /opt/data/redis:/data -d --restart=always redis:latest redis-server --appendonly yes --requirepass "redis123456qwe" --databases 64
```

## memcache

```shell
docker run --name memcache -p 11211:11211 -d --restart=always memcached memcached -m 512
```

## consul
```shell
docker run -d --net=host --name=test-consul01 -v /opt/data/consul/:/data/consul -e CONSUL_BIND_INTERFACE=ens160 consul
```

## pgsql
```shell
docker run --name postgresql -p 5432:5432 -v /opt/postgresql/:/var/lib/postgresql/data -e "POSTGRES_USER=kong" -e "POSTGRES_DB=kong" -e "MYSQL_ROOT_PASSWORD=kong123" -d postgres
```

## konga
```shell
docker run -d -p 1337:1337  -e "DB_ADAPTER=postgres" -e "DB_HOST=172.27.83.61" -e "DB_PORT=5432" -e "DB_USER=kong" -e "DB_PASSWORD=kong" -e "DB_DATABASE=wuchen-konga" -e "NODE_ENV=development" --name konga pantsel/konga
```

## elasticsearch
```shell
docker run -d -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -v /opt/elasticsearch/data/:/usr/share/elasticsearch/data  docker.elastic.co/elasticsearch/elasticsearch:7.3.1
```

## cadvisor
```shell
docker run --volume=/:/rootfs:ro --volume=/var/run:/var/run:rw --volume=/sys:/sys:ro --volume=/var/lib/docker/:/var/lib/docker:ro --publish=8080:8080 --detach=true --name=cadvisor google/cadvisor
```

## prometheus
```shell
docker run -d -p 9090:9090 -v /opt/data/prometheus:/prometheus -v /opt/config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml --name=prometheus prom/prometheus
```

## node-exporter
```shell
docker run -d -p 9100:9100 -v "/proc:/host/proc:ro" -v "/sys:/host/sys:ro" -v "/:/rootfs:ro" --name=node-exporter prom/node-exporter
```
