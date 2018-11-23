---
layout: post
title: 'Docker基础'
date: 2018-10-19
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/10-19/EcolaSP_ZH-CN10746626161_1920x1080.jpg'
tags: Docker
---

# Docker基础

> 最近一直在搞kubernetes，就顺便写一下docker

### 概念

docker需要知道它的三个组件，分别是镜像（image）、容器（container）、仓库（repository）。这三个组件是docker的原理核心

#### 镜像

Docker运行容器前需要本地存在对应的镜像：

镜像可以用来创建Docker容器的。一个镜像可以包含一个完整的操作系统环境和用户需要的其它应用程序。在docker hub 里面有大量现成的镜像提供下载。docker的镜像是只可读的，一个镜像可以创建多个容器。

#### 容器

docker利用容器来开发、运行应用。

容器是镜像创建的实例。它可以被启动、开始、停止、删除。每个容器都是 相互隔离的、保证安全的平台。

#### 仓库

仓库是集中存放镜像文件的场所。

每个 仓库中又包含了多个镜像，每个镜像有不同的标签（tag）

### 安装

> centos7安装docker很简单,如果之前安装过，先进行卸载

```shell
#个人比较喜欢清华大学开源镜像站
yum remove docker docker-common docker-selinux docker-engine
yum install -y yum-utils device-mapper-persistent-data lvm2

wget -O /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/centos/docker-ce.repo

sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo

yum makecache fast
yum -y install docker-ce
docker version             #检验是否安装成功
```

> 然后因为在国内嘛，总会有一系列原因导致镜像拉去失败或者速度过慢，我们来配置阿里的镜像加速

首先找到自己专属的加速地址：[阿里云镜像仓库加速](<https://cr.console.aliyun.com/?spm=5176.doc60750.2.3.AzdSXF#/accelerator>)

```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://6voein3j.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 基础命令

#### 镜像操作

```shell
#搜索可用的docker镜像，输出信息包括镜像名字、描述、星级、是否为官方创建、是否自动创建）
docker search centos
docker pull centos             #获取镜像,格式为docker pull [OPTIONS] NAME[:TAG|@DIGEST]
                               #默认为latest 可以去官方仓库看tag
docker images                  #列出本地镜像 
                               #REPOSITORY：表示镜像的仓库源 TAG：镜像的标签                                                #IMAGE ID：镜像 IDCREATED：镜像创建时间 SIZE：镜像大小
docker inspect 458ff0e55eef    #列出镜像详细信息
docker rmi 458ff0e55eef        #删除镜像
docker system df               #查看镜像、容器、数据卷所占用的空间
#更新镜像，首先要对镜像启动的容器进行更改，然后进行更新镜像
docker commit --author "wuchen" --message "我就测试一下更新" webserver nginx:v2
#--author    作者
#--message   修改内容
#webserver   容器名
#nginx:v2    版本
```

#### 容器操作

```shell
docker run -d -P #容器后台运行，容器内部使用的端口映射到外部（不指定端口的话外部是随机端口）
#-i：交互式操作 -t：终端 以centos为镜像 --rm 退出容器之后删除 bash是交互式shell
docker run -it centos --rm centos bash
exit #退出容器
docker ps -a                     #查看启动的容器
docker port 273695360ee5         #查看指定容器映射的端口
docker logs -f 273695360ee5      #查看容器日志  -f等于tail -f
#使用nginx启动一个名字为webserver的容器 并且映射了80端口
docker run --name webserver -d -p 80:80 nginx   #-p是指定映射端口
docker exec -it webserver bash   #进入已启动的容器
docker diff webserver            #查看对容器的修改
docker top  273695360ee5         #查看容器的进程
docker inspect   273695360ee5    #查看容器详细的配置信息和状态，返回json字符串
docker stop 273695360ee5         #停止容器
docker ps -l                     #查询最后一次创建的容器
docker restart 273695360ee5      #重启容器
docker rm 273695360ee5           #删除容器，容器必须是停滞状态才能删除
```

#### 镜像仓库操作

> 因为在国内，公有镜像仓库我选择阿里云

首先在阿里云上创建一个容器镜像服务，然后创建一个镜像仓库

```shell
#登陆到镜像仓库
sudo docker login --username="阿里云账号" registry.cn-hangzhou.aliyuncs.com -p "密码"
#从镜像仓库拉去镜像
sudo docker pull registry.cn-hangzhou.aliyuncs.com/镜像仓库:[镜像版本号]
#登陆到镜像仓库
sudo docker login --username="阿里云账号" registry.cn-hangzhou.aliyuncs.com -p "密码"
#重命名镜像
sudo docker tag [ImageId] registry.cn-hangzhou.aliyuncs.com/镜像仓库:[镜像版本号]
#推送镜像到镜像仓库
sudo docker push registry.cn-hangzhou.aliyuncs.com/镜像仓库:[镜像版本号]
```


