---
layout:     post
title:      "Prometheus 如何感知并获取 K8S 相关组件 metrics"
date:       2019-07-11
author:     "luyi"
header-img: "img/post-bg-metalworking.jpg"
tags:
    - kubernetes
    - scrape
    - Prometheus
---
这篇主要从源码层面分析 Prometheus 如何感知并获取 K8S 相关组件（pod,node...） metrics。

## 1. Prometheus 的样例配置

首先提供个基本的 Prometheus 的样例配置，该配置是让 prometheus 感知宿主机(node)的集群（部署）分布，并抓取它们各自的 metrics 信息。

```
scrape_configs:
- job_name: 'kubernetes-nodes'
  kubernetes_sd_configs:
  - role: node
```


## 2. 配置解析逻辑分析
main.go 的启动部分，有完整解析 promethues 的 YML 配置的逻辑；

（1）/config/config.go: LoadFile

 loadFile 通过加载配置文件 prometheus.yml 的文本，生成下面的总体的 Config 对象。


（2）/cmd/main.go: reloadConfig

 reloaders 是一个函数数组，其中包括 :（scape包的Manager）scrapeManager.ApplyConfig 等方法


（3）/config/config.go :ScrapeConfig

  抓取k8s集群组件的配置部分（比如: [kubernetes_sd_config 部分](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)，该过程中还依赖 ServiceDiscoveryConfig ，这是用于服务自动发现的相关配置[下一部分析]。

## 3. 服务发现的分析

服务发现,是根据我们的配置文件默认采用 k8s client lib 库开发进行观察 apiserver 关于相关组件服务的变更通知。

/cmd/main.go：中服务发现部分 `discoveryManagerScrape.ApplyConfig(c)`，使用用户的配置分析:

```
package discovery

func (m *Manager) ApplyConfig(cfg map[string]sd_config.ServiceDiscoveryConfig) error {
...
   for name, scfg := range cfg {
      m.registerProviders(scfg, name)
...
       }
   for _, prov := range m.providers {
      m.startProvider(m.ctx, prov)
   }
...
}
```


* （1）registerProviders 过程中 new Discoverer （接口）的对象（包括 pods,services,nodes 等）；
* （2）startProvider  触发 discovery 的 run 函数，从而激发了（discovery 中所有 discovers 的 run 行为）；每个discover (比如 node , pod,services,endpoint 等)就获取到这些 roles 的结点( 通过k8s 的go的client库观察 apiserver资源的变化,拿到的信息)并加工为 TargetGroup，然后加到（等待消费）chan 中。

```
func (m *Manager) startProvider(ctx context.Context, p *provider) {
...
   updates := make(chan []*targetgroup.Group)

//如果是k8s.run发现，这里根据用户在YAML中配置的，比如要感知的ROLE（比如: node,pod....）
   go p.d.Run(ctx, updates)
   go m.updater(ctx, p, updates)
}
```

以 node 为例展示下，展示run的逻辑，加工生成感知到的节点 targets信息，并发送到 updates中。

package kubernetes 的 node.go 中逻辑：

```
func (n *Node) Run(ctx context.Context, ch chan<- []*targetgroup.Group) {
...
   go func() {for n.process(ctx, ch) {}}()
...
}


func (n *Node) process(ctx context.Context, ch chan<- []*targetgroup.Group) bool {
   keyObj, quit := n.queue.Get()
...
   key := keyObj.(string)
...


   o, exists, err := n.store.GetByKey(key)
...
   node, err := convertToNode(o)
...
   send(ctx, n.logger, RoleNode, ch, n.buildNode(node))
...
}
```

最后 updates 的内容会发送到抓取管理器同步的 chan 中（谁在观察这个chan？下面继续分析）。


## 4.感知到目标，如何进行抓取

scrape.NewManager(log.With(logger, "component", "scrape manager"), fanoutStorage),这是 main.go 中在构建 scrapeManager，其中 fanoutStorage 就是抓取到数据存储的地方。

scrapeManager.Run(discoveryManagerScrape.SyncCh()) ，对要抓取的结点信息（通过服务发现得到的）进行定时拉取。

- (1)启动“抓取对象”变化的reload的监听逻辑

```
package scrape

func (m *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) error {
//这里启动reloader的监听逻辑
   go m.reloader()
   for {
      select {
      case ts := <-tsets:
//先更新要抓取的目标
         m.updateTsets(ts)
         select {
//然后发送reload信号
         case m.triggerReload <- struct{}{}:
         default:
         }
...
      }
   }
}
```
m.reloader（）的异步执行，逻辑是等待 triggerReload的信号 chan 的更新，从而触发 reload 函数，如下

```
func (m *Manager) reload() {
m.mtxScrape.Lock()
...

//m.targetSets 现在获取到的是更新后的了！（因为更新的流程，是先更新它，再发 triggerReload 信号的）
   for setName, groups := range m.targetSets {
...
         sp, err := newScrapePool(scrapeConfig, m.append, m.jitterSeed, log.With(m.logger, "scrape_pool", setName))
...
         m.scrapePools[setName] = sp
...
      wg.Add(1)
      // Run the sync in parallel as these take a while and at high load can't catch up.
      go func(sp *scrapePool, groups []*targetgroup.Group) {

// scape pool 同步并处理这些变更的对象
         sp.Sync(groups)
         wg.Done()
      }(m.scrapePools[setName], groups)
   }
   m.mtxScrape.Unlock()
   wg.Wait()
}
```



- (2)Sync同步到要处理的 targetgroups，分别为它们创建个“循环抓取”任务

```
func (sp *scrapePool) Sync(tgs []*targetgroup.Group) {
...
   var all []*Target
...
   for _, tg := range tgs {


//这里操作：是从 targetGroup 中解析目标，relabel 过程也在这个过程中
      targets, err := targetsFromGroup(tg, sp.config)
...
      for _, t := range targets {
...
            all = append(all, t)
...
      }
   }
...
   sp.sync(all)
```

下面 sp(scrapePool) 的 sync 的处理逻辑：

```
func (sp *scrapePool) sync(targets []*Target) {
...
   for _, t := range targets {
...
      if _, ok := sp.activeTargets[hash]; !ok {
         s := &targetScraper{Target: t, client: sp.client, timeout: timeout}


//为某个target对象，产生个loop的抓取任务    
         l := sp.newLoop(scrapeLoopOptions{})
      }
...


//并发运行这个loop的抓取任务
      go l.run(interval, timeout, nil)
```


- (3)“循环抓取”任务主要是解析对应的 target ，并进行抓取

```

func (s *targetScraper) scrape(ctx context.Context, w io.Writer) (string, error) {
...
	req, err := http.NewRequest("GET", s.URL().String(), nil)
...
	resp, err := s.client.Do(s.req.WithContext(ctx))
...
	_, err = io.Copy(w, s.gzipr)
...
}
```

上面请求的是 node 10.0.0.15 的 10250 端口的 URL: /metrics 的路径来获取 metrics 信息，（ 备注:由于 ip 地址是 INNER 地址，如果 Prometheus 在 K8S 外部实际请求不通）。

文章，通过 prometheus 如何监控 k8s 的相关宿主机器的指标场景，讲述了大概的实现步骤。
