---
layout: post
title: '删除docker image'
date: 2019-11-07
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-11-07/th.jpg'
tags: Docker
---





# 删除docker image

> 当一个k8s节点上构建过很多次之后，会留下很多过时的镜像需要删除，类似于下图

![image-20191107174134485](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-11-07/image-20191107174134485.png )

#### 删除过时镜像

```shell
docker rmi $(docker images |grep artemis-server |awk '{print $3}')
#或
for i in $(docker images |grep xxxxx |awk '{print $2}' ); do docker rmi registry.cn-hangzhou.aliyuncs.com/xxxx/xxxxx:$i; done
```

#### 删除异常停止的容器

```shell
docker rm `docker ps -a | grep Exited | awk '{print $1}'`
```

####  删除 dangling 或所有未被使用的镜像 

```shell
docker image prune 
```

#### 删除未被使用的数据卷

```shell
docker volume prune
```

####  删除所有退出状态的容器 

```shell
docker container prune
```

####  删除已停止的容器、dangling 镜像、未被容器引用的 network 和构建过程中的 cache 

```shell
docker system prune
```

