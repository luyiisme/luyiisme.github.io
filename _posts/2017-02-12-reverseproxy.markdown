---
layout:     post
title:      "反向代理场景合理使用长连接"
date:       2017-02-12
author:     "luyi"
header-img: "img/post-bg-metalworking.jpg"
tags:
    - java
---
讨论场景：      NGINX 可以 http1.1 反向代理泛web容器(比如，Tomcat)的部署形式

在企业级部署架构中，我们经常会使用到 Nginx,Apache 等软件来反向代理  HTTP 请求到内部 web 服务器。以 Nginx 使用为例，将其前置于常见的动态服务器（Tomcat，Jetty,NodeJs...）可获得很多好处。比如，将应用的静态资源（css,js,html...)交由 Nginx 处理，我们可以不用关心 eTags 或者 Cache-Control 的缓存头处理以及资源压缩即可获得更快的响应和更高的吞吐。虽然很多web开发框架（Play,Spring MVC,NodeJs相关开发框架等）都有相关能力，但是开发时，你还是要去确认下使用是否正确，有没生效，生产环境如何方便运维以及性能如何等问题。 Nginx将非静态资源的请求代理到上游的 web 服务器，这种结构在后端 web 服务器出问题或发布变更时，也能由 Nginx 提供更友好的对应错误页面提示。

HTTP协议从 1.1 版本开始，在未特殊声明的情况下，都默认建立持久连接（长连接），所以我们需要确认服务端是否正确配置支持持久化连接了。尤其 NGINX 向后台web服务器转发请求时，因为频繁的创建销毁连接不仅消耗 CPU，也增加延迟，如果 HTTPS 协议，那成本就尤为明显。

## 1、那么如何确定整个请求链接路是不是有效利用了长连接（conn keepalive)呢

keep-alive 协议头，是 HTTP 协议层面引入的，为了复用底层 TCP 连接的标识。下面以浏览器端（其他支持 Keep alive 机制的 HTTP 客户端类似）访问页面为例逐个阶段分析。

![image](https://cdn-1.wp.nginx.com/wp-content/uploads/2016/03/Python-NGINX-architecture-1024x596.png)

### （1）首先是 [浏览器-->nginx 阶段] 的http请求基于的connection。 nginx 的 http module 我们除了配置使用 http1.1 协议外，还需要关心：

```
  keepalive_disable none； //表示对所有的浏览器都支持keepalive;(默认只对 msie6)

  keepalive_requests 100;  //默认100，一般是合适的。这个配置什么意思呢？参考下面对tomcat的connector的配置参数：maxKeepAliveRequests 的说明；

  keepalive_timeout 75s；  //默认75s. 0表示disable keepalive; 正常情况下可能5秒左右就合适
```

至此，如果你是请求静态资源(即 nginx 提供的资源服务)那么 http 协议的 keepalive 优化机制就生效了；

### (2)如果请求被反向代理到了应用服务器（以 Tomcat 为例），「nginx-->tomcat 阶段」

 这种反向代理的部署架构，对 tomcat 而言就比较友好了，因为其面对的 client 是确定的，主要请求是来自 nginx 转发过来的。因此这一阶段，我们应该尽量最大化发挥 keepalive 能力，重复使用已持久化的连接。

  * NGINX 端的 http-upstream 模块，有个相关配置 keepalive(nginx 版本至少 1.1.4)：

```
  keepalive  connections; //connections表示最大空闲上限数目，没有默认值；
```
怎么理解这个最大空闲的阀值呢。假设当前处于空闲状态的连接数目超过了该值，那么超过数目的部分连接会被关闭；
这种情况也是比较正常的，当一段时间并发的请求过多（比如整点的业务活动），现有的（nginx到tomcat的）连接不够用了，就会创建新的，一旦这些请求处理响应结束了，就会多出很多的空闲连接了。

```
upstream http_backend {
    server 127.0.0.1:8080;
    keepalive 16;
}

server {
    ...
    location /http/ {
        proxy_pass http://http_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        ...
    }
}
```
另外proxy_pass 需搭配“proxy_http_version 1.1”，“proxy_set_header Connection ""”两个配置项。表示在代理外部请求到上游服务器时，忽略原始请求的关于 Connection 头的设置，且重新以 HTTP 1.1协议版本（因为该版本是默认支持Keep-Alive的）转发给应用服务器端.

这里补充说明下 HTTP 1.* 协议复用持久连接的背景。因为先前的年代浏览器都默认短连接，如果页面中资源过多时，频繁创建新连接去服务每个资源请求这样明显效率不高的，因此通过连接持久化供多个请求复用来提高效率。但是这种复用对请求而言是同步的，后面的请求需要等待前面的请求收到响应后，才能复用。比如，某个主要的业务服务的http请求在应用服务器处理需要：100毫秒, 那么每秒大约一个连接最大可以供10个请求复用。因此上面最大空闲连接数设置多大，是可以粗略估算的。

另外，如果 nginx 中 upstream 配置了多个 server，那么每个server的每个最大空闲数目都是 “connections” ；

 * 最后讨论下 tomcat 的 connector 的配置参数：

```
  keepAliveTimeout  //默认值是connectionTimeout的值（单位毫秒），我们可以用-1表示不超时。

  maxKeepAliveRequests //默认值是100，直接面向用户合适，但是面向nginx就不合适；我们可以使用-1,表示没有限制；这个配置什么意思呢？一个 tcp 链接建立起，一共处理过请求数目如果达到{maxKeepAliveRequests}，那么这个服务器就取消这个连接的keepalive并关闭链接；  
```

这里我们截取 tomcat 的 http1.1processor 一段基于 maxKeepAliveRequests 这个 socket 对应的属性变量，将 keepalive 关闭的代码。可以看出如何配置为1,效果就是一个连接一个请求的经典模式，所以没必要 keepalive。还有一种情况 maxKeepAliveRequests>0(>1) 会统计请求，是否达到了连接承载了请求的数量，每次请求就-1，达到了 maxKeepAliveRequests 值，同样会关闭 keepalive,并最终关闭 tcp 连接。

      tomcat_maxkeepaliverequests


好的，这样整个 http 链路应该是合理的 keepalive；怎么确认了，我们只要 netstat 对应的分别的服务器监听端口，然后发起请求，如果发现很多 tcp state 是 time await (表示 nginx 发起到 tomcat 连接，主动关闭了) 那就还是一次请求一个连接。如果 state 是 established，那么就对了；

最后值得说明的是，这样可能并不会显著提高 tps 压测值，因为这些还不是主要的吞吐量，处理能力相关指标。但是它极大程度上复用了连接，避免了频繁的连接创建和销毁（这也会产生一定的性能影响）。   假设系统面临了持续一定时间高访问量，那么这个期间总共创建和销毁的连接总数近似为（tomcat_maxConnentions — nginx_keepalive）。 笔者，做过了简单的压测nginx的 keepalive=100,tomcat最大连接数500，那么整个过程一直到流量为0时，共创建销毁近400个连接，而如果我们tomcat的不支持 keepalive,那么整个过程中会创建销毁三四千甚至更多的连接数。

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
