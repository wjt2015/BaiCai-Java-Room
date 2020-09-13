> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 Nginx篇（一）](https://juejin.im/post/6865077469934944264)<br>
> [Java工程师的进阶之路 Nginx篇（二）](https://juejin.im/post/6865217052181954567)<br>

## 1. Nginx 模块功能

Nginx配置文件的整体结构：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b3ba8b190e854f5cbb0bc979f9c039d9~tplv-k3u1fbpfcp-zoom-1.image)

可以看出主要包含以下几大部分内容：

### 1.1. 全局块

* 配置运行Nginx服务器用户（组）
* worker process数
* Nginx进程PID存放路径
* 错误日志的存放路径
* 配置文件的引入

### 1.2. events块

* 设置网络连接的序列化
* 是否允许同时接收多个网络连接
* 事件驱动模型的选择
* 最大连接数的配置

### 1.3. http块

* 定义MIMI-Type
* 自定义服务日志
* 允许sendfile方式传输文件
* 连接超时时间
* 单连接请求数上限

### 1.4. server块

* 配置网络监听
* 基于名称的虚拟主机配置
* 基于IP的虚拟主机配置

### 1.5. location块

* location配置
* 请求根目录配置
* 更改location的URI
* 网站默认首页配置

## 2. Nginx 配置解析

按照前面文章，给出一份简要的清单配置举例：
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07d67093a78e480d88fee08d91b6c021~tplv-k3u1fbpfcp-zoom-1.image)

### 2.1. 配置运行Nginx服务器用户（组）

```
指令格式：user user [group];
```

**user**：指定可以运行Nginx服务器的用户

**group**：可选项，可以运行Nginx服务器的用户组

如果user指令不配置或者配置为 `user nobody nobody` ，则默认所有用户都可以启动Nginx进程。

### 2.2. worker_process数配置

Nginx服务器实现并发处理服务的关键。

```
指令格式：worker_processes number | auto;
```

**number**：Nginx进程最多可以产生的worker process数

**auto**：Nginx进程将自动检测

启动Nginx服务器后，我们可以后台看一下主机上的Nginx进程情况：

```
ps -aux | grep nginx
```

### 2.3. Nginx进程PID存放路径

Nginx进程是作为系统守护进程在运行，需要在某文件中保存当前运行程序的主进程号，Nginx支持该保存文件路径的自定义。

```
指令格式：pid file;
```

**file**：指定存放路径和文件名称

如果不指定默认置于路径 logs/nginx.pid。

### 2.4. 错误日志的存放路径

```
指定格式：error_log file | stderr;
```

**file**：日志输出到某个文件file

**stderr**：日志输出到标准错误输出

### 2.5. 配置文件的引入

```
指令格式：include file;
```

该指令主要用于将其他的Nginx配置或者第三方模块的配置引用到当前的主配置文件中。

### 2.6. 设置网络连接的序列化

```
指令格式：accept_mutex on | off;
```

该指令默认为on状态，表示会对多个Nginx进程接收连接进行序列化，防止多个进程对连接的争抢。

如果accept_mutex on，那么多个worker将是以串行方式来处理，其中有一个worker会被唤醒；反之若accept_mutex off，那么所有的worker都会被唤醒，不过只有一个worker能获取新连接，其它的worker会重新进入休眠状态。

### 2.7. 允许同时接收多个网络连接

```
指令格式：multi_accept on | off;
```

该指令默认为off状态，意指每个worker process 一次只能接收一个新到达的网络连接。若想让每个Nginx的worker process都有能力同时接收多个网络连接，则需要开启此配置。

### 2.8. 事件驱动模型的选择

```
指令格式：use model;
```

model模型可选择项包括：select、poll、kqueue、epoll、rtsig等......

### 2.9. 最大连接数的配置

```
指令格式：worker_connections number;
```

number默认值为512，表示允许每一个worker process可以同时开启的最大连接数。

### 2.10. 定义MIME-Type

```
指令格式：include mime.types;default_type mime-type;
```

MIME-Type指的是网络资源的媒体类型，也即前端请求的资源类型

include指令将mime.types文件包含进来

cat mime.types 来查看mime.types文件内容，我们发现其就是一个types结构，里面包含了各种浏览器能够识别的MIME类型以及对应类型的文件后缀名字。

### 2.11. 自定义服务日志

```
指令格式：access_log path [format];
```

**path**：自定义服务日志的路径 + 名称

**format**：可选项，自定义服务日志的字符串格式。其也可以使用 log_format 定义的格式

### 2.12. 允许sendfile方式传输文件

```
指令格式：sendfile on | off;sendfile_max_chunk size;
```

前者用于开启或关闭使用sendfile()传输文件，默认off

后者指令若size>0，则Nginx进程的每个worker process每次调用sendfile()传输的数据了最大不能超出此值；若size=0则表示不限制。默认值为0。

### 2.13. 连接超时时间配置

```
指令格式：keepalive_timeout timeout [header_timeout];
```

keepalive_timeout timeout [header_timeout];
timeout 表示server端对连接的保持时间，默认75秒

header_timeout 为可选项，表示在应答报文头部的 Keep-Alive 域设置超时时间："Keep-Alive : timeout = header_timeout"。

### 2.14. 单连接请求数上限

