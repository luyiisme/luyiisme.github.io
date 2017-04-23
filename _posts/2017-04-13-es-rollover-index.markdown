---
layout:     post
title:      "Elasticsearch Rollover 高效管理时序数据（上）"
date:       2017-04-13
author:     "luyi"
header-img: "img/post-bg-metalworking.jpg"
tags:
    - elastic
---
大家在使用 Elasticsearch 管理时序数据（比如 日志事件）时经常习惯于将每一天的数据作为一个 Index。根据日志事件的时间戳可以产生最近一天的新索引，新Index的定义可以事先使用 [index 模板](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-templates.html).

这个是易懂和易于实现的形式，但隐藏了索引管理的一些复杂性，比如：

- 要实现高的数据获取率，您希望活跃索引的分片应该分布到尽量多的节点上。
- 为了搜索优化和低的资源消耗，您希望尽可能少的分片，但不要那么大的碎片，以防变得笨重。
- 每一天一个索引使得很容易过期旧数据，但是一天你需要几个分片呢？
- 每天都是一样的吗？还是有可能某一天产生太多分片，第二天可能又太少？

在这篇博文中，我将介绍新的Rollover Pattern和支持它的API，这是一种更简单，更有效的管理基于时间的索引的方式。

## Rollover Pattern

滚动模式的工作原理如下：

- 有一个索引别名，它指向活跃索引。
- 另一个别名指向活跃和非活跃索引，并用于搜索。
- 活跃索引可以具有与您的热点节点一样多的分片，以利用所有昂贵硬件的索引资源。
- 当活跃索引太满或太旧时，它将滚动 ：创建一个新索引，索引别名从旧索引到原始索引。
- 旧的索引被移动到一个冷节点，并且被缩小到一个分片，这也可以被强制合并和压缩。

## 入门

假设我们有一个具有10个 热节点和 1个冷节点（注：冷热是通过设置机器实例的属性区分的）集群。 理想情况下，活跃索引（接收所有写操作的索引）应该在每个热节点上都有一个分片，以便将索引负载分解成尽可能多的机器上。

我们希望每个主分片的有一个副本，以确保我们可以容忍某个节点的挂掉，而数据不丢。 这意味着我们的活跃索引应该有5个主分片，共提供10个分片（每个热节点一个）。 我们还可以使用10个主分片（共20个分片，包括副本），每个节点上有两个分片。

首先，为活跃索引创建一个索引模板 ：

```
PUT _template/active-logs
{
  "template": "active-logs-*",
  "settings": {
    "number_of_shards":   5,
    "number_of_replicas": 1,
    "routing.allocation.include.box_type": "hot",
    "routing.allocation.total_shards_per_node": 2
  },
  "aliases": {
    "active-logs":  {},
    "search-logs": {}
  }
}

```


由此模板创建的索引将分配给带有 "box_type: hot" 标记的节点，并且 total_shards_per_node 设置将有助于确保分片尽可能扩展到尽可能多的hot节点。 我把它设置为2而不是1 ，所以如果一个节点失败，我们仍然可以分配碎片。

我们将使用 active-logs 索引别名映射当前的活跃索引，并使用 search-logs 别名来搜索所有日志索引。

以下是我们的非活跃索引将使用的模板：

```
PUT _template/inactive-logs
{
  "template": "inactive-logs-*",
  "settings": {
    "number_of_shards":   1,
    "number_of_replicas": 0,
    "routing.allocation.include.box_type": "cold",
    "codec": "best_compression"
  }
}
```
归档索引应分配给cold节点，并应使用deflate压缩来节省空间。 我会解释为什么我稍后将replicas设置为0 。

现在我们可以创建第一个活跃索引：

```
PUT active-logs-1
```
名称中 `-1` 的模式会被 rollover API 识别为计数器（能够进行累加）。

## 灌入数据

