---
layout:     post
title:      "Ribbon2 核心设计和原理分析"
date:       2017-05-09
author:     "luyi"
header-img: "img/post-bg-metalworking.jpg"
tags:
    - ribbon
---

[Ribbon](https://github.com/Netflix/ribbon), Netflix 公司开源的用于进程通信场景的经过实战检验的客户端项目。它提供了远程调用的负载均衡，网络状态检查和容错等功能。Ribbon 基于软件的负载均衡方式与目标集群中的机器进行通信。这里忽略状态统计部分，健康检查的逻辑部分，重点分析负载均衡部分。

下面是个典型的使用方式开始，先指定一些目标服务器地址，

```
//sample-client.ribbon.listOfServers=www.taobao.com:80,www.baidu.com:80,www.sina.com:80
```
然后借助ribbon 封装的 httpClient 来发送请求（访问站点首页），循环20次是希望打印出负载均衡的效果。

```
...
RestClient client = (RestClient) ClientFactory.getNamedClient("sample-client");  
HttpRequest request = HttpRequest.newBuilder().setUri(new URI("/")).build();
for (int i = 0; i < 20; i++)  {
  HttpResponse response = client.executeWithLoadBalancer(request);
  System.out.println("Status code for " + response.getRequestedURI() + "  :" + response.getStatus());
}
```
下面按 Ribbon 的主要模块进行设计分析，并在最后部分分析上述代码的效果。

## Ribbon-core 模块
这一模块中，主要定义通用的调用抽象。ribbon 的 client 处理请求并返回响应。

```
public interface IClient<S extends ClientRequest, T extends IResponse> {
    public T execute(S request, IClientConfig requestConfig) throws Exception;
}
```
ClientRequest， 是独立于所以具体实现的通信协议的抽象。它主要包括：uri(远程资源的定位符)，loadBalancerKey（Object类型，这是决定请求调用到目标服务器的关键信息），isRetriable; 当然它的具体实现类，可能还有请求headers,body等信息。

IResponse， 是服务端响应的抽象，主要包括: body(payload),isSuccess(响应状态的简化)，响应headers,请求url;

VipAddress 术语，是一个地址的逻辑名，该逻辑名代表一系列目标服务器。比如“apple.bar:80”。

RetryHandler,用来决定哪些异常（比如:ConnectException,SocketTimeoutException）发生时要做重试，哪些异常(SocketException,SocketTimeoutException)发生时表示应该熔断掉（对应服务器）的调用，默认的是不重试。

## Ribbon-loadbalancer 模块

负载均衡，宏观上效果是希望将请求的流量均衡（不是简单意义上的平均）的负载到提供服务的服务器之上。而对于每次请求而言，是通过计算拿到一个目标服务器地址的。

- ILoadBalancer

ILoadBalancer， 负载均衡器的接口。它的核心方法是 `public Server chooseServer(Object key);` 每次请求发生时根据参数传入的 loadBalancerKey对象，决定出一个目标的服务器地址。Server 表示一个服务器(包括：主机地址，端口号以及一些元数据标识信息)。  那么服务均衡器应该需要有一批可供选择的目标服务集合吧，所以它还有个重要方法 `addServers(List<Server> newServers)`，一般在启动阶段就要完成初始化地调用。

- BaseLoadBalancer

BaseLoadBalancer（实现 ILoadBalancer),这里声明了一个基础lb,应该关联个 IRule(负载均衡的规则）默认是轮询规则：RoundRobinRule)。 它还默认支持一个Ping检查功能（可以帮忙我们定时检查目标服务器是否可通信）的***定时任务***，开发者可以在构造实例时明确指定 Iping(判断一个服务是否还是活的)，IPingStrateg 默认是串行地检查策略，但如果目标服务地址过多，或者IPing执行过慢就不太合适。

IRule 路由规则，Rule 和 LoadBalancer 是一对一的相互关联关系，Rule 是具体的负责均衡策略。常见的规则包括: 轮询，随机，基于响应的延迟等; `public Server choose(Object key)`核心方法的输入输出基本一致，区别是 Rule 拿取目标 server list 是通过借助对应依赖的 LoadBalancer 拿到的。


- DynamicServerListLoadBalancer

DynamicServerListLoadBalancer（继承 BaseLoadBalancer），事实上从启动后一直不变的 ServerList 场景一般不太多，尤其在微服务场景：服务挂了，扩容，缩容等都会需要对服务消费方的客户端的服务列表做出实时调整（通常借助服务发现产品:eureka,consul,zk...）,DynamicServerListLoadBalancer 顾名思义就是针对该类场景的。要做到运行时实时更新，既要保证更新的实时可见，也要保证更新操作本身的同步。

ServerListUpdater，是动态服务列表的更新器。现实场景中一般有个专门提供发布和查询/订阅服务列表服务的角色，这里暂时简称为”远程注册表服务”。更新 ServerList 常见的有两种实现模式：push 或者 pull。 push 机制就是客户端保持对“远程注册表服务”观察就行，服务器发生变化时会，客户端就会及时得到通知。 还有一种 pull 机制，一般是频繁的发请求进行询问“远程注册表服务”是否有变化发生。这个一般建议基于常见的服务发现产品进行实现。

ServerListFilter，是DynamicServerListLoadBalancer的可选构造参数之一。作用是在发生 ServerList 更新时筛选过滤出符合条件的一个子集。 因为 ServerList 可能比较大，包含成百上千台机器地址，如果都尝试去调用，那么客户端的连接数就会非常多，这也会造成必要的消耗。

## Ribbon-httpclient 模块

ribbon-loadbalancer 模块中有个 ClientFactory 静态工具类，可以生成和管理多个名字唯一的 IClient 具体实例。`ClientFactory.getNamedClient("sample-client"); `，这里因为未特殊配置（指定具体的 IClient 的实现类），所以选择的是默认的Client实现类 com.netflix.niws.client.http.RestClient。

- RestClient

RestClient，是 ribbon-httpclient 模块中针对 IClient 的具体实现，通过使用 Jesery Client (实际依赖 apacheHttpClient4 发送 http),同样 HttpRequest 实现 ClientRequest,而HttpResponse 实现了 ClientResponse。

`client.executeWithLoadBalancer(request)`,是client的父类 AbstractLoadBalancerAwareClient 中定义的方法，最终是调用 `LoadBalancerContext#getServerFromLoadBalancer(URI,loadBalancerKey)`方法,该方法主要LB逻辑如下:

```
Server svc = lb.chooseServer(loadBalancerKey);
if (svc == null){
    throw new ClientException(ClientException.ErrorType.GENERAL,
            "Load balancer does not have available server for client: "
                    + clientName);
}

//... check host not null

logger.debug("{} using LB returned Server: {} for request {}", new Object[]{clientName, svc, original});
return svc;
```

---
### 著作权声明

`首次发布于此，转载请保留以上链接`