```
指令格式：keepalive_requests number;
```

该指令用于限制用户通过某一个连接向Nginx服务器发起请求的次数。

### 2.15. 配置网络监听


指令格式：
```
第一种：配置监听的IP地址：listen IP[:PORT];
```
```
第二种：配置监听的端口：listen PORT;
```

实际举例：
```
listen 192.168.31.177:8080;   # 监听具体IP和具体端口上的连接
listen 192.168.31.177;   # 监听IP上所有端口上的连接
listen 8080;     # 监听具体端口上的所有IP的连接
```

### 2.16. 基于名称和IP的虚拟主机配置

```
指令格式：server_name IP地址
```
```
指令格式：server_name name1 name2 ...
```

name可以有多个并列名称，而且此处的name支持正则表达式书写

实际举例：
```
server_name ~^www\d+\.myserver\.com$
```

### 2.17. location配置

```
指令格式：location [ = | ~ | ~* | ^~ ] uri {...}
```

这里的uri分为标准uri和正则uri，两者的唯一区别是uri中是否包含正则表达式(URI，是uniform resource identifier，统一资源标识符，用来唯一的标识一个资源。而URL是uniform resource locator，统一资源定位器，它是一种具体的URI，即URL可以用来标识一个资源，而且还指明了如何locate这个资源。)

**uri前面的方括号中的内容是可选项**，解释如下：

**"="**：用于标准uri前，要求请求字符串与uri严格匹配，一旦匹配成功则停止

**"~"**：用于正则uri前，并且区分大小写

**"~*"**：用于正则uri前，但不区分大小写

**"^~"**：用于标准uri前，要求Nginx找到标识uri和请求字符串匹配度最高的location后，立即使用此location处理请求，而不再使用location块中的正则uri和请求字符串做匹配

### 2.18. 请求根目录配置

```
指令格式：root path;
```

**path**：Nginx接收到请求以后查找资源的根目录路径

当然，还可以通过alias指令来更改location接收到的URI请求路径，指令为：
```
alias path; # path为修改后的根路径
```

### 2.19. 设置网站的默认首页

```
指令格式：index file ......
```

file可以包含多个用空格隔开的文件名，首先找到哪个页面，就使用哪个页面响应请求

其实Nginx的配置真的是很简单，对于新手们来说其实最大的问题就是Nginx所有的配置都是基于配置文件和各个模块语法的，这些看着给人的感觉好复杂的样子，其实理解了各个模块的意义和基本语法后就变的尤为简单了！

## 3. Nginx 配置HTTPS

随着微信小程序和appstore对ssl安全的需求，越来越多的网站和app需要支持SSL功能，需要开启https的方式来打开网站或传输数据。

Nginx配置ssl并把Http跳转到Https，ssl证书网上可以找到收费和免费的申请，需要修改Nginx.conf配置文件：

```
# Settings for a TLS enabled server.

# 如果是http请求默认访问80端口，此时return强行301重定向到https://www.baidu.com

server {

  listen 80;

  server_name www.baidu.com;

  return 301 https://www.baidu.com$request_uri;

  # 把http重定向到https使用了nginx的重定向命令，之前老版本的nginx可能使用了以下类似的格式：
  # rewrite ^/(.*)$ http://www.baidu.com/$1 permanent;
  # 或者：
  # rewrite ^ http://www.baidu.com$request_uri? permanent;
  # 现在nginx新版本已经换了种写法，上面这些已经不再推荐。现在网上可能还有很多文章写的是第一种。
  # 新的写法比较推荐方式是：return 301 https://www.baidu.com$request_uri;
}

server {

  listen 443;
  server_name www.baidu.com;
  root /data/release/weapp/uploadFiles;

  # 开启ssl功能
  ssl on;

  # 配置ssl证书，直接用.pem和.key文件的绝对路径

  ssl_certificate/data/release/nginx/自己去申请一个ssl证书.pem;

  ssl_certificate_key/data/release/nginx/自己去申请一个ssl证书.key;

  ssl_session_timeout 5m;

  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

  ssl_ciphers ECDHE - RSA - AES128 - GCM - SHA256: ECDHE: ECDH: AES: HIGH: !NULL: !aNULL: !MD5: !ADH: !RC4;

  ssl_prefer_server_ciphers on;

  location / {

     proxy_pass http://app_weapp;

     proxy_http_version 1.1;

     proxy_set_header Upgrade $http_upgrade;

     proxy_set_header Connection 'upgrade';

     proxy_set_header Host $host;

     proxy_cache_bypass $http_upgrade;

  }

  location /images/ {
    autoindex on;
  }

  # 配置uri， ~用于正则uri前，其中.(png|jpg)为正则表达式，如果后缀是.png或.jpg的url请求，则匹配成功
  # root用于配置接收到请求以后查找资源的根目录路径

  location ~ \.(png|jpg) {
     root /data/release/weapp/uploadFiles;
  }

  error_page 404 /404.html;

  location = /40x.html {
  }

  error_page 500 502 503 504 /50x.html;

  location = /50x.html {
  }
}
```

> [Java工程师的进阶之路 Nginx篇（一）](https://juejin.im/post/6865077469934944264)<br>
> [Java工程师的进阶之路 Nginx篇（二）](https://juejin.im/post/6865217052181954567)<br>