> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 Nginx篇（一）](https://juejin.im/post/6865077469934944264)<br>
> [Java工程师的进阶之路 Nginx篇（二）](https://juejin.im/post/6865217052181954567)<br>

## 1. Nginx 简介

> Nginx (engine x) 是一个高性能的HTTP和反向代理服务，也是一个IMAP/POP3/SMTP服务。Nginx是由伊戈尔·赛索耶夫为俄罗斯访问量第二的Rambler.ru站点（俄文：Рамблер）开发的，第一个公开版本0.1.0发布于2004年10月4日。其将源代码以类BSD许可证的形式发布，因它的稳定性、丰富的功能集、示例配置文件和低系统资源的消耗而闻名。2011年6月1日，nginx 1.0.4发布。

Nginx是一款轻量级的Web 服务器/反向代理服务器及电子邮件（IMAP/POP3）代理服务器，并在一个BSD-like 协议下发行。其特点是占有内存少，并发能力强，事实上nginx的并发能力确实在同类型的网页服务器中表现较好。

## 2. 正向代理和反向代理

代理其实就是一个中介，A和B本来可以直连，中间插入一个C，C就是中介。
刚开始的时候，代理多数是帮助内网client访问外网server用的
后来出现了反向代理，"反向"这个词在这儿的意思其实是指方向相反，即代理将来自外网客户端的请求转发到内网服务器，从外到内。

* **正向代理即是客户端代理, 代理客户端, 服务端不知道实际发起请求的客户端**。
* **反向代理即是服务端代理, 代理服务端, 客户端不知道实际提供服务的服务端**。
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d9f8ff204de44cc7b510c21a1ebda303~tplv-k3u1fbpfcp-zoom-1.image)

正向代理中，proxy和client同属一个LAN，对server透明；
反向代理中，proxy和server同属一个LAN，对client透明。
实际上proxy在两种代理中做的事都是代为收发请求和响应，不过从结构上来看正好左右互换了下，所以把后出现的那种代理方式叫成了反向代理。

* **正向代理的作用**

1. 访问原来无法访问的资源，如google；
2. 可以做缓存，加速访问资源；
3. 对客户端访问授权，上网进行认证；
4. 代理可以记录用户访问记录（上网行为管理），对外隐藏用户信息。
  
* **反向代理的作用**

1. 保证内网的安全，阻止web攻击，大型网站，通常将反向代理作为公网访问地址，Web服务器是内网；
2. 负载均衡，通过反向代理服务器来优化网站的负载。

## 3. Nginx 特点

Nginx 做为 HTTP 服务器，有以下几项基本特性：

1. 处理静态文件，索引文件以及自动索引；打开文件描述符缓冲。
2. 无缓存的反向代理加速，简单的负载均衡和容错。
3. FastCGI，简单的负载均衡和容错。
4. 模块化的结构。包括 gzipping, byte ranges, chunked responses,以及 SSI-filter 等 filter。如果由 FastCGI 或其它代理服务器处理单页中存在的多个 SSI，则这项处理可以并行运行，而不需要相互等待。
5. 支持 SSL 和 TLSSNI。

## 4. Nginx 工作原理

Nginx由内核和模块组成。

Nginx本身做的工作实际很少，当它接到一个HTTP请求时，它仅仅是通过查找配置文件将此次请求映射到一个location block，而此location中所配置的各个指令则会启动不同的模块去完成工作，因此模块可以看做Nginx真正的劳动工作者。通常一个location中的指令会涉及一个handler模块和多个filter模块（当然，多个location可以复用同一个模块）。handler模块负责处理请求，完成响应内容的生成，而filter模块对响应内容进行处理。
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/252688e336a943dea21eb4c5e4685946~tplv-k3u1fbpfcp-zoom-1.image)

用户根据自己的需要开发的模块都属于第三方模块。正是有了这么多模块的支撑，Nginx的功能才会如此强大。

Nginx的模块从结构上分为核心模块、基础模块和第三方模块：

* **核心模块**：HTTP模块、EVENT模块和MAIL模块；
* **基础模块**：HTTP Access模块、HTTP FastCGI模块、HTTP Proxy模块和HTTP Rewrite模块；
* **第三方模块**：HTTP Upstream Request Hash模块、Notice模块和HTTP Access Key模块。

