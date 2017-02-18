---
layout:     post
title:      "ClientAbortException 问题分析"
date:       2017-02-18
author:     "luyi"
header-img: "img/post-bg-metalworking.jpg"
tags:
    - ClientAbortException
    - Broken pipe
    - socketException
---

## 话题：

基于WEB层面来讨论分析问题，引出tcp层面的问题本质；

## 背景：

笔者常被问到 ClientAbortException 问题，正好就一种场景的分析来简单分析下（其他场景地分析也较为类似）。下面讨论的是使用 nginx 反向代理 tomcat java项目时，有时java应用会发现频繁打印  ClientAbortException 错误日志问题。

- 1、什么情况下会出现“ClientAbortException:  java.net.socketException:  Broken pipe”日志?

客户端非正常（标准握手协议）退出连接，体现在http请求，可能是用户等待页面响应过程中，关闭浏览器，或停止了请求。

- 2、解决方法是 nginx 加个固定的[proxy_ignore_client_abort on;](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ignore_client_abort)就可以了?

nginx 加上这个头效果一样，处理客户端非正常退出情况，保持与应用服务器端的conn，让其转发的服务器端还是可以继续对其写；

- 3、为什么有的应用报这个问题，有的 nginx 无 proxy_ignore_client_abort 确没有打错误日志？

tomcat 的专门 coyoteWriter 的写方法，已经替你忽略掉写的时候发生的异常。因此，异常其实是产生的，只不过被吞了。
      但为什么有的还打印出日志呢？这些应用是直接使用response.getWriter()拿到的，而拿取 response.getOutputStream() 来构造的writer,则没有忽略异常，而是选择了抛出，因此你看到了这个问题；`那么更一般的情况下，我们会在 mvc 框架的 controller 里（或者 servlet 里，或其他场合），直接操作 response.getOutputStream() 来进行业务输出响应流操作，异常会同样抛出`。

     备注：一旦这个异常发生，如果你还尝试去拦截处理的这个异常，并再次返回响应，但是因为你操作过了response.getOutputStream了，所有会再抛异常说`...response has been commited`! 那么解决的思路就是，加 nginx 的头，或者在 controller 里拦截这种 ClientAbortException，并忽略。


- 4、我自己在分析这个异常产生时，发现有的情况下套接字输出会抛ClientAbortException，有的时候提交正常？

分析发现，对一个对端已经关闭的socket调用两次write,即第二次尝试写才会生成SIGPIPE信号（why? 因为tcp的四次挥手）, 该信号默认结束进程，会抛出 Broken pipe 的 exception.  客户端自己关闭 socket,但是服务器端不知道，然后向客户端写一次数据，这时客户端会回应服务器端RST报文(Reset the connection), （如果你此时服务器端，尝试读就会看到这个错误报文）。我们这里是flush buffer，继续写第二次，那么此时就会抛异常：ClientAbortException 而且cause by broken pipe；

- 5、最后，有些应用正确配置了'proxy_ignore_client_abort on;'，但是还是会抛上述异常？

比较常见的情况是，被转发的请求处理耗时过久,已经超过了['proxy_read_timeout'](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_read_timeout)的时限，默认 60s。这种情况下「转发连接」已经读超时了，’proxy_ignore_client_abort‘当然也就没有效果了。’proxy_ignore_client_abort‘是防止 Nginx 前置的连接断掉请求，而对Nginx 后置（连接实际应用服务器）「转发连接」是不能防止的。 解决方法：***根本是解决应用该慢请求问题***。
         那么怎么确认这个问题呢？可以看 Nginx 的错误日志，里面会有错误信息
`...[error] 14345#0: *5 upstream timed out (110: Connection timed out) while reading response header from upstream, client: 10.15...`

## 总结：

（1）建议使用nginx的应用，都加上proxy_ignore_client_abort on；（加在Location的配置区域内）

（2）这个异常如果在到JAVA应用服务器，除了异常fillstacktrace，和打印日志io消耗,其他对业务本身服务是没什么影响的；

---

### 著作权声明

`首次发布于此，转载请保留以上链接`

参考文章TODO;
