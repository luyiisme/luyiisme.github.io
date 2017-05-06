---
layout:     post
title:      "Elasticsearch 5 docker 集群部署--单虚拟机多容器实例"
date:       2017-05-06
author:     "luyi"
header-img: "img/post-bg-metalworking.jpg"
tags:
    - elasticsearch
    - docker
---
如何在单一虚拟机上部署多个 elasticsearch 5 容器实例和1个 kibana 5 的容器实例呢？

## 1.准备

- 镜像的选择

 * docker.elastic.co/elasticsearch/elasticsearch:5.3.2

ES Docker 镜像选用： (ES 的官方文档中提供的)镜像注册中心 docker.elastic.co 中（es,kibana）镜像默认集成了 x-pack。x-pack 插件可以方便的进行集群监控，且即使试用证书过期了,[监控功能是「免费证书」也支持的功能](https://www.elastic.co/subscriptions)。

注意，Docker Hub 中的 elasticsearch,kibana 镜像虽然注明也是官方提供的，但是没有默认集成 x-pack 插件(如果需要可以基于此定制镜像，并安装相关插件的)。

- 目标机器

 * 选择了一台阿里云的 centos7,4c4g的虚拟机

## 2.部署步骤

- 首先，（拉取镜像）运行第一个名叫：elas1 的容器；

```
docker run -d -p 9200:9200 -p 9300:9300 --name elas1 -h elas1\
 -e cluster.name=lookout-es -e ES_JAVA_OPTS="-Xms512m -Xmx512m" -e xpack.security.enabled=false\
  docker.elastic.co/elasticsearch/elasticsearch:5.3.2
```

由于在同一台虚拟机上，保证网络的默认 bridge 模式不变，同时映射9200，9300默认端口。同时指定容器的主机名为: elas1。

“-e xpack.security.enabled=false” 表示关闭 x-pack 的安全功能，不做访问认证控制。

"ES_JAVA_OPTS=-Xms512m -Xmx512m"，指定 elasticsearch java 进程的内存大小参数（默认是2g），考虑到会部署多个实例，这里适当调下了。

- 运行第二个es 容器 elas2；

link 到容器 elas1。同时端口分别映射到宿主机器的:9211,9311端口。

```
docker run -d -p 9211:9200 -p 9311:9300 --link elas1\
  --name elas2 -e cluster.name=lookout-es -e ES_JAVA_OPTS="-Xms512m -Xmx512m" -e xpack.security.enabled=false\
  -e discovery.zen.ping.unicast.hosts=elas1 docker.elastic.co/elasticsearch/elasticsearch:5.3.2
```
- 检查 es 集群状态

```
curl http://localhost:9200/_cat/health?v

epoch      timestamp cluster    status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1494051382 06:16:22  lookout-es green           2         2      4   2    0    0        0             0                  -                100.0%
```
看到已经部署的2个 es 实例运行正常。

- 也可以运行一个 kibana 容器，图形化查看监控集群状态；

link 到容器 elas1，暴露映射5601端口，同时指定 elas1主机为 kibana 对应交互的 es 机器；

```
docker run -d --name kibana  --link elas1 -e ELASTICSEARCH_URL=http://elas1:9200 -p 5601:5601\
 docker.elastic.co/kibana/kibana:5.3.2
```
然后再浏览器中访问 kibana的监控功能，显示总体集群状态如下：

![image](https://luyiisme.github.io/img/in-post/kibana050701.jpg)

---
### 著作权声明

`首次发布于此，转载请保留以上链接`
