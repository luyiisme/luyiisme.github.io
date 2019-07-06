---
layout:     post
title:      "Prometheus 如何从外部监控 minikube 部署的K8S"
date:       2019-07-06
author:     "luyi"
header-img: "img/post-bg-metalworking.jpg"
tags:
    - minikube
    - Prometheus
---
最近研究 Prometheus 是如何感知和监控 k8s 集群组件的。为了方便 debug Prometheus 代码，在本机上安装了 minikube。目前文章讲(prometheus)使用 in-cluster 模式监控 k8s 集群的比较多，但是从外部监控观测怎么做比较少，所以这里记录下，基于 minikube 摸索的方式。

##  1.启动 minikube 并找到 apiserver 的地址（端口）

- 启动并创建个集群

```
$ minikube start

😄  minikube v1.2.0 on darwin (amd64)
✅  using image repository registry.cn-hangzhou.aliyuncs.com/google_containers
💡  Tip: Use 'minikube start -p <name>' to create a new cluster, or 'minikube delete' to delete this one.
🔄  Restarting existing virtualbox VM for "minikube" ...
⌛  Waiting for SSH access ...
🐳  Configuring environment for Kubernetes v1.15.0 on Docker 18.09.6
🔄  Relaunching Kubernetes v1.15.0 using kubeadm ...
⌛  Verifying: apiserver proxy etcd scheduler controller dns
🏄  Done! kubectl is now configured to use "minikube"
```

- 查看 apiserver 地址

```
$ kubectl cluster-info

Kubernetes master is running at https://192.168.99.100:8443
KubeDNS is running at https://192.168.99.100:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

- 访问检查

```
$ curl --cacert ~/.minikube/ca.crt --cert ~/.minikube/client.crt --key ~/.minikube/client.key https://192.168.99.100:8443/api
```
响应成功:
```
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.99.100:8443"
    }
  ]
}%
```

## prometheus 的对应配置

比如只关注(服务发现) node 的 metrics，配置如下("~/.minikube" 目录下含有相关证书):

```
scrape_configs:
- job_name: 'kubernetes-nodes'
  kubernetes_sd_configs:
  - role: node
    api_server: 'https://192.168.99.100:8443'
    tls_config:
      ca_file: /Users/xx/.minikube/ca.crt
      cert_file: /Users/xx/.minikube/client.crt
      key_file: /Users/xx/.minikube/client.key
```
在 prometheus 的控制台上查看：

![image](/img/in-post/prom-node-discovery.jpg)