当我们创建了 active-logs-1 索引时，我们还创建了 active-logs 别名。 从此刻开始，我们应该仅使用别名进行索引，我们输入的文档将被发送到当前的活跃索引：

```
POST active-logs/log/_bulk
{ "create": {}}
{ "text": "Some log message", "@timestamp": "2016-07-01T01:00:00Z" }
{ "create": {}}
{ "text": "Some log message", "@timestamp": "2016-07-02T01:00:00Z" }
{ "create": {}}
{ "text": "Some log message", "@timestamp": "2016-07-03T01:00:00Z" }
{ "create": {}}
{ "text": "Some log message", "@timestamp": "2016-07-04T01:00:00Z" }
{ "create": {}}
{ "text": "Some log message", "@timestamp": "2016-07-05T01:00:00Z" }
```

## Rolling over 索引

在某个阶段，活跃索引会变得太大或太老，您将需要用新的空索引替换它。 Rollover API 允许您指定索引可以有多大或有多老。

多大算太大了？这取决于你所拥有的硬件，你执行的搜索类型，你期待的性能，你愿意等待分片恢复的时间等等。在实践中，你可以尝试不同的分片大小，看看什么对你有用。刚开始时，选择一些任意数字，如1亿或10亿。 您可以根据搜索性能，数据保留期和可用空间来上下调整此数字。

单个分片可以包含的文档数量有限制：2,147,483,519。 如果您计划将活跃索引收缩到单个分片，则您的活跃索引中的文档必须少于21亿。 如果您有更多的文档，则可以将索引缩小到多个分片，只要目标数量的分片是原始的因数，例如 6→3 或 6→2。

按时间多久来滚动索引可能很方便，因为它允许您按小时，天，周等方式存日志，但是通常根据索引中的文档数量来进行 rollover 更为有效。 基于尺寸的翻转的一个好处是，所有的分片都具有大致相同的重量，这使得它们更容易平衡。

Rollover API 将由 cron job 定期调用，以检查 max_docs或 max_age 约束是否已被打破(即满足滚动条件)。 一旦至少有一个约束被打破，索引就会滚动。 由于我们仅在上述示例中模拟了5个文档，因此我们将指定一个 max_docs 值为 5 ，（为了完整）， max_age 为一周：

```
POST active-logs/_rollover
{
  "conditions": {
    "max_age":   "7d",
    "max_docs":  5
  }
}
```

该请求告诉Elasticsearch翻转active-logs别名指向的索引，如果该索引至少在七天前创建，或至少包含5个文档。 响应如下所示：

```
{
  "old_index": "active-logs-1",
  "new_index": "active-logs-2",
  "rolled_over": true,
  "dry_run": false,
  "conditions": {
    "[max_docs: 5]": true,
    "[max_age: 7d]": false
  }
}

```

由于 max_docs: 5 约束被满足，所以 active-logs-1 索引已经被滚动到 active-logs-2 索引。 这意味着根据 active-logs 模板创建了一个名为 active-logs-2 的新索引，并将 active-logs 别名从 active-logs-1 切换到 active-logs-2 。

顺便说一下，如果要覆盖索引模板中的任何值，例如 settings或 mappings ，您可以像使用 [create index API](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-create-index.html) 一样将它们传递到 _rollover 请求正文。

## 为什么不支持 max_size 约束？

鉴于目的是生成均匀大小的分片，为什么除 max_docs 之外我们不支持 max_size 约束？ 答案是，碎片大小是不太可靠的方式，因为正在进行的合并，可能会使分片大小产生明显的临时增长，一旦合并完成又会消失。 5个主碎片，每个在合并到一个 5GB 分片的过程中，会临时将索引大小提高25GB！相比之下，文档数量则是可预测地增长。

TODO:

下篇：如何处理非活跃索引

补充篇：Rollover 的索引名字中带日期格式

---
### 著作权声明

`首次发布于此，转载请保留以上链接`

参考翻译文章：https://www.elastic.co/blog/managing-time-based-indices-efficiently