Nginx的模块从功能上分为如下三类：

* **Handlers（处理器模块）**：此类模块直接处理请求，并进行输出内容和修改headers信息等操作。Handlers处理器模块一般只能有一个。
* **Filters （过滤器模块）**：此类模块主要对其他处理器模块输出的内容进行修改操作，最后由Nginx输出。
* **Proxies （代理类模块）**：此类模块是Nginx的HTTP Upstream之类的模块，这些模块主要与后端一些服务比如FastCGI等进行交互，实现服务代理和负载均衡等功能。

## 5. Nginx处理HTTP请求流程

http请求是典型的请求-响应类型的的网络协议。

http是文件协议，所以我们在分析请求行与请求头，以及输出响应行与响应头，往往是一行一行的进行处理。通常在一个连接建立好后，读取一行数据，分析出请求行中包含的method、uri、http_version信息。然后再一行一行处理请求头，并根据请求method与请求头的信息来决定是否有请求体以及请求体的长度，然后再去读取请求体。得到请求后，我们处理请求产生需要输出的数据，然后再生成响应行，响应头以及响应体。在将响应发送给客户端之后，一个完整的请求就处理完了。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b1480189886b41c189d9effba68b8cb0~tplv-k3u1fbpfcp-zoom-1.image)

Nginx处理HTTP反向代理请求流程：
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03aad5e4fee74c18be9834adfb2dae29~tplv-k3u1fbpfcp-zoom-1.image)

## 6. Nginx 进程模型

Nginx默认采用多进程工作方式，Nginx启动后，会运行一个master进程和多个worker进程。其中master充当整个进程组与用户的交互接口，同时对进程进行监护，管理worker进程来实现重启服务、平滑升级、更换日志文件、配置文件实时生效等功能。worker用来处理基本的网络事件，worker之间是平等的，他们共同竞争来处理来自客户端的请求。

nginx的进程模型如图所示：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8bff4d2289c439a8ee4ad1885241bea~tplv-k3u1fbpfcp-zoom-1.image)

在创建master进程时，先建立需要监听的socket（listenfd），然后从master进程中fork()出多个worker进程，如此一来每个worker进程多可以监听用户请求的socket。一般来说，**当一个连接进来后，所有在Worker都会收到通知，但是只有一个进程可以接受这个连接请求，其它的都失败，这是所谓的惊群现象**。nginx提供了一个accept_mutex（互斥锁），有了这把锁之后，同一时刻，就只会有一个进程在accpet连接，这样就不会有惊群问题了。

先打开accept_mutex选项，只有获得了accept_mutex的进程才会去添加accept事件。**nginx使用一个叫ngx_accept_disabled的变量来控制是否去竞争accept_mutex锁**。ngx_accept_disabled = nginx单进程的所有连接总数 / 8 -空闲连接数量，当ngx_accept_disabled大于0时，不会去尝试获取accept_mutex锁，ngx_accept_disable越大，于是让出的机会就越多，这样其它进程获取锁的机会也就越大。不去accept，每个worker进程的连接数就控制下来了，其它进程的连接池就会得到利用，这样，nginx就控制了多进程间连接的平衡。

**每个worker进程都有一个独立的连接池，连接池的大小是worker_connections**。这里的连接池里面保存的其实不是真实的连接，它只是一个worker_connections大小的一个ngx_connection_t结构的数组。并且，nginx会通过一个链表free_connections来保存所有的空闲ngx_connection_t，每次获取一个连接时，就从空闲连接链表中获取一个，用完后，再放回空闲连接链表里面。一个nginx能建立的最大连接数，应该是worker_connections * worker_processes。当然，这里说的是最大连接数，对于HTTP请求本地资源来说，能够支持的最大并发数量是worker_connections * worker_processes，而如果是**HTTP作为反向代理来说，最大并发数量应该是 worker_connections * worker_processes/2 **。因为作为反向代理服务器，每个并发会建立与客户端的连接和与后端服务的连接，会占用两个连接。

### 6.1. master与worker

nginx在启动后，会有**一个master进程和多个worker进程**。

master进程主要用来管理worker进程，接收来自外界的信号，向各worker进程发送信号，监控worker进程的运行状态，当worker进程退出后(异常情况下)，会自动重新启动新的worker进程。 

