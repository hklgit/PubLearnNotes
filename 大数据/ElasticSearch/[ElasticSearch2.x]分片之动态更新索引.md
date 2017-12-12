
### 1. 不变性

倒排索引被写入磁盘后是 `不可改变`(immutable) :它永远不会修改。 不变性有重要的优势：

- 不需要锁。如果你没有必要更新索引，你就没有必要担心多进程会同时修改数据。
- 一旦索引被读入内核的文件系统缓存中，由于其不会改变，便会留在那里。只要文件系统缓存中还有足够的空间，那么大部分读请求会直接请求内存，而不会命中磁盘。这提供了很大的性能提升。
- 其它缓存(例如filter缓存)，在索引的生命周期内始终保持有效。因为数据不会改变，不需要在每次数据改变时被重建。
- 写入一个大的倒排索引中允许数据被压缩，减少磁盘 I/O 和 缓存索引所需的RAM量。

当然，一个不变的索引也有缺点。主要事实是它是不可变的! 你不能修改它。如果你需要让一个新的文档 可被搜索，你需要重建整个索引。这对索引可以包含的数据量或可以更新索引的频率造成很大的限制。

### 2. 动态更新索引

下一个需要解决的问题是如何更新倒排索引，而不会失去其不变性的好处？ 答案是：`使用多个索引`。

通过增加一个新的补充索引来反映最近的修改，而不是直接重写整个倒排索引。每一个倒排索引都会被轮流查询--从最旧的开始--再对各个索引的查询结果进行合并。

Lucene是Elasticsearch所基于的Java库，引入了 `按段搜索` 的概念。 每一个段 本身就是一个倒排索引， 但`Lucene`中的`index`除表示段`segments`的集合外，还增加了提交点`commit point`的概念，一个列出了所有已知段的文件，如下图所示展示了带有一个提交点和三个分段的Lucene索引:

![image](http://img.blog.csdn.net/20170523142849636?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

新文档首先被添加到内存中的索引缓冲区中，如下图所示展示了一个在内存缓存中包含新文档准备提交的Lucene索引:

![image](http://img.blog.csdn.net/20170510095548192?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

然后写入到一个基于磁盘的段，如下图所示展示了在一次提交后一个新的段添加到提交点而且缓存被清空:

![image](http://img.blog.csdn.net/20170524095459365?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 2.1 索引与分片

一个Lucene索引就是我们Elasticsearch中的分片`shard`，而Elasticsearch中的一个索引是分片的集合。 当Elasticsearch搜索索引时，它将查询发送到属于该索引的每个分片(Lucene索引)的副本(主分片，副本分片)上，然后将每个分片的结果聚合成全局结果集，如[Distributed Search Execution](https://www.elastic.co/guide/en/elasticsearch/guide/current/distributed-search.html)中描述．

#### 2.2 按段搜索工作过程

(1)新文档被收集到内存索引缓存中，如上第一图；

(2)缓存经常被提交： 
- 一个新的段(补充的倒排索引)被写入磁盘
- 一个新的提交点`commit point`被写入磁盘，其中包括新的段的名称。
- 磁盘进行`同步` — 所有在文件系统缓存中等待的写入都刷新到磁盘，以确保它们被写入物理文件。
 
(3)新分段被开启，使其包含的文档可以被搜索。

(4)内存缓冲区被清除，并准备好接受新的文档。

当一个查询被触发，所有已知的段轮流被查询。词项统计会对所有段的结果进行聚合，以保证每个词和每个文档的关联都被准确计算。 这种方式可以用相对较低的成本将新文档添加到索引。

### 3. 删除与更新

分段是不可变的，因此无法从较旧的分段中删除文档，也不能更新旧的分段以反映较新版本的文档。 相反，每个提交点`commit point`都包括一个.del文件，其中列出了哪个文档在哪个段中已经被删除了。

当文档被“删除”时，它实际上只是在.del文件中被标记为已删除。 标记为已删除的文档仍然可以匹配查询，但在最终查询结果返回之前，它将从结果列表中删除。

文档更新以类似的方式工作：当文档更新时，文档的旧版本被标记为已删除，并且文档的新版本被索引到新的段中。 也许文档的两个版本都可以匹配查询，但是在查询结果返回之前较旧的标记删除版本的文档会被移除。

在[分段合并](https://www.elastic.co/guide/en/elasticsearch/guide/current/merge-process.html)中，我们将展示如何从文件系统中清除已删除的文档。


原文：https://www.elastic.co/guide/en/elasticsearch/guide/current/dynamic-indices.html