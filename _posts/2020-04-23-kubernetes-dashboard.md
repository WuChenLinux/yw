---
layout: post
title: 'kubernetes-dashboard'
date: 2020-04-23
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/2019-12-04/th.jpg'
tags: k8s
---

# kubernetes-dashboard

#### [查看dashboard官方yaml](https://github.com/kubernetes/dashboard/releases)

```bash
# 当前的最新版本 兼容k8s 1.18
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

#### 修改yaml

~~~shell
vim recommended.yaml
# 修改token登陆失效时间 args里添加
 - --token-ttl=0
~~~

#### 创建服务账户登录

这里创建了一个admi-user的服务账户绑定到cluster-admin的角色

~~~shell
vim admin-user.yaml
~~~

~~~yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system

~~~

#### 创建服务与账号

```shell
kubectl apply  -f recommended.yaml
kubectl apply  -f admin-user.yaml
   
#查看是否是running状态
kubectl get pods -n kubernetes-dashboard
```

#### dashboard多种访问方式及修改参数

- proxy代理访问（只允许127.0.0.1和localhost登录  不推荐)
- NodePort           (通过节点IP加NodePort端口访问   不推荐）
- api暴露（推荐）

~~~
# 首先找到kubectl命令的配置文件，默认情况下为/etc/kubernetes/admin.conf 我们在上一章已经把这文件拷贝到Jenkins用户的~/.kube/config中
#然后我们使用以下client-certficate-data和client-key-key生成一个p12文件

# 生成client-certificate-data
grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt

# 生成client-key-data
grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key

# 生成p12
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"

# 把文件导出到浏览器的证书里即可

~~~

#### 访问

> 浏览器打开

  ~~~http
https://172.27.83.23:6443/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login
  ~~~

  - 令牌登录方式

  ~~~bash
  # 获取令牌密码
  kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
  ~~~

  


