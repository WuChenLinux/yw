---
layout: post
title: 'nginx解读'
date: 2018-10-29
author: 邬晨
color: rgb(255,210,32)
cover: 'https://wuchen-1252812685.cos.ap-shanghai.myqcloud.com/img/10-29-pxe/GeorgiaAquarium_ZH-CN12748518316_1920x1080.jpg'
tags: nginx
---

# nginx.conf 注释

> 阿里云的slb用的惯了，但是底层也是nginx的
>
> 所以在这里记录一下

```shell
#运行用户
user www www;
#启动进程
worker_processes auto;
worker_cpu_affinity auto;

#worker_processes 4 （可以设置成核心数，不过要设置worker_cpu_affinity）
#worker_cpu_affinity 0001 0010 0100 1000

#全局错误日志及PID文件
#(error_log 级别分为 debug, info, notice, warn, error, crit 默认为crit.
#crit 记录的日志最少，而debug记录的日志最多。如果你的nginx遇到一些问题，比如502比较频繁出现，但是看默认的error_log并没有看到有意义的信息，那么就可以调一下错误日志的级别，错误日志记录的内容会更加丰富。)
error_log /data/wwwlogs/error_nginx.log crit;

#pid文件存放路径
pid /var/run/nginx.pid;

#这个指令是指当一个nginx进程打开的最多文件描述符数目，理论值应该是最多打开文件数（ulimit -n）与nginx进程数相除，但是nginx分配请求并不是那么均匀，所以最好与ulimit -n的值保持一致。文件资源限制的配置可以在/etc/security/limits.conf设置，针对root/user等各个用户或者*代表所有用户来设置。
#* soft nofile 65535
#* hard nofile 65535
#用户重新登录生效（ulimit -n）
worker_rlimit_nofile 65535;

#工作模式及连接数上限
events {
use epoll;                #epoll是多路复用IO(I/O Multiplexing)中的一种方式,
worker_connections 65535; #单个后台worker process进程的最大并发链接数
multi_accept on;
}

#nginx采用epoll事件模型，处理效率高
#work_connections是单个worker进程允许客户端最大连接数，这个数值一般根据服务器性能和内存来制定，实际最大值就是worker进程数乘以work_connections，即是 worker_processes 和 worker_connections 的乘积
#实际我们填入一个65535，足够了，这些都算并发值，一个网站的并发达到这么大的数量，也算一个大站了！
#multi_accept 告诉nginx收到一个新连接通知后接受尽可能多的连接，默认是on，设置为on后，多个worker按串行方式来处理连接，也就是一个连接只有一个worker被唤醒，其他的处于休眠状态，设置为off后，多个worker按并行方式来处理连接，也就是一个连接会唤醒所有的worker，直到连接分配完毕，没有取得连接的继续休眠。当你的服务器连接数不多时，开启这个参数会让负载有一定的降低，但是当服务器的吞吐量很大时，为了效率，可以关闭这个参数。

http {
include mime.types;             #媒体类型,include 只是一个在当前文件中包含另一个文件内容的指令
default_type application/octet-stream;    #默认媒体类型足够
server_names_hash_bucket_size 128;        #等于hash表的大小，并且是一路处理器缓存大小的倍数。
client_header_buffer_size 32k;            #它为请求头分配一个缓冲区。 如果请求头大小大于指定的缓冲区，则使用large_client_header_buffers指令分配更大的缓冲区
large_client_header_buffers 4 32k;        #用于读取大型客户端请求头的缓冲区的最大数量和大小。 这些缓冲区仅在缺省缓冲区不足时按需分配。 当处理请求或连接转换到保持活动状态时，释放缓冲区。
client_max_body_size 1024m;               #NGINX能处理的最大请求主体大小。 如果请求大于指定的大小，则NGINX发回HTTP 413（Request Entity too large）错误。 如果服务器处理大文件上传，则该指令非常重要。
client_body_buffer_size 10m;              #用于请求主体的缓冲区大小。 如果主体超过缓冲区大小，则完整主体或其一部分将写入临时文件。 如果NGINX配置为使用文件而不是内存缓冲区，则该指令会被忽略。 默认情况下，该指令为32位系统设置一个8k缓冲区，为64位系统设置一个16k缓冲区
sendfile on;                              #开启高效文件传输模式，sendfile指令指定nginx是否调用sendfile函数来输出文件，对于普通应用设为 on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络I/O处理速度，降低系统的负载。 注意：如果图片显示不正常把这个改成off。
tcp_nopush on;                            #必须在sendfile开启模式才有效，防止网路阻塞，积极的减少网络报文段的数量（将响应头和正文的开始部分一起发送，而不一个接一个的发送。）
keepalive_timeout 120;                    #客户端连接保持会话超时时间，超过这个时间，服务器断开这个链接
client_header_timeout 10;                 #设置客户端请求头读取超时时间。如果超过这个时间，客户端还没有发送任何数据，Nginx将返回“Request time out(408)”错误。
client_body_timeout 10;                  #设置客户端请求主体读取超时时间。如果超过这个时间，客户端还没有发送任何数据，Nginx将返回“Request time out（408）”错误，默认值是60。
send_timeout        10;                  #设定响应客户端的超时时间。这个超时仅限于两个链接活动之间的时间，如果超过这个时间，客户端没有任何活动，Nginx将会关闭连接。

server_tokens off;                        #隐藏版本号
tcp_nodelay on;                           #也是防止网络阻塞，不过要包涵在keepalived参数才有效
autoindex off;                            #开启目录列表访问，合适下载服务器，默认关闭。

#nginx 为了快速处理静态数据集，比如：server names、map 指令的参数值、MIME 类型、请求首部字符串的名字等，采用了 hash 表来处理。nginx 在启动和重新加载配置时，会选择可能的最小的 hash 表大小，这是为了在存储拥有相同 hash 值的 key 的时候，使得 bucket size 不超过配置的上限值。
#hash 表的大小以 bucket 来表示。在 hash 表大小超过 hash max size 参数的设置之前，bucket 能够自动调整大小。大多数 hash 值有对应的配置指令，比如对于 server name hash，对应的指令为：server_names_hash_max_size 和 server_names_hash_bucket_size.
#server_names_hash_max_size 表示支持的 server name 总数；
#server_names_hash_bucket_size 表示 server name 的字符串长度上限值
#hash bucket size 的大小需要对齐 CPU 的 cache line size，它必须是 cache line size 的整数倍。这样设置，能够使现代处理器在进行 key 检索时减少内存访问次数从而提升检索速度。
#如果 hash bucket size 等于 cache line size，那么在进行 key 检索时所需要的访问内存次数最多需要 2 次。第一次，是计算 bucket address；第二次，是为了在 bucket 中进行 key 检索。因此，如果 nginx 给出消息要求增加 hash max size 或者 hash bucket size 时，应该首先增加 hash max size 的值。



fastcgi_connect_timeout 300;              #指定连接到后端FastCGI的超时时间
fastcgi_send_timeout 300;                 #向FastCGI传送请求的超时时间
fastcgi_read_timeout 300;                 #指定接收FastCGI应答的超时时间
fastcgi_buffer_size 64k;                  #指定读取FastCGI应答第一部分需要用多大的缓冲区，默认的缓冲区大小为fastcgi_buffers指令中的每块大小，可以将这个值设置更小。
fastcgi_buffers 4 64k;                    #指定本地需要用多少和多大的缓冲区来缓冲FastCGI的应答请求，如果一个php脚本所产生的页面大小为256KB，那么会分配4个64KB的缓冲区来缓存，如果页面大小大于256KB，那么大于256KB的部分会缓存到fastcgi_temp_path指定的路径中，但是这并不是好方法，因为内存中的数据处理速度要快于磁盘。一般这个值应该为站点中php脚本所产生的页面大小的中间值，如果站点大部分脚本所产生的页面大小为256KB，那么可以把这个值设置为“8 32K”、“4 64k”等。
fastcgi_busy_buffers_size 128k;           #建议设置为fastcgi_buffers的两倍，繁忙时候的buffer
fastcgi_temp_file_write_size 128k;        #在写入fastcgi_temp_path时将用多大的数据块，默认值是fastcgi_buffers的两倍，该数值设置小时若负载上来时可能报502BadGateway
fastcgi_intercept_errors on;              #这个指令指定是否传递4xx和5xx错误信息到客户端，或者允许nginx使用error_page处理错误信息。 注：静态文件不存在会返回404页面，但是php页面则返回空白页！！

#Gzip Compression
gzip on;                                  #开启压缩功能
gzip_buffers 16 8k;                       #压缩缓冲区大小，表示申请4个单位为32K的内存作为压缩结果流缓存，默认值是申请与原始数据大小相同的内存空间来存储gzip压缩结果
gzip_comp_level 6;                        #压缩比例，用来指定GZIP压缩比，1压缩比最小，处理速度最快，9压缩比最大，传输速度快，但是处理慢，也比较消耗CPU资源。
gzip_http_version 1.1;                    #压缩版本，用于设置识别HTTP协议版本，默认是1.1，目前大部分浏览器已经支持GZIP解压，使用默认即可
gzip_min_length 256;                      #设置允许压缩的页面最小字节数，页面字节数从header头的Content-Length中获取，默认值是0，不管页面多大都进行压缩，建议设置成大于1K，如果小与1K可能会越压越大。
gzip_proxied any;                         #nginx 做前端代理时启用该选项，表示无论后端服务器的headers头返回什么信息，都无条件启用压缩
gzip_vary on;                             #和http头有关系，加个vary头，给代理服务器用的，有的浏览器支持压缩，有的不支持。因此，为避免浪费不支持的也压缩，需要根据客户端的HTTP头来判断，是否需要压缩。
gzip_types                                #匹配mime类型进行压缩，无论是否指定,”text/html”类型总是会被压缩的。
text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml
text/javascript application/javascript application/x-javascript
text/x-json application/json application/x-web-app-manifest+json
text/css text/plain text/x-component
font/opentype application/x-font-ttf application/vnd.ms-fontobject
image/x-icon;
gzip_disable "MSIE [1-6]\.(?!.*SV1)";            #禁用IE 6 gzip

#curl -I -H "Accept-Encoding: gzip, deflate" "https://www.blog-wuchen.cn" //查看是否开启gzip压缩
#HTTP/1.1 200 OK
#Server: nginx
#Date: Thu, 30 Nov 2017 01:06:46 GMT
#Content-Type: text/html; charset=UTF-8
#Connection: keep-alive
#Vary: Accept-Encoding
#Link: <https://www.blog-wuchen.cn/wp-json/>; rel="https://api.w.org/"
#Strict-Transport-Security: max-age=15768000
#Content-Encoding: gzip

#If you have a lot of static files to serve through Nginx then caching of the files' metadata (not the actual files' contents) can save some latency.
open_file_cache max=1000 inactive=20s;           #打开文件描述符，其大小和修改时间;
open_file_cache_valid 30s;                       #设置一个时间，之后应该检查open_file_cache元素的源文件是否有变化。
open_file_cache_min_uses 2;                      #设置由open_file_cache指令的非活动参数配置的时间段内文件访问的最小数量，文件描述符在缓存中保持打开所需。例如配置数量为1，则源文件在访问至少一次以后则会将源文件进行缓存
open_file_cache_errors on;                   #启用或禁用文件查找错误的缓存open_file_cache.

server {
listen 80;                                   #监听端口
server_name _;
server_tokens off;                          #隐藏版本号
access_log /data/wwwlogs/access_nginx.log combined;          #日志位置与日志级别
root /data/wwwroot/default;                                  #网站路径
index index.html index.htm index.php;                    #访问一个文件夹，先找html，再找htm，再找php
#error_page 404 /404.html;                              #404页面 网站根目录下的404.html
#error_page 502 /502.html;                              #502页面 网站根目录下的502.html
location /nginx_status {
stub_status on;                                         #/启用nginx status配置
access_log off;
allow 127.0.0.1;                                        #允许的IP
deny all;
}

#curl http://127.0.0.1/nginx-status
#Active connections: 11921                             #active connections – 活跃的连接数量
#server accepts handled requests                     #总共处理了11989个连接 , 成功创建11989次握手, 总共处理了11991个请求
# 11989 11989 11991
#Reading: 56 Writing: 127 Waiting: 242             //reading — 读取客户端的连接数 writing — 响应数据到客户端的数量 waiting — 开启 keep-alive 的情况下,这个值等于 active – (reading+writing), 意思就是 Nginx 已经处理完正在等候下一次请求指令的驻留连接。所以,在访问效率高,请求很快被处理完毕的情况下,Waiting数比较多是正常的.如果reading +writing数较多,则说明并发访问量非常大,正在处理过程中。

location ~ [^/]\.php(/|$) {
#fastcgi_pass remote_php_ip:9000;                 #Nginx连接fastcgi的方式有2种：unix domain socket和TCP，Unix domain socket 或者 IPC socket是一种终端，可以使同一台操作系统上的两个或多个进程进行数据通信。与管道相比，Unix domain sockets 既可以使用字节流和数据队列，而管道通信则只能通过字节流。Unix domain sockets的接口和Internet socket很像，但它不使用网络底层协议来通信。Unix domain socket 的功能是POSIX操作系统里的一种组件。
fastcgi_pass unix:/dev/shm/php-cgi.sock;
fastcgi_index index.php;
include fastcgi.conf;
}
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|flv|mp4|ico)$ {         #图片缓存时间设置
expires 30d;                                          #保存以上格式到本地，保存30天
access_log off;                                       #不记录日志
}
location ~ .*\.(js|css)?$ {                           #JS和CSS缓存时间设置
expires 7d;                                           #保存以上格式到本地，保存7天
access_log off;                                       #不记录日志
}
location ~ /\.ht {
deny all;                                             #禁止访问htaccess
}
}
include vhost/*.conf;                                #虚拟主机配置文件存放位置
}

#设定虚拟主机配置
server {
listen 80 ;                                         #监听端口
listen 443 ssl http2;                               #监听443端口，开启ssl，开启http2
ssl_certificate /usr/local/nginx/conf/ssl/www.blog-wuchen.cn.crt;        #证书公钥
ssl_certificate_key /usr/local/nginx/conf/ssl/www.blog-wuchen.cn.key;    #证书私钥
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;                                     #隐性默认是SSLv3 TLSv1 TLSv1.1 TLSv1.2 这样配置把SSLv3关闭了
ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5; #选择加密套件，不同的浏览器所支持的套件（和顺序）可能会不同。这里指定的是OpenSSL库能够识别的算法
ssl_prefer_server_ciphers on;                                 #设置协商加密算法时,优先使用我们服务端的加密套件,而不是客户端浏览器的加密套件
ssl_session_timeout 10m;                                      #客户端可以重用会话缓存中ssl参数的过期时间 十分钟
ssl_session_cache builtin:1000 shared:SSL:10m;           #设置ssl/tls会话缓存的类型和大小 shared:SSL:10m表示我所有的nginx工作进程共享ssl会话缓存
ssl_buffer_size 1400;                                    #默认情况下, 缓冲区大小为 16k, 它对应于发送大响应时的最小开销 这里的1400是MTU
add_header Strict-Transport-Security max-age=15768000;   #添加 header头 告诉浏览器网站使用 HTTPS 访问，支持HSTS的浏览器
ssl_stapling on;                                       #指定客户端可以重用会话参数的时间。
ssl_stapling_verify on;                                #启用或禁用服务器对 OCSP 响应的验证。
server_name www.blog-wuchen.cn;                        #网站域名
access_log /data/wwwlogs/www.blog-wuchen.cn_nginx.log combined;   #日志位置与日志级别
index index.html index.htm index.php;                  #定义首页索引文件的名称
root /data/wwwroot/www.blog-wuchen.cn;                 #网站根目录
if ($ssl_protocol = "") { return 301 https://$host$request_uri; }  #http301跳转https

# if ($host = "106.14.204.61") { return 301 https://$host$request_uri; }

if ($host = '106.14.204.61' ) { return 403; }             #IP访问直接403
include /usr/local/nginx/conf/rewrite/wordpress.conf;     #伪静态文件
#error_page 404 /404.html;
#error_page 502 /502.html;
location ~ [^/]\.php(/|$) {                               #php以sock方式提供服务
#fastcgi_pass remote_php_ip:9000;
fastcgi_pass unix:/dev/shm/php-cgi.sock;
fastcgi_index index.php;
include fastcgi.conf;
}
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|flv|mp4|ico)$ {       #图片缓存
expires 30d;
access_log off;
}
location ~ .*\.(js|css)?$ {                                    #js css缓存
expires 7d;
access_log off;
}
location ~ /\.ht {                                            #禁止访问htaccess
deny all;
}
}

#nginx的upstream目前支持4种方式的分配
#1、轮询（默认）
#每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。
#2、weight
#指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。
#例如：
upstream bakend {
  server 192.168.0.14 weight=10;
  server 192.168.0.15 weight=10;
}
#2、ip_hash
#每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。
#例如：
upstream bakend {
  ip_hash;
  server 192.168.0.14:88;
  server 192.168.0.15:80;
}
#3、fair（第三方）
#按后端服务器的响应时间来分配请求，响应时间短的优先分配。
upstream backend {
  server server1;
  server server2;
  fair;
}
#4、url_hash（第三方）
#按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效。
#例：在upstream中加入hash语句，server语句中不能写入weight等其他的参数，hash_method是使用的hash算法
upstream backend {
  server squid1:3128;
  server squid2:3128;
  hash $request_uri;
  hash_method crc32;
}

#tips:
upstream bakend{#定义负载均衡设备的Ip及设备状态}{
  ip_hash;
  server 127.0.0.1:9090 down;
  server 127.0.0.1:8080 weight=2;
  server 127.0.0.1:6060;
  server 127.0.0.1:7070 backup;
}
#在需要使用负载均衡的server中增加 proxy_pass http://bakend/;

#每个设备的状态设置为:
#1.down表示单前的server暂时不参与负载
#2.weight为weight越大，负载的权重就越大。
#3.max_fails：允许请求失败的次数默认为1.当超过最大次数时，返回proxy_next_upstream模块定义的错误
#4.fail_timeout:max_fails次失败后，暂停的时间。
#5.backup： 其它所有的非backup机器down或者忙的时候，请求backup机器。所以这台机器压力会最轻。

#nginx支持同时设置多组的负载均衡，用来给不用的server来使用。
#client_body_in_file_only设置为On 可以讲client post过来的数据记录到文件中用来做debug
#client_body_temp_path设置记录文件的目录 可以设置最多3层目录
#location对URL进行匹配.可以进行重定向或者进行新的代理 负载均衡

server
{
listen 80;                         #监听端口

server_name www.blog-wuchen.cn blog-wuchen.cn     #域名可以有多个，用空格隔开
index index.html index.htm index.php;
root /data/wwwroot/www.blog-wuchen.cn;            #网站根目录
location ~ .*\.(php|php5)?$
{
fastcgi_pass 127.0.0.1:9000;
fastcgi_index index.php;
include fastcgi.conf;
}

location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$        #图片缓存时间设置
{
expires 10d;
}

location ~ .*\.(js|css)?$                         #JS和CSS缓存时间设置
{
expires 1h;
}
#日志格式设定
log_format access '$remote_addr - $remote_user [$time_local] "$request" '
'$status $body_bytes_sent "$http_referer" '
'"$http_user_agent" $http_x_forwarded_for';
#定义本虚拟主机的访问日志
access_log /var/log/nginx/ha97access.log access;
#对 "/" 启用反向代理
location / {
proxy_pass http://127.0.0.1:88;
proxy_redirect off;
proxy_set_header X-Real-IP $remote_addr;
#后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#以下是一些反向代理的配置，可选。
proxy_set_header Host $host;
client_max_body_size 10m; #允许客户端请求的最大单文件字节数
client_body_buffer_size 128k; #缓冲区代理缓冲用户端请求的最大字节数，
proxy_connect_timeout 90; #nginx跟后端服务器连接超时时间(代理连接超时)
proxy_send_timeout 90; #后端服务器数据回传时间(代理发送超时)
proxy_read_timeout 90; #连接成功后，后端服务器响应时间(代理接收超时)
proxy_buffer_size 4k; #设置代理服务器（nginx）保存用户头信息的缓冲区大小
proxy_buffers 4 32k; #proxy_buffers缓冲区，网页平均在32k以下的设置
proxy_busy_buffers_size 64k; #高负荷下缓冲大小（proxy_buffers*2）
proxy_temp_file_write_size 64k;
#设定缓存文件夹大小，大于这个值，将从upstream服务器传
}
#设定查看Nginx状态的地址
location /NginxStatus {
stub_status on;
access_log on;
auth_basic "NginxStatus";
auth_basic_user_file conf/htpasswd;
#htpasswd文件的内容可以用apache提供的htpasswd工具来产生。
}
#本地动静分离反向代理配置
#所有jsp的页面均交由tomcat或resin处理
location ~ .(jsp|jspx|do)?$ {
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_pass http://127.0.0.1:8080;
}
#所有静态文件由nginx直接读取不经过tomcat或resin
location ~ .*.(htm|html|gif|jpg|jpeg|png|bmp|swf|ioc|rar|zip|txt|flv|mid|doc|ppt|pdf|xls|mp3|wma)$
{ expires 15d; }
location ~ .*.(js|css)?$
{ expires 1h; }
}
}
```

