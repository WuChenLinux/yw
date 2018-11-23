---
layout: post
title: 'ZooKeeper四字命令'
date: 2018-10-19
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/10-19/EcolaSP_ZH-CN10746626161_1920x1080.jpg'
tags: ZooKeeper
---

# ZooKeeper 四字命令

> 四字命令查看Zookeeper运行相关信息,极其好用

### 使用方法
```shell
echo 'command'|nc ip port
```
 最小化系统安装是没有nc的

```shell
yum -y install nc
echo 'conf'|nc 127.0.0.1 2181
```

### 四字解释

```shell
conf:                                 
#输出详细的服务配置信息。
cons: 
#列出所有 客户端 链接到 服务端的 session 详细信息。包括所有 接收/发送 的信息包数，session Id，延迟的操作，最后执行的操作…
crst:
#重置对所有connection/session统计。
dump: 
#列出所有的未处理的会话和零时节点。这个命令只能使用在leader服务上。
envi:
#输出当前服务的详细信息
ruok:
#确认服务运行状态是否正常。如果服务正在运行，则回复”imok”。相反的服务将不会回应。返回了”imok”的服务并不一定表名服务是在集群内，只是说明服务器进程激活和绑定到了指定的客户端端口。使用”stat” 可以获取集群的组成情况和客户端的链接信息
srst:
#重置server统计
srvr:
#打印所有的服务器信息
stat:
#列出简略的服务信息和链接客户端信息
wchs:
#3.3.0版本中新增: 列出简略的服务器上的watches信息
wchc:
#按照session列出详细的服务器上的watches信息。输出的内容是sessions(connections) 和相关的watches(路径)。注意：在不同的watches的数量情况下，这个操作肯能很消耗性能，使用它要格外的小心。
wchp:
#按照path 列出详细的服务器上的watches信息。输出的内容是paths (znodes)和相关sessions。注意：在不同的watches的数量情况下，这个操作肯能很消耗性能，使用它要格外的小心。
mntr:
#输出可以用来监控集群健康情况的一系列变量。输出的是能兼容Java属性文件的格式，而且内容可能会随时间而改变（比如增加新的变量)
```

### 监控指标

下面我们来梳理一下返回了哪些监控指标：

```shell
conf:
clientPort:               #客户端端口号 
dataDir：                 #数据文件目录
dataLogDir：              #日志文件目录  
tickTime：                #间隔单位时间
maxClientCnxns：          #最大连接数  
minSessionTimeout：       #最小session超时
maxSessionTimeout：       #最大session超时  
serverId：                #id  
initLimit：               #初始化时间  
syncLimit：               #心跳时间间隔  
electionAlg：             #选举算法 默认3  
electionPort：            #选举端口  
quorumPort：              #法人端口  
peerType：                #未确认

cons:
ip=ip
port=端口
queued=所在队列
received=收包数
sent=发包数
sid=session id
lop=最后操作
est=连接时间戳
to=超时时间
lcxid=最后id(未确认具体id)
lzxid=最后id(状态变更id)
lresp=最后响应时间戳
llat=最后/最新 延时
minlat=最小延时
maxlat=最大延时
avglat=平均延时

crst:
重置所有连接

dump:
session id : znode path  (1对多   ,  处于队列中排队的session和临时节点)

envi:
zookeeper.version=版本
host.name=host信息
java.version=java版本
java.vendor=供应商
java.home=jdk目录
java.class.path=classpath
java.library.path=lib path
java.io.tmpdir=temp目录
java.compiler=<NA> 
os.name=Linux 
os.arch=amd64 
os.version=2.6.32-358.el6.x86_64
user.name=hhz
user.home=/home/hhz user.dir=/export/servers/zookeeper-3.4.6

ruok:                     #查看server是否正常
imok=正常

srst:                     #重置server状态

srvr：
Zookeeper version:版本
Latency min/avg/max: 延时
Received: 收包
Sent: 发包
Connections: 连接数
Outstanding: 堆积数
Zxid: 操作id
Mode: leader/follower
Node count: 节点数

stat：
Zookeeper version: 3.4.6-1569965, built on 02/20/2014 09:09 GMT Clients:
/192.168.147.102:56168[1](queued=0,recved=41,sent=41) 
/192.168.144.102:34378[1](queued=0,recved=54,sent=54) 
/192.168.162.16:43108[1](queued=0,recved=40,sent=40) 
/192.168.144.107:39948[1](queued=0,recved=1421,sent=1421) 
/192.168.162.16:43112[1](queued=0,recved=54,sent=54) 
/192.168.162.16:43107[1](queued=0,recved=54,sent=54) 
/192.168.162.16:43110[1](queued=0,recved=53,sent=53) 
/192.168.144.98:34702[1](queued=0,recved=41,sent=41) 
/192.168.144.98:34135[1](queued=0,recved=61,sent=65)
/192.168.162.16:43109[1](queued=0,recved=54,sent=54)
/192.168.147.102:56038[1](queued=0,recved=165313,sent=165314) 
/192.168.147.102:56039[1](queued=0,recved=165526,sent=165527) 
/192.168.147.101:44124[1](queued=0,recved=162811,sent=162812)
/192.168.147.102:39271[1](queued=0,recved=41,sent=41)
/192.168.144.107:45476[1](queued=0,recved=166422,sent=166423)
/192.168.144.103:45100[1](queued=0,recved=54,sent=54)
/192.168.162.16:43133[0](queued=0,recved=1,sent=0) 
/192.168.144.107:39945[1](queued=0,recved=1825,sent=1825) 
/192.168.144.107:39919[1](queued=0,recved=325,sent=325)
/192.168.144.106:47163[1](queued=0,recved=17891,sent=17891)
/192.168.144.107:45488[1](queued=0,recved=166554,sent=166555)
/172.17.36.11:32728[1](queued=0,recved=54,sent=54)
/192.168.162.16:43115[1](queued=0,recved=54,sent=54)
Latency min/avg/max: 0/0/599
Received: 224869 
Sent: 224817 
Connections: 23 
Outstanding: 0 
Zxid: 0x68000af707 
Mode: follower 
Node count: 101081 
（同上面的命令整合的信息）

wchs:
connectsions=连接数
watch-paths=watch节点数 
watchers=watcher数量 

wchc: 
session id 对应 path 

wchp: 
path 对应 session id 

mntr: 
zk_version=版本 
zk_avg_latency=平均延时 
zk_max_latency=最大延时 
zk_min_latency=最小延时
zk_packets_received=收包数
zk_packets_sent=发包数
zk_num_alive_connections=连接数 
zk_outstanding_requests=堆积请求数
zk_server_state=leader/follower 状态 
zk_znode_count=znode数量 
zk_watch_count=watch数量 
zk_ephemerals_count=临时节点（znode） 
zk_approximate_data_size=数据大小 
zk_open_file_descriptor_count=打开的文件描述符数量 
zk_max_file_descriptor_count=最大文件描述符数量 
zk_followers=follower数量 
zk_synced_followers=同步的follower数量 
zk_pending_syncs=准备同步数
```

