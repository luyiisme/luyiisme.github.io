---
layout:     post
title:      "反向代理场景合理使用长连接"
date:       2017-02-12
author:     "luyi"
header-img: "img/post-bg-metalworking.jpg"
tags:
    - nginx
    - tomcat
    - 长连接
---
讨论场景：      NGINX 可以 http1.1 反向代理泛web容器(比如，Tomcat)的部署形式

## 1、整个请求链接路是不是有效利用了长连接（conn keepalive)？

keepalive机制，是http协议层面引入的，为了复用tcp连接的标识。

 - 首先是 [浏览器-->nginx 阶段] 的http请求基于的connection。 nginx的 http module 我们除了配置使用 http1.1 协议外，还需要关心：

```
  keepalive_disable none； //表示对所有的浏览器都支持keepalive;(默认只对 msie6)

  keepalive_requests 100;  //默认100，一般是合适的。这个配置什么意思呢？参考下面对tomcat的connector的配置参数：maxKeepAliveRequests 的说明；

  keepalive_timeout 75s；  //默认75s. 0表示disable keepalive; 正常情况下可能5秒左右就合适
```

至此，如果你是请求静态资源(即 nginx 提供的资源服务)那么 http 协议的 keepalive 优化机制就生效了；

- 如果请求被反向代理进行了 tomcat，「nginx-->java server 阶段」

 这里场景比较特殊，因为对 tomcat 而言，面对的 client 是确定的，不再是不确定的用户端，因此我们尽量最大化去使用 keepalive 能力；

 * （2.1）nginx 端 http-upstream 模块，有个相关配置 keepalive(nginx 版本至少 1.1.4)：

```
  keepalive  connections; //没有默认值；缓存住多少个空闲keepalive的conn供使用，超过部分会关闭；（为什么会超过？因为如果并发较大，当时的conn资源小于请求数量就会创建）；另外如果upstream配置了个多个，每个都connections个；
```

 * （2.2）最后tomcat的connector的配置参数：

```
  keepAliveTimeout  //默认值是connectionTimeout的值（单位毫秒），我们可以用-1表示不超时。

  maxKeepAliveRequests //默认值是100，直接面向用户合适，但是面向nginx就不合适；我们可以使用-1,表示没有限制；这个配置什么意思呢？一个tcp conn建立起，一共处理过请求数目如果达到{maxKeepAliveRequests}那么这个服务器就取消这个连接的keepalive并关闭链接；  
```

这里我们截取 tomcat 的 http1.1processor 一段基于 maxKeepAliveRequests 这个 socket 对应的属性变量，将 keepalive 关闭的代码。可以看出如何配置为1,效果就是一个连接一个请求的经典模式，所以没必要 keepalive。还有一种情况 maxKeepAliveRequests>0(>1) 会统计请求，是否达到了连接承载了请求的数量，每次请求就-1，达到了 maxKeepAliveRequests 值，同样会关闭 keepalive,并最终关闭 tcp 连接。

      tomcat_maxkeepaliverequests


好的，这样整个 http 链路应该是合理的 keepalive；怎么确认了，我们只要 netstat 对应的分别的服务器监听端口，然后发起请求，如果发现很多 tcp state 是 time await (表示 nginx 发起到 tomcat 连接，主动关闭了) 那就还是一次请求一个连接。如果 state 是 established，那么就对了；

最后值得说明的是，这样可能并不会显著提供 tps 压测值，因为这些还不是主要的吞吐量，处理能力相关指标。但是它极大程度上复用了连接，避免了频繁的连接创建和销毁（这也会产生一定的性能影响）。   假设系统面临了持续一定时间高访问量，那么这个期间总共创建和销毁的连接总数近似为（tomcat_maxConnentions — nginx_keepalive）。 笔者，做过了简单的压测nginx的 keepalive=100,tomcat最大连接数500，那么整个过程一直到流量为0时，共创建销毁近400个连接，而如果我们tomcat的不支持 keepalive,那么整个过程中会创建销毁三四千甚至更多的连接数。

注：频繁的连接创建也会出现socket不够用的情况。why?

    linux可创建的最大连接数，受限于：系统最大文件句柄数量，本地端口范围限制；

    TCP协议不允许处于TIME_WAIT状态的连接启动一个新的可用连接。而TIME_WAIT状态持续2MSL，因为这样可以保证当成功建立一个新TCP连接的时候，来自旧连接重复分组已经在网络中消逝。

 MSL就是maximum segment lifetime(最大分节生命期），这是一个IP数据包能在互联网上生存的最长时间，超过这个时间IP数据包将在网络中消失 。MSL在RFC 1122上建议是2分钟，而源自berkeley的TCP实现传统上使用30秒。--
TIME_WAIT状态维持时间是两个MSL时间长度，也就是在1-4分钟。Windows操作系统就是4分钟。



## 2、下面在简单讲下，其他tomcat的参数配置(主要是会影响到吞吐量和tps等性能指标的)；


    minSpareThreads   //默认值是10，线上应用这是太少了； 保持运行的线程数目阀值；    

    acceptCount //默认值100，但请求到达了maxConnections的数目，那么就会缓存在等待队列中，队列长度就是acceptCount;

    maxConnections  //NIO默认10000，BIO情况下，参考maxThreads的配置值；

    maxThreads //默认是200，但是如果配置executor,就按照executors;

    protocol="org.apache.coyote.http11.Http11NioProtocol"    使用非传统一个请求一个线程的，非阻塞处理方式；
---

### 著作权声明

`首次发布于此，转载请保留以上链接`

参考文章TODO;
