### 1. 深分页

在[[ElasticSearch]Search之分页](http://blog.csdn.net/sunnyyoona/article/details/72558047)一文中，我们初步了解到分布式系统中深度分页．在这里我们再具体的了解一下深分页，可能带来的问题，以及ElasticSearch给出的解决方案．

在 [[ElasticSearch2.x]原理之分布式搜索](http://blog.csdn.net/sunnyyoona/article/details/72781802) 一文中我们了解到分布式搜索的工作原理，分布式搜索这种先查后取的过程支持用`from`和`size`参数分页，但是这是有限制的．　请记住，每个分片必须构建一个长度为`from+size`的优先级队列，所有这些队列都需要传递回协调节点。 协调节点需要对`number_of_shards *（from + size）`个文档进行排序，以便正确找到`size`个文档。

取决于你的文档的大小，分片的数量和你使用的硬件，给 10,000 到 50,000 的结果文档深分页（ 1,000 到 5,000 页）是完全可行的。但是使用足够大的`from` 值，排序过程可能会变得非常沉重，使用大量的CPU、内存和带宽。因为这个原因，我们强烈建议你不要使用深分页。

实际上， “深分页” 很少符合人的行为。当2到3页过去以后，人会停止翻页，并且改变搜索条件。不知疲倦地一页一页的获取网页直到你的服务崩溃的罪魁祸首一般是机器人或者网络爬虫。

如果你确实需要从集群里取回大量的文档，你可以通过使用`scroll`查询==禁用排序==来更有效率的取回文档，具体我们会在下面进行讨论。

### 2. 游标

`scroll`查询用于从Elasticsearch有效地检索大量文档，而又不需付出深度分页那种代价。

`scroll`允许我们进行初步搜索，并且不断地从Elasticsearch中取回批量结果，直到取回所有结果(Scrolling allows us to do an initial search and to keep pulling batches of results from Elasticsearch until there are no more results left.)。 它有点像传统数据库中的游标。

`scroll`搜索在某个时间上生成快照。 在初始搜索请求完成后，搜索不会看到发生在索引上的任何更改。 它通过保留旧的数据文件来实现这一点，以便可以保留其在开始搜索时索引的“视图”( it can preserve its “view” on what the index looked like at the time it started.)。

深层分页的代价主要花费在全局排序结果上，但是如果我们禁用排序，那么我们可以花费较少的代价就能返回所有的文档。为此，我们按`_doc`排序。 这样Elasticsearch只是从仍然有结果需要返回的每个分片返回下一批结果。

要滚动查询结果，我们执行一个搜索请求，并将`scroll`值设置为保持滚动窗口打开的时间长度。 每次运行滚动请求时都会刷新滚动到期时间，因此只需要足够长的时间来处理当前批次的结果，而不是所有与查询匹配的文档。 超时设置是非常重要的，因为保持滚动窗口打开需要消耗资源，我们希望在不再需要时释放它们。 设置这个超时能够让 Elasticsearch 在稍后空闲的时候自动释放这部分资源。

```
GET /old_index/_search?scroll=1m 
{
    "query": { "match_all": {}},
    "sort" : ["_doc"], 
    "size":  1000
}
```
==说明==

上面语句保持游标查询窗口一分钟。并且根据`_doc`进行排序；

对此请求的响应包括`_scroll_id`，它是一个Base-64编码的长字符串。 现在我们可以将`_scroll_id`传递给`_search/scroll`接口来检索下一批结果：
```
GET /_search/scroll
{
    "scroll": "1m", 
    "scroll_id" : "cXVlcnlUaGVuRmV0Y2g7NTsxMDk5NDpkUmpiR2FjOFNhNnlCM1ZDMWpWYnRROzEwOTk1OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MTA5OTM6ZFJqYkdhYzhTYTZ5QjNWQzFqVmJ0UTsxMTE5MDpBVUtwN2lxc1FLZV8yRGVjWlI2QUVBOzEwOTk2OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MDs="
}
```
==说明==

再次设置游标查询过期时间为一分钟。

对此`scroll`请求的响应包括下一批结果。 虽然我们指定了请求大小为1000，但是我们可能会得到更多的文件。当查询的时候，size 作用于每个个分片，所以每个批次实际返回的文档数量最大为 `size * number_of_primary_shards` 。

==注意==

`scroll`请求还返回一个新的`_scroll_id`。 每次我们进行下一个`scroll`请求时，我们必须传递上一个`scroll`请求返回的`_scroll_id`。


当没有更多的命中返回时，我们已经处理了所有匹配的文档。


原文：https://www.elastic.co/guide/en/elasticsearch/guide/current/scroll.html
