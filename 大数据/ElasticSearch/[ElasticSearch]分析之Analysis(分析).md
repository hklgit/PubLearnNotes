分析(analysis)是将文本（如任何电子邮件的正文）转换为添加到倒排索引中进行搜索的`tokens`或`terms`的过程。 分析由分析器`analyzer`执行，分析器可以是内置分析器或者每个索引定制的自定义分析器。

### 1. 索引时分析(Index time analysis)

例如在索引时，内置的英文分析器将会转换下面句子：
```
"The QUICK brown foxes jumped over the lazy dog!"
```
转换为添加到倒排索引中的`terms`如下：
```
[ quick, brown, fox, jump, over, lazi, dog ]
```

#### 1.1 指定索引时分析器

映射中的每个`text`字段都可以指定其自己的分析器：
```
curl -XPUT 'localhost:9200/my_index?pretty' -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "my_type": {
      "properties": {
        "title": {
          "type":     "text",
          "analyzer": "standard"
        }
      }
    }
  }
}
'
```
在索引时，如果没有指定分析器，则会在索引设置中查找一个叫做`default`的分析器。 如果没有找到，默认使用标准分析器`standard analyzer`。

### 2. 搜索时分析(Search time analysis)

同样的分析过程可以应用于进行全文检索搜索(例如`match query`　匹配查询)时，将查询字符串的文本转换为与存储在倒排索引中相同形式的`terms`。

例如，用户可能搜索：
```
"a quick fox"
```
这将由相同的英语分析器分析为以下`terms`(上面索引时举例使用的是英语分析器)：
```
[ quick, fox ]
```

即使在查询字符串中使用的确切单词不会出现在原始存储文本（`quick` vs `QUICK`，`fox` vs `foxes`）中，因为我们已将相同的分析器应用于文本和查询字符串上，查询字符串中的`terms` 能够完全匹配到倒排索引中来自文本的`terms`，这意味着此查询将与我们的示例文档匹配。

#### 2.1 指定搜索时分析器

通常情况下，在索引时和搜索时应该使用相同的分析器，而全文查询`full text queries`(例如匹配查询 `match query`)将使用映射来查找用于每个字段的分析器(use the mapping to look up the analyzer to use for each field.)。

通过查找用于搜索特定字段的分析器来决定：：
- 在查询本身中指定的分析器。
- `search_analyzer` 映射参数。
- `analyzer` 映射参数。
- 索引设置中的`default_search`分析器。
- 索引设置中的`default`分析器。
- `standard` 标准分析器.



==ElasticSearch版本==

5.4 


原文：https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html