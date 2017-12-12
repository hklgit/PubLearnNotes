由于自动刷新过程每秒会创建一个新的段 ，这样短时间内段数量就会暴增。有太多分段会带来较大的麻烦。 每一个段都会消耗文件句柄、内存和cpu运行周期。更重要的是，每个搜索请求都必须依次检查每个分段；所以分段越多，搜索也就越慢。

Elasticsearch通过在后台进行`段合并`来解决这个问题。小的段被合并成大的段，其又会被合并成更大的段(Small segments are merged into bigger segments, which, in turn, are merged into even bigger segments)。

这个时候那些被标记为删除的旧文档会从文件系统中删除．被标记删除的文档或者更新文档的旧版本文档不会被拷贝到新的更大的段中.

段合并不需要你做什么，在索引和搜索时会自动发生。该过程的工作原理如下图所示，两个提交过的段和一个未提交的段被合并到更大的段中：

![image](http://img.blog.csdn.net/20170526172226127?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


(1) 在索引时，刷新`refresh`过程会创建新的片段并开放供搜索。

(2) 合并过程选择几个相似大小的分段，并在后台将它们合并到一个新的更大的段中。这不会中断索引和搜索。

(3) 下图阐述了合并的完成过程：

![image](http://img.blog.csdn.net/20170526172216490?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- 新的片段被刷新(`flush`)到磁盘。
- 写入一个新的提交点，其中包含新的段，并排除旧的较小段。
- 新的分段开放供搜索。
- 旧段被删除。


合并大的段需要消耗大量的I/O和CPU资源，如果任其发展会影响搜索性能。 默认情况下，Elasticsearch会调节合并进程，以使搜索仍具有足够的资源来执行。

==提示==

关于为你的实例调整合并的建议，请参阅[分段与合并](https://www.elastic.co/guide/en/elasticsearch/guide/current/indexing-performance.html#segments-and-merging)


### 2. optimize AP

`optimize API`可以看做是 强制合并 API 。它会将一个分片强制合并到`max_num_segments`参数指定大小的段数目。 这样做的目的是减少段的数量（通常减少到一个），来提升搜索性能。

==警告==

`optimize API` 不应该被用在一个动态索引，一个正在被更新的索引。后台合并进程可以很好地完成这项工作。优化会阻碍这个进程。不要干扰它！


在某些具体情况下，`optimize API`可能是有好处的。 典型的用例是日志记录，每天、每周、每月的日志被存储在一个索引中( logs are stored in an index per day, week, or month)。 较旧的索引基本上是只读的; 他们不太可能改变。

在这种情况下，将旧索引的每个分片优化为单个分段是有用的; 它会使用更少的资源，搜索会更快：
```
POST /logstash-2014-10/_optimize?max_num_segments=1 
```
==说明==

将索引中的每个分片合并到单个分段中

==警告==


请注意，使用 `optimize API` 触发段合并的操作不会受到任何的限制。这可能会消耗掉你节点上全部的I/O资源, 使其没有足够的资源来处理搜索请求，从而有可能使集群失去响应。 如果你想要对索引执行 `optimize`，你需要先使用分片分配（查看 [迁移旧索引](https://www.elastic.co/guide/en/elasticsearch/guide/current/retiring-data.html#migrate-indices)）把索引移到一个安全的节点，再执行。