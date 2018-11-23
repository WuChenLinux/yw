---
layout: post
title: 'SaltStack的安装与应用'
date: 2018-10-18
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/10-18/LeGivre_EN-AU7576437900_1920x1080.jpg'
tags: SaltStack
---

# SaltStack的安装与应用

> 因为日常工作中常用到saltstack，所以记录一下，以后用来查看也好

## 目录

- [环境准备](#环境准备)
  - [系统环境](#系统环境)
  - [环境部署](#环境部署)
  - [基础命令](#基础命令)
  - [配置sls](#配置sls)


## 环境准备

### 系统环境

| 主机名     | 系统版本 | 内网IP        | 安装软件              | 备注               |
| ---------- | -------- | ------------- | --------------------- | ------------------ |
| zabbix     | centos7  | 192.168.1.110 | salt-master，salt-api | 防火墙,selinux关闭 |
| nginx-web1 | centos7  | 192.168.1.10  | salt-minion           | 防火墙,selinux关闭 |
| nginx-web2 | centos7  | 192.168.1.11  | salt-minion           | 防火墙,selinux关闭 |

### 环境部署

> 因为自带的centos7的yum源里没有saltstack的包，所以安装一下epel源，也可以安装[官网源](https://repo.saltstack.com/#rhel)来安装

```shell
#三台都安装epel源
yum -y install epel-release
#在master节点安装salt-master和salt-api
yum -y install salt-master.noarch
yum -y install salt-api
#在客户端节点安装salt-minion
yum -y install salt-minion.noarch
```

> 修改master的配置文件，启动服务端并设置开机自启

```shell
vim /etc/salt/master
#这里只写用到的，其他的默认就好，用到再改
interface: 192.168.1.110                    #监听地址，默认0.0.0.0
auto_accept: True                           #可以自动接受客户端发来key
file_roots:                                 #文件服务器传送时，在master的工作目录，可以有多个
  base:
    - /srv/salt
pillar_roots:                               #设置用于保存sls文件的环境和目录
  base:
    - /srv/pillar
nodegroups:                                 #可以设置逻辑分组
  nginx: 'L@nginx-web1，nginx-web2'
  
  
systemctl restart salt-master
systemctl enable salt-master
```

> 修改minion的配置文件，启动客户端并设置开机自启

```shell
 vim /etc/salt/minion
 #这里只写用到的，其他的默认就好，用到再改
 master: 192.168.1.110                      #服务端地址
 id: nginx-web1                             #可以写，不写的话默认是主机名

systemctl restart salt-minion
systemctl enable salt-minion
```

### 基础命令

```shell
salt-key                        #密钥管理
salt-key -L                     #查看所有key
salt-key -a <key-name>          #接受某个客户端的key
salt-key -A                     #接受所有客户端的key
salt-key -d <key-name>          #删除某个客户端的key
salt-key -D                     #删除所有客户端的key
```
常用命令：salt [options] '<target>' <function> [arguments]

```shell
salt '*' test.ping                     #测试与客户端的连接性

salt -G 'os:centos' test.ping           #-G表示用节点属性（grains.item）来匹配
salt -E 'nginx-web[0-9]' test.ping      #-E表示按正则表达式匹配
salt -N 'nginx' test.ping               #-N表示按组名匹配（组的定义在/etc/salt/master里面   
salt -L 'nginx-web01' test.ping         #-L表示主机名匹配

salt 'pro-yw-sonar01' grains.ls         #查看grains分类
salt 'pro-yw-sonar01' grains.items      #查看grains的所有属性
salt 'pro-yw-sonar01' grains.item shell #查看单独的属性 

salt '*' cmd.run 'df -hT'               #cmd.run是远程执行shell命令，例如df hostname等等
salt-cp '*' 1.sh /root/                 #复制文件(可相对路径绝对路径)到客户端的/root/目录下，不支持目录
salt '*' cp.get_dir salt://test_dir /tmp gzip=9 #分发目录，并启用压缩 
salt '*' cp.get_file salt://1.txt /tmp/srv/1.txt makedirs=True #当分发的在目标主机上不存在时，自动创建该目录
salt-run manage.status               #检查所有客户端的up/down状态

```

### 配置sls

> 使用小巧，易读，易于理解的配置文件，网上的解释很多了，我最常用的还是文件操作，所有模块详见[官网](https://docs.saltstack.com/en/latest/ref/states/all/)

```shell
cd /srv/salt                            #配置的sls路径
```



```shell
vim scripts.sls
#同步文件夹
scripts_dir:                                        #自定义，别的配置段可以引用
  file.recurse:                                     #递归目录管理
    - name: /clzdata/scripts                        #客户端地址
    - source: salt://data/SystemScripts/scripts/    #服务端源地址
    - user: opsuser                                 #客户端文件夹的用户
    - group: opsuser                                #客户端文件夹的组用户
    - file_mode: 555                                #客户端文件的权限
    - dir_mode: 555                                 #客户端文件夹的权限
    - mkdir: True                                   #是否创建客户端文件夹
    - clean: True                                   #源删除文件或目录,目标也会跟着删除

salt "pre-ca-offical01" state.sls scripts test=True #test=True 是指测试安装,不进行实际操作

#同步单个文件
supervisor:
   file.managed:                                    #从master节点下载文件
     - source: salt://system_config/supervisord.conf
     - name: /etc/supervisor/supervisord.conf
     - user: root
     - group: root
     - mode: 755
     - backup: minion                               #有改动会备份,备份到/var/cache/salt/minion/file_backup/目录下
  
```
> 用户操作

```shell
vim user_manage.sls
opsuser:
  group.present:                           #组模块，组可以存在可以不存在
    - gid: 500                             #分配给指定组的组ID
    - system: True
    - addusers:
      - user_wuc                           #添加用户到组，不能和members一起用
     - delusers:
       - user_wc                           #删除组内的用户，不能和member一起用

opsuser:
  group.present:                           #组模块，组可以存在可以不存在
    - gid: 500                             #分配给指定组的组ID
    - system: True
    - members:
      - user_wuc                           #替换组内的用户，


user_wuc:
  user.present:                             #创建和管理用户设置
    - shell: /bin/bash                      #用户的登陆shell
    - home: /home/user_wuc                  #用户的家目录
    - uid: 503                              #用户的UID
    - groups:                               #用户的附加组
      - opsuser
    - password:                             #设置密码
      - 000000               
```

