# nginx配置文件详解

#定义Nginx运行的用户和用户组user www www;

\#nginx进程数，建议设置为等于CPU总核心数。
worker_processes 8;

\#全局错误日志定义类型，[ debug | info | notice | warn | error | crit ]
error_log /var/log/nginx/error.log info;

\#进程文件
pid /var/run/nginx.pid;

\#一个nginx进程打开的最多文件描述符数目，理论值应该是最多打开文件数（系统的值ulimit -n）与nginx进程数相除，但是nginx分配请求并不均匀，所以建议与ulimit -n的值保持一致。
worker_rlimit_nofile 65535;

\#工作模式与连接数上限
events
{
\#参考事件模型，use [ kqueue | rtsig | epoll | /dev/poll | select | poll ]; epoll模型是Linux 2.6以上版本内核中的高性能网络I/O模型，如果跑在FreeBSD上面，就用kqueue模型。
use epoll;
\#单个进程最大连接数（最大连接数=连接数*进程数）
worker_connections 65535;
}

\#设定http服务器
http
{
include mime.types; #文件扩展名与文件类型映射表
default_type application/octet-stream; #默认文件类型
\#charset utf-8; #默认编码
server_names_hash_bucket_size 128; #服务器名字的hash表大小
client_header_buffer_size 32k; #上传文件大小限制
large_client_header_buffers 4 64k; #设定请求缓
client_max_body_size 8m; #设定请求缓
sendfile on; #开启高效文件传输模式，sendfile指令指定nginx是否调用sendfile函数来输出文件，对于普通应用设为 on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络I/O处理速度，降低系统的负载。注意：如果图片显示不正常把这个改成off。
autoindex on; #开启目录列表访问，合适下载服务器，默认关闭。
tcp_nopush on; #防止网络阻塞
tcp_nodelay on; #防止网络阻塞
keepalive_timeout 120; #长连接超时时间，单位是秒

``

\#FastCGI相关参数是为了改善网站的性能：减少资源占用，提高访问速度。下面参数看字面意思都能理解。
fastcgi_connect_timeout 300;
fastcgi_send_timeout 300;
fastcgi_read_timeout 300;
fastcgi_buffer_size 64k;
fastcgi_buffers 4 64k;
fastcgi_busy_buffers_size 128k;
fastcgi_temp_file_write_size 128k;

``

\#gzip模块设置
gzip on; #开启gzip压缩输出
gzip_min_length 1k; #最小压缩文件大小
gzip_buffers 4 16k; #压缩缓冲区
gzip_http_version 1.0; #压缩版本（默认1.1，前端如果是squid2.5请使用1.0）
gzip_comp_level 2; #压缩等级
gzip_types text/plain application/x-javascript text/css application/xml;
\#压缩类型，默认就已经包含text/html，所以下面就不用再写了，写上去也不会有问题，但是会有一个warn。
gzip_vary on;
\#limit_zone crawler $binary_remote_addr 10m; #开启限制IP连接数的时候需要使用

``

upstream blog.ha97.com {
\#upstream的负载均衡，weight是权重，可以根据机器配置定义权重。weigth参数表示权值，权值越高被分配到的几率越大。
server 192.168.80.121:80 weight=3;
server 192.168.80.122:80 weight=2;
server 192.168.80.123:80 weight=3;
}

``

\#虚拟主机的配置
server
{
\#监听端口
listen 80;
\#域名可以有多个，用空格隔开
server_name www.ha97.com ha97.com;
index index.html index.htm index.php;
root /data/www/ha97;
location ~ .*\.(php|php5)?$
{
fastcgi_pass 127.0.0.1:9000;
fastcgi_index index.php;
include fastcgi.conf;
}
\#图片缓存时间设置
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
{
expires 10d;
}
\#JS和CSS缓存时间设置
location ~ .*\.(js|css)?$
{
expires 1h;
}
\#日志格式设定
log_format access ‘$remote_addr – $remote_user [$time_local] “$request” ‘
‘$status $body_bytes_sent “$http_referer” ‘
‘”$http_user_agent” $http_x_forwarded_for’;
\#定义本虚拟主机的访问日志
access_log /var/log/nginx/ha97access.log access;

``

\#对 “/” 启用反向代理
location / {
proxy_pass http://127.0.0.1:88;
proxy_redirect off;
proxy_set_header X-Real-IP $remote_addr;
\#后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
\#以下是一些反向代理的配置，可选。
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
\#设定缓存文件夹大小，大于这个值，将从upstream服务器传
}

``

\#设定查看Nginx状态的地址
location /NginxStatus {
stub_status on;
access_log on;
auth_basic “NginxStatus”;
auth_basic_user_file conf/htpasswd;
\#htpasswd文件的内容可以用apache提供的htpasswd工具来产生。
}

``

```
#本地动静分离反向代理配置#所有jsp的页面均交由tomcat或resin处理location ~ .(jsp|jspx|do)?$ {proxy_set_header Host $host;proxy_set_header X-Real-IP $remote_addr;proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;proxy_pass http://127.0.0.1:8080;}#所有静态文件由nginx直接读取不经过tomcat或resinlocation ~ .*.(htm|html|gif|jpg|jpeg|png|bmp|swf|ioc|rar|zip|txt|flv|mid|doc|ppt|pdf|xls|mp3|wma)${ expires 15d; }location ~ .*.(js|css)?${ expires 1h; }}}
```

匹配符 匹配规则 

优先级 =    精确匹配    1

 ^~    以某个字符串开头    2

 ~    区分大小写的正则匹配    3 

~*    不区分大小写的正则匹配    4 

!~    区分大小写不匹配的正则    5 

!~*    不区分大小写不匹配的正则    6

 /    通用匹配，任何请求都会匹配到    **7**

**阅读目录**

- [一、Location语法优先级排列](https://www.cnblogs.com/WiseAdministrator/articles/11121653.html#_label0)
- [二、nginx.conf配置文件实例](https://www.cnblogs.com/WiseAdministrator/articles/11121653.html#_label1)
- [三、nginx语法之root和alias区别实战](https://www.cnblogs.com/WiseAdministrator/articles/11121653.html#_label2)

 

------

## 一、Location语法优先级排列

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

![复制代码](https://common.cnblogs.com/images/copycode.gif)

```
匹配符 匹配规则 优先级
=    精确匹配    1
^~    以某个字符串开头    2
~    区分大小写的正则匹配    3
~*    不区分大小写的正则匹配    4
!~    区分大小写不匹配的正则    5
!~*    不区分大小写不匹配的正则    6
/    通用匹配，任何请求都会匹配到    7
```

![复制代码](https://common.cnblogs.com/images/copycode.gif)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

## 二、nginx.conf配置文件实例

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

![复制代码](https://common.cnblogs.com/images/copycode.gif)

```
server {
    listen 80;
    server_name pythonav.cn;

    #优先级1,精确匹配，根路径
    location =/ {
        return 400;
    }

    #优先级2,以某个字符串开头,以av开头的，优先匹配这里，区分大小写
    location ^~ /av {
       root /data/av/;
    }

    #优先级3，区分大小写的正则匹配，匹配/media*****路径
    location ~ /media {
          alias /data/static/;
    }

    #优先级4 ，不区分大小写的正则匹配，所有的****.jpg|gif|png 都走这里
    location ~* .*\.(jpg|gif|png|js|css)$ {
       root  /data/av/;
        }

    #优先7，通用匹配
    location / {
        return 403;
    }
}
```

![复制代码](https://common.cnblogs.com/images/copycode.gif)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

## 三、nginx语法之root和alias区别实战

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

![复制代码](https://common.cnblogs.com/images/copycode.gif)

```
nginx指定文件路径有root和alias两种方法
区别在方法和作用域：

方法：

root
语法  root  路径;
默认值 root   html;
配置块  http{}   server {}   location{}

alias
语法： alias  路径
配置块  location{}


root和alias区别在nginx如何解释location后面的url，这会使得两者分别以不同的方式讲请求映射到服务器文件上

root参数是root路径+location位置

root实例：

    location ^~ /av {
        root /data/av;   注意这里可有可无结尾的   /
    }
请求url是pythonav.cn/av/index.html时
web服务器会返回服务器上的/data/av/av/index.html

root实例2：
location ~* .*\.(jpg|gif|png|js|css)$ {
       root  /data/av/;
}

请求url是pythonav.cn/girl.gif时
web服务器会返回服务器上的/data/static/girl.gif




alias实例：
alias参数是使用alias路径替换location路径
alias是一个目录的别名
注意alias必须有 "/"  结束！
alias只能位于location块中


请求url是pythonav.cn/av/index.html时
web服务器会返回服务器上的/data/static/index.html

location ^~ /av {
    alias /data/static/;
}
```

## 四、nginx location proxy_pass详解

```
在nginx中配置proxy_pass时，如果在proxy_pass后面的端口号后边（切记看的一定要是端口号的后边）加/，相当于是绝对根路径，则nginx不会把location中匹配的路径部分代理走;如果没有/，则会把匹配的路径部分给代理走.

2022/12/15理解：端口后边有/,会去除掉location匹配的内容，然后再追加去掉后的请求路径（请求路径是指端口之后的路径）例如：
/api/test   去掉 /api 则为 /test直接追加在proxy_pass后边


下面四种情况分别用http://106.12.74.123/abc/index.html进行访问。
# 第一种
location  /abc
    {
        proxy_pass http://106.12.74.123:83/;
        proxy_set_header Host   $host;
        proxy_set_header X-Real-IP      $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
 
# 结论：会被代理到http://106.12.74.123//index.html 这个url
 
 
 
# 第二种(相对于第一种，最后少一个 /)
location  /abc
    {
        proxy_pass http://106.12.74.123:83;
        proxy_set_header Host   $host;
        proxy_set_header X-Real-IP      $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
 
# 结论：会被代理到http://106.12.74.123/abc/index.html 这个url
 
 
 
第三种：
location  /abc
    {
        proxy_pass http://106.12.74.123:83/linux/;
        proxy_set_header Host   $host;
        proxy_set_header X-Real-IP      $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
 
# 结论：会被代理到http://106.12.74.123/linux//index.html 这个url。
 
 
 
# 第四种(相对于第三种，最后少一个 / )：
location  /abc/
    {
        proxy_pass http://106.12.74.123:83/linux;
        proxy_set_header Host   $host;
        proxy_set_header X-Real-IP      $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
 
# 结论：会被代理到http://106.12.74.123/linuxindex.html 这个url
下面我们验证一下：

环境准备：nginx的配置如下

主配置文件：

server {
    listen       80;
    server_name  localhost;
 
    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;
 
  
    location  /abc
    {
        proxy_pass http://106.12.74.123:83/;
        proxy_set_header Host   $host;
        proxy_set_header X-Real-IP      $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
副配置文件，上面是代理到83端口了，我们看一下83端口的配置为文件，之前做其他实验，现在我们就在这个上面简单修改下。

[root@master conf.d]# cat 83.conf 
server
{
   listen 83;
   server_name 106.127.74.123;
   root /usr/share/nginx/83;
}
查看静态文件地址：

[root@master nginx]# tree 83/
83/
├── abc
│   └── index.html
├── index.html
└── linux
    └── index.html
 
[root@master nginx]# cat 83/index.html 
83
 
[root@master nginx]# cat 83/abc/index.html 
83  abc
 
[root@master nginx]# cat 83/linux/index.html 
linux
第一种：当我们http://106.12.74.123/abc/index.html时候，页面出现的是83

第二种：当我们http://106.12.74.123/abc/index.html时候，页面出现的是83  abc的内容

第三种：当我们http://106.12.74.123/abc/index.html时候，页面出现的是linux的内容

第四种：当我们http://106.12.74.123/abc/index.html时候，页面出现的是404  因为83目录下并没有linxuindex.html的文件
```