而基本的网络事件，则是放在worker进程中来处理了。多个worker进程之间是对等的，他们同等竞争来自客户端的请求，各进程互相之间是独立的。 
**一个请求，只可能在一个worker进程中处理，一个worker进程，不可能处理其它进程的请求**。worker进程的个数是可以设置的，一般我们会设置与机器cpu核数一致。

### 6.2. master与处理请求

那么我们该怎么操作ngnix呢？
其实我们只需要通过与master进行通信（命令）就可以操作ngnix，master进程会接收来自外界发来的信号，再根据信号做不同的事情。 

比如`kill -HUP pid`，我们一般用这个信号来重启nginx，或重新加载配置，因为是从容地重启，因此服务是不中断的。**首先master进程在接到信号后，会先重新加载配置文件，然后再启动新的worker进程，并向所有老的worker进程发送信号，告诉他们可以退出了。新的worker在启动后，就开始接收新的请求**，而老的worker在收到来自master的信号后，就不再接收新的请求，并且在当前进程中的所有未处理完的请求处理完成后，再退出。 

当然，直接给master进程发送信号，这是比较老的操作方式，nginx在0.8版本之后，引入了一系列命令行参数，来方便我们管理。 
比如，`./nginx -s reload`，就是来重启nginx，`./nginx -s stop`，就是来停止nginx的运行。如何做到的呢？我们还是拿reload来说，我们看到，执行命令时，我们是启动一个新的nginx进程，而新的nginx进程在解析到reload参数后，就知道我们的目的是控制nginx来重新加载配置文件了，它会向master进程发送信号，然后接下来的动作，就和我们直接向master进程发送信号一样了。

### 6.3. worker与处理请求

前面有提到，worker进程之间是平等的，每个进程，处理请求的机会也是一样的。当我们提供80端口的http服务时，一个连接请求过来，每个进程都有可能处理这个连接。怎么做到的呢？ 

首先，每个worker进程都是从master进程fork过来，在master进程里面，先建立好需要listen的socket（listenfd）之后，然后再fork出多个worker进程。**所有worker进程的listenfd会在新连接到来时变得可读。为保证只有一个进程处理该连接，所有worker进程在注册listenfd读事件前抢accept_mutex，抢到互斥锁的那个进程注册listenfd读事件，在读事件里调用accept接受该连接**。 
当一个worker进程在accept这个连接之后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接，这样一个完整的请求就是这样的了。

### 6.4. nginx进程模型的好处

首先，对于每个worker进程来说，独立的进程，不需要加锁，所以省掉了锁带来的开销，同时在编程以及问题查找时，也会方便很多。 
其次，**采用独立的进程，可以让worker互相之间不会影响，一个worker退出后，其它worker还在工作，服务不会中断，master进程则很快启动新的worker进程**。 
当然，worker进程的异常退出，肯定是程序有bug了，异常退出，会导致当前worker上的所有请求失败，不过不会影响到所有请求，所以降低了风险。 
好处是很多的，只能在使用中慢慢体会了。

### 6.5. nginx处理高并发（网络事件）

nginx如何处理高并发呢？按理说nginx采用多worker的方式来处理请求，每个worker里面只有一个主线程，那能够处理的并发数很有限啊，多少个worker就能处理多少个并发，何来高并发呢？ 

其实这就是**nginx的高明之处，nginx采用了异步非阻塞的方式来处理请求**，也就是说，nginx是可以同时处理成千上万个请求的。 
为什么nginx可以采用异步非阻塞的方式来处理呢，或者异步非阻塞到底是怎么回事呢？

一个完整过程：请求过来，要建立连接，然后再接收数据，接收数据后，再发送数据，具体到系统底层，就是读写事件，而当读写事件没有准备好时，必然不可操作，如果不用非阻塞的方式来调用，那就得阻塞调用了，事件没有准备好，那就只能等了，等事件准备好了，你再继续吧。阻塞调用会进入内核等待，cpu就会让出去给别人用了，对单线程的worker来说，显然不合适，当网络事件越多时，大家都在等待呢，cpu空闲下来没人用，cpu利用率自然上不去了，更别谈高并发了。

> [Java工程师的进阶之路 Nginx篇（一）](https://juejin.im/post/6865077469934944264)<br>
> [Java工程师的进阶之路 Nginx篇（二）](https://juejin.im/post/6865217052181954567)<br>