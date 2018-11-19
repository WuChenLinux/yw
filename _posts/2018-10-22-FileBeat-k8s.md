---
layout: post
title: 'filebeat配置'
date: 2018-10-22
author: 邬晨
color: rgb(255,210,32)
cover: 'https://bing.ioliu.cn/v1/rand?w=900&h=300'
tags: ELK
---

# filebeat-k8s

使用每台node节点部署一个filebeat来收集日志

官方的yaml，然后删除了点配置

服务将日志写到标准输出

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: test-t3-yw
  labels:
    k8s-app: filebeat
data:
  filebeat.yml: |-
    filebeat.config:
      inputs:
        # Mounted `filebeat-inputs` configmap:
        #默认是path.home=/usr/share/filebeat
        path: ${path.config}/inputs.d/*.yml
        # Reload inputs configs as they change:
        reload.enabled: false


    # To enable hints based autodiscover, remove `filebeat.config.inputs` configuration and uncomment this:
    #filebeat.autodiscover:
    #  providers:
    #    - type: kubernetes
    #      hints.enabled: true

    output.logstash:   #logstash的配置 直接输出到 也可以到kafka
      hosts: ['${LOGSTASH_HOST:logstash-svc}:${LOGSTASH_PORT:5044}']

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-inputs
  namespace: test-t3-yw
  labels:
    k8s-app: filebeat
data:
  kubernetes.yml: |-
    - type: docker
      containers.ids:
      - "*"              #从所有容器中读取，下面配置是向上合并
      multiline.pattern: '^\d{4}-'
      multiline.negate: true
      multiline.match: after
      processors:
        - add_kubernetes_metadata:
            in_cluster: true  #当作k8s的pod运行
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: test-t3-yw
  labels:
    k8s-app: filebeat
spec:
  template:
    metadata:
      labels:
        k8s-app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:6.4.2
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: LOGSTASH_HOST
          value: logstash-svc
        - name: LOGSTASH_PORT
          value: "5044"
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 512Mi
        volumeMounts:
        - name: config                 #生成一个config的配置文件，挂载到容器里
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: inputs                  #生成一个input的配置文件，挂载到容器里
          mountPath: /usr/share/filebeat/inputs.d
          readOnly: true
        - name: data                    #filebeat自身的数据挂载
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers  #日志路径，从这里收集
          mountPath: /var/lib/docker/containers
          readOnly: true
      imagePullSecrets:                 #私有仓库的话 这里是key
        - name: wuchen
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: inputs
        configMap:
          defaultMode: 0600
          name: filebeat-inputs
      # data folder stores a registry of read status for all files, so we don't send everything again on a Filebeat pod restart
      - name: data
        hostPath:
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: test-t3-yw
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  verbs:
  - get
  - watch
  - list
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: test-t3-yw
  labels:
    k8s-app: filebeat
---

```

后来发现开发有些日志并没有输出到控制台，然后定义了路径和日志格式，**把宿主机目录挂载到日志路径**

然后修改了下配置文件即可

```yaml
filebeat.prospectors:

- input_type: log
  paths:
    - /var/lib/container/logs/*/*.log     #宿主机目录
  multiline:
    pattern: ^\d{4}-
    negate: true
    match: after
    timeout: 2s

#filebeat.config:
#  inputs:
    # Mounted `filebeat-inputs` configmap:
#    path: ${path.config}/inputs.d/*.yml
    # Reload inputs configs as they change:
#    reload.enabled: false


# To enable hints based autodiscover, remove `filebeat.config.inputs` configuration and uncomment this:
#filebeat.autodiscover:
#  providers:
#    - type: kubernetes
#      hints.enabled: true

output.logstash:
  hosts: ['${LOGSTASH_HOST:logstash-svc}:${LOGSTASH_PORT:5044}']
```

