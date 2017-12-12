### 1. 刷新flush

如果没有使用`fsync`将文件系统缓存中的数据刷新到磁盘上，我们无法保证数据在断电后甚至在正常退出应用程序后仍然存在。 为了使Elasticsearch具有可靠性，我们需要确保将更改持久化到磁盘上。

在[动态更新索引](https://www.elastic.co/guide/en/elasticsearch/guide/current/dynamic-indices.html)中，我们说过一次完全提交会将分段刷新到磁盘，并写入一个提交点`commit point`，其中列出了所有已知的段。Elasticsearch在启动或重新打开索引时使用此提交点，来确定哪些段属于当前分片。

当我们每秒刷新一次即可实现近实时搜索，但是我们仍然需要定期进行全面的提交，以确保我们可以从故障中恢复。 但发生在两次提交之间文件变化怎么办？ 我们也不想丢失。

Elasticsearch添加了一个`translog`或事务日志，它记录了Elasticsearch中的每个操作。 使用`translog`，处理过程现在如下所示：

(1) 索引文档时，将其添加到内存索引缓冲区中，并追加到`translog`中，如下图所示:

![image](http://img.blog.csdn.net/20170525202809867?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

(2) 刷新`refresh`使分片处于下图描述的状态，分片每秒被刷新（refresh）一次：
- 内存缓冲区中的文档将写入一个新的段，而不需要fsync
- 分段被打开以使其可以搜索。
- 内存缓冲区被清除。

![image](http://img.blog.csdn.net/20170525202755633?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


(3) 该过程继续，将更多的文档添加到内存缓冲区并追加到事务日志，如下图所示:

![image](http://img.blog.csdn.net/20170525202746195?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

(4) 每隔一段时间--例如`translog`变得非常大，索引被刷新`flush`到磁盘；一个新的`translog`被创建，并且执行一个全量提交：

- 内存缓冲区中的任何文档都将写入新的段。
- 内存缓冲区被清除。
- 一个提交点被写入硬盘。
- 文件系统缓存通过`fsync`被刷新`flush`到磁盘。
- 老的`translog`被删除。

![image](http://img.blog.csdn.net/20170525202734383?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


`translog`提供了尚未刷新到磁盘的所有操作的持久记录。 启动时，Elasticsearch将使用最后一个提交点从磁盘中恢复已知的段，然后将重新执行`translog`中的所有操作，以添加最后一次提交后发生的更改。

`translog`也被用来提供实时`CRUD` 。当你试着通过ID查询、更新、删除一个文档，在尝试从相应的段中检索文档之前，首先检查`translog`来查看 最近的变更。这意味着它总是能够实时地获取到文档的最新版本。


### 2. flush API

在Elasticsearch中执行提交和截断`translog`的操作被称作一次`flush`。分片每30分钟或者当`translog`变得太大时会自动刷新一次。 有关可用于控制这些阈值的设置，请参阅[translog文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-translog.html#_translog_settings)．

flush API可用于执行手动刷新：
```
POST /blogs/_flush 

POST /_flush?wait_for_ongoing 
```
==说明==

第一个语句刷新`blogs`索引；第二个语句刷新所有索引，等待所有刷新完成后返回。

你很少需要手动刷新`flush`；通常情况下，自动刷新就足够了。

这就是说，在重启节点或关闭索引之前执行`flush`有益于你的索引。当 Elasticsearch 尝试恢复或重新打开一个索引，它需要重新执行`translog` 中所有的操作，所以如果`translog`中日志越短，恢复越快。

### 3. Translog有多安全？

`translog`的目的是确保操作不会丢失。 这就提出了一个问题：`translog`的安全性如何？

在文件被`fsync`到磁盘前，被写入的文件在重启之后就会丢失。默认情况下，`translog`每5秒进行一次`fsync`刷新到磁盘，或者在每次写请求(例如`index`, `delete`, `update`, `bulk`)完成之后执行。这个过程发生在主分片和副本分片上。最终，这意味着在整个请求被`fsync` 到主分片和副本分片上的`translog`之前，你的客户端不会得到一个`200 OK`响应。

在每个请求之后执行`fsync`都会带来一些性能消耗，尽管实际上相对较小（特别是对于bulk导入，在单个请求中平摊了许多文档的开销）。

但是对于一些高容量的群集而言，丢失几秒钟的数据并不严重，因此使用异步的`fsync`还是比较有益的。比如，写入的数据被缓存到内存中，再每5秒整体执行一次`fsync` 。

可以通过将`durability`参数设置为异步来启用此行为：
```
PUT /my_index/_settings
{
    "index.translog.durability": "async",
    "index.translog.sync_interval": "5s"
}
```

可以为每个索引配置此设置，并可动态更新。 如果你决定启用异步`translog`行为，你需要确认如果发生崩溃，丢失掉`sync_interval`时间段的数据的价值( lose sync_interval's worth of data)。 在决定使用这个参数前请注意这个特征！


如果您不确定此操作的后果，最好使用默认值（“index.translog.durability”：“request”）以避免数据丢失。





原文：https://www.elastic.co/guide/en/elasticsearch/guide/current/translog.html