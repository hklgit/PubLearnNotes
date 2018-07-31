### 1. 副本分片
到目前为止，我们只讨论了主分片，但是我们还有另一个工具：副本分片。 副本分片的主要目的是为了故障转移（failover），如[深入集群生命周期](https://www.elastic.co/guide/en/elasticsearch/guide/current/distributed-cluster.html)所述：如果持有主分片的节点死亡，则将其副本提升为主分片的角色。

在索引写入时，副本分片做着与主分片相同的工作。新文档首先被索引进主分片然后再同步到其它所有的副本分片。增加副本数并不会增加索引容量。

但是，副本分片可以为读取请求提供帮助。 如果通常情况下，你的索引搜索占很大比重（偏向于查询使用），则可以通过增加副本数量来增加搜索性能，但这样你也会为此付出占用额外的硬件资源的代价。

让我们回到那个具有两个主分片的索引示例中。 我们通过添加第二个节点来增加索引的容量。 添加更多节点不会帮助我们提升索引写入能力，但是我们可以在搜索时通过增加副本分片的数量来充分利用额外硬件资源：

```
PUT /my_index/_settings
{
  "number_of_replicas": 1
}
```

拥有两个主分片，另外加上每个主分片的一个副本，我们总共拥有四个分片：每个节点一个，如下图所示：
![image](http://img.blog.csdn.net/20170510095448053?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### 2. 通过副本进行负载均衡

搜索性能取决于最慢节点的响应时间，所以尝试均衡所有节点的负载是一个好想法。如果我们只是增加一个节点而不是两个，最终我们会有三个节点，其中两个节点只拥有一个分片，另一个节点拥有两个分片做着两倍的工作（ one node doing double the work with two shards）。

我们可以通过调整分片副本数量来平衡这些。通过分配两个副本，最终我们会拥有六个分片，刚好可以平均分给三个节点
```
PUT /my_index/_settings
{
  "number_of_replicas": 2
}
```
如下图所示：

![image](http://img.blog.csdn.net/20170510095501769?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

作为奖励，我们同时提升了我们的可用性。我们可以容忍丢失两个节点而仍然保持一份完整数据的拷贝。

备注

事实上节点 3 拥有两个副本分片，没有主分片并不重要。副本分片与主分片做着相同的工作；它们只是扮演着略微不同的角色。没有必要确保主分片均匀地分布在所有节点中。




原文：
