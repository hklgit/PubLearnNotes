### 1. 查询与过滤
Elasticsearch 使用的查询语言（DSL） 拥有一套查询组件(queries)，这些组件可以以无限组合的方式进行搭配(mixed and matched)。这套组件可以在以下两种上下文中使用：过滤上下文（filtering context）和查询上下文（query context）。

当在过滤上下文(filtering context)中使用 时，该查询被设置成一个“不评分”或者“过滤”的查询。换句话说，这个查询只是简单的问一个问题：“这篇文档是否匹配？”。回答也是非常的简单，是或者不是。

- created 时间是否在 2013 与 2014 这个区间？
- status 字段是否包含 published 这个词项？
- lat_lon 字段表示的位置是否在指定点的 10km 范围内？

当在查询上下文中使用时，查询就变成了一个“评分”的查询。和不评分的查询类似，也要去判断这个文档是否匹配，同时它还需要判断这个文档匹配程度如何（相关度）。 

此查询的典型用法是用于查找以下文档：

- 查找与 `full text search` 这个词语最佳匹配的文档
- 包含 run 这个词,也能匹配 runs，running，jog 或者 sprint
- 包含 quick, brown 和 fox 这几个词,它们之间距离越近，文档相关性越高
- 标有lucene, search 或者 java 标签, 含有标签越多, 相关性越高

一个评分查询计算每一个文档与此查询的 相关程度，同时将这个相关程度赋值给一个相关性变量 `_score`，然后使用这个变量按照相关性对匹配到的文档进行排序。相关性的概念是非常适合全文搜索的情况，因为全文搜索几乎没有完全 “正确” 的答案。

==注意==

以前版本中，查询(queries)和过滤器(filters)是Elasticsearch中两个相互独立的组件。 从Elasticsearch 2.0版本开始，过滤器就已经删除了，同时所有查询(queries)都获得了不评分查询的能力。

然而，为了简洁与清晰，我们将使用“过滤器”术语来表示在非评分过滤上下文中使用的查询。 您可以将“过滤器”，“过滤查询”和“非评分查询”等术语视为相同的。

类似地，如果术语“查询”在没有限定符的情况下单独使用，我们指的是“评分查询”。

### 2. 性能差异

过滤查询是简单的包含/排除检查，这使得它们计算速度非常快。 当至少有一个过滤查询的结果是“稀疏”的（匹配到少数文档）时，有很多不同优化方法可以做，并且非评分查询经常被使用来缓存在内存中，以便更快的访问(ere are various optimizations that can be leveraged when at least one of your filtering query is "sparse" (few matching documents), and frequently used non-scoring queries can be cached in memory for faster access.)。


相反，评分查询不仅要找到匹配的文档，还要计算每个文档的相关度，这通常使它们比非评分查询更重，更缓慢。 此外，查询结果不可缓存。

由于倒排索引的存在，一个简单的评分查询(scoring query)在只匹配几个文档时可能会比过滤数百万文档的过滤器(filter)更好。 然而，一般来说，过滤器比评分查询性能更优异，并且表现的很稳定。

过滤的目标是减少那些需要通过评分查询（scoring queries）进行检查的文档。

### 3. 如何选择查询与过滤

通常的规则是，使用 查询（query）语句来进行 `全文搜索` 或者 有对于任何影响相关性得分的条件。除此以外的情况都使用过滤（filters)(As a general rule, use query clauses for full-text search or for any condition that should affect the relevance score, and use filters for everything else)。


原文连接：https://www.elastic.co/guide/en/elasticsearch/guide/current/_queries_and_filters.html#_queries_and_filters













