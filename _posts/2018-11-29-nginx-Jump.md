---
layout: post
title: 'nginx二级目录跳转和LDAP认证'
date: 2018-11-29
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/11-29/FrankfurtXmas_ZH-CN9289866662_1920x1080.jpg'
tags: nginx LDAP
---

# nginx二级目录跳转 和 LDAP认证

> 使用kubernetes之后，运维自己这边的项目，如果每个都映射一个公网IP的话，不是那么的友好，所以通过nginx 二级目录的形式，来跳转一下
>
> 因为阿里云slb这边不支持验证，公网不安全，所以搞了一个ldap认证

**注意 后端服务要在同一个namespace下**

图将就看，大概就这样

![1543493091172](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/11-29/1543493091172.png)



## 镜像准备

> 官方的nginx镜像，不带ldap的模块，所以要自己做一个镜像
>
> 模块地址 https://github.com/kvspb/nginx-auth-ldap
>
> 编译的时候 --add-module=/clzdata/apps/nginx-auth-ldap

略过~~~~

## 配置文件

> 分开写比较美观

**nginx.conf**

```shell
user  nginx;
worker_processes  1;
daemon off;
error_log  /var/log/nginx_error.log warn;
pid        /var/run/nginx.pid;

events {
	use epoll;
	worker_connections  20480;
	multi_accept on;
}http {
	include        /clzdata/apps/nginx/conf/mime.types;
	default_type  application/octet-stream;

	log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
	'$status $body_bytes_sent "$http_referer" '
	'"$http_user_agent" "$http_x_forwarded_for"';

	access_log  /var/log/nginx_access.log  main;

	underscores_in_headers on;

	sendfile            on;
	tcp_nopush          on;
	tcp_nodelay         on;
	keepalive_timeout   65;
	types_hash_max_size 2048;

	ldap_server openldap {
		url ldap://x.x.x.x:389/dc=blog-wuchen,dc=cn?uid?sub?(&(objectClass=*)); #地址
		binddn "cn=Manager,dc=blog-wuchen,dc=cn";        #用户名
		binddn_passwd "000000";                  #密码
		group_attribute memberuid;
		group_attribute_is_dn on;
		require valid_user;
	}
	
	include /clzdata/apps/nginx/conf.d/*.conf;
	
}
```

**以kibana为例**

> kibana这边要改一下默认的访问路径，因为我的nginx功力不够。还因为kibana自己也会跳- -

**kibana.yml**

```y&#39;m
server.name: kibana
server.host: "0"
elasticsearch.url: http://elasticsearch-svc:9200
xpack.monitoring.ui.container.elasticsearch.enabled: true
server.basePath: "/elk"             #默认路径
server.rewriteBasePath: true        #更改
```

**kibana.conf**

```shell
server {
	listen       80 default_server;
	listen       [::]:80 default_server;
	server_name  localhost;
	root         /usr/share/nginx/html;

	client_max_body_size 2048m;
	access_log  /var/log/elk_access.log;
	error_log   /var/log/elk_error.log;

	location /elk/ {                      #注意后面的/ 带上的话 路径删除由nginx自动完成
		proxy_pass http://kibana-svc:5601;         #代理的地址
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection 'upgrade';
		proxy_set_header Host $host;
		proxy_cache_bypass $http_upgrade;
		auth_ldap "Restricted Space";          # 
		auth_ldap_servers openldap;            #  使用刚才设置的ldap服务
	}

	location /zipkin/ {
		proxy_pass http://zipkin-svc:9411;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection 'upgrade';
		proxy_set_header Host $host;
		proxy_cache_bypass $http_upgrade;
		auth_ldap "Restricted Space";
		auth_ldap_servers openldap;
	}

	error_page 404 /404.html;
	location = /40x.html {
	}

	error_page 500 502 503 504 /50x.html;
	location = /50x.html {
	}


```

## 挂载

```yaml
          volumeMounts:
            - mountPath: /clzdata/apps/nginx/conf/nginx.conf
              name: volume-1543481578799
              subPath: nginx.conf             #以文件的形式挂载
            - mountPath: /clzdata/apps/nginx/conf.d
              name: volume-1543481580150
```

## Ingress

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/service-weight: 'nginx-gateway: 100'
  generation: 6
  name: nginx-gateway
  namespace: 你的namespace
spec:
  rules:
    - host: clz.blog-wuchen.cn
      http:
        paths:
          - backend:
              serviceName: nginx-gateway
              servicePort: 80
            path: /
```



## 效果

> 嗯 emmmm

![1543492480242](https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/11-29/1543492480242.png)