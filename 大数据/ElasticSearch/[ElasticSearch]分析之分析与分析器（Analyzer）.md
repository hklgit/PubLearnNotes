### 1. 分析过程

分析(analysis)是这样一个过程：
- 首先，标记化一个文本块为适用于倒排索引单独的词(term)
- 然后标准化这些词为标准形式，提高它们的“可搜索性”或“查全率”

这个工作是分析器(Analyzer)完成的。


### 2. 分析器组成

分析器（Analyzer） 一般由三部分构成，字符过滤器（Character Filters）、分词器（Tokenizers）、分词过滤器（Token filters）。

![image](http://img.blog.csdn.net/20170522113039734?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 2.1 字符过滤器

首先字符串要按顺序依次经过几个字符过滤器(Character Filter)。它们的任务就是在分词（tokenization）前对字符串进行一次处理。字符过滤器能够剔除HTML标记，或者转换"&"为"and"。

#### 2.2 分词器

下一步，字符串经过分词器(tokenizer)被分词成独立的词条（ the string is tokenized into individual terms by a tokenizer）。一个简单的分词器(tokenizer)可以根据空格或逗号将文本分成词条（A simple tokenizer might split the text into terms whenever it encounters whitespace or punctuation）。

#### 2.3 分词过滤器

最后，每个词条都要按顺序依次经过几个分词过滤器(Token Filters)。可以修改词（例如，将"Quick"转为小写），删除词（例如，停用词像"a"、"and"、"the"等等），或者增加词（例如，同义词像"jump"和"leap"）。

Elasticsearch提供很多开箱即用的字符过滤器，分词器和分词过滤器。这些可以组合来创建自定义的分析器以应对不同的需求。



### 3. 内建分析器

不过，Elasticsearch还附带了一些预装的分析器，你可以直接使用它们。下面我们列出了最重要的几个分析器，来演示一下它们有啥差异。我们来看看使用下面的字符串会产生什么样的分词：
```
Set the shape to semi-transparent by calling set_trans(5)
```

#### 3.1  标准分析器（Standard analyzer）

标准分析器是Elasticsearch默认使用的分析器。对于文本分析，它对于任何语言都是最佳选择（对于任何一个国家的语言，这个分析器基本够用）。它根据Unicode Consortium（http://www.unicode.org/reports/tr29/ ）的定义的单词边界(word boundaries)来切分文本，然后去掉大部分标点符号。最后，把所有词转为小写。

```
    @Test
    public void analyzeByAnalyzer() throws Exception {
        String standardAnalyzer = "standard";
        String value = "Set the shape to semi-transparent by calling set_trans(5)";
        AnalyzeAPI.analyzeByAnalyzer(client, standardAnalyzer, value);
    }
```    
产生的结果为：
```
set, the, shape, to, semi, transparent, by, calling, set_trans, 5
```
#### 3.2 简单分析器（Simple analyzer）

简单分析器将依据不是字母的任何字符切分文本，然后把每个词转为小写（The simple analyzer splits the text on anything that isn’t a letter, and lowercases the terms）。
```
    @Test
    public void analyzeByAnalyzer() throws Exception {
        String simpleAnalyzer = "simple";
        String value = "Set the shape to semi-transparent by calling set_trans(5)";
        AnalyzeAPI.analyzeByAnalyzer(client, simpleAnalyzer, value);
    }
```    
产生的结果为：

```
set, the, shape, to, semi, transparent, by, calling, set, trans
```
#### 3.3 空格分析器（Whitespace analyzer）

空格分析器依据空格切分文本（The whitespace analyzer splits the text on whitespace）。它不转换小写。
```
    @Test
    public void analyzeByAnalyzer() throws Exception {
        String whitespaceAnalyzer = "whitespace";
        String value = "Set the shape to semi-transparent by calling set_trans(5)";
        AnalyzeAPI.analyzeByAnalyzer(client, whitespaceAnalyzer, value);
    }
```    
产生结果为：
```
Set, the, shape, to, semi-transparent, by, calling, set_trans(5)
```

#### 3.4 语言分析器（Language analyzers）

特定语言分析器适用于很多语言（https://www.elastic.co/guide/en/elasticsearch/reference/2.4/analysis-lang-analyzer.html ）。它们能够考虑到特定语言的特性（They are able to take the peculiarities of the specified language into account）。例如，english分析器自带一套英语停用词库（像and或the这些与语义无关的通用词），分析器将会这些词移除。因为语法规则的存在，英语单词的主体含义依旧能被理解（This analyzer also is able to stem English words because it understands the rules of English grammar）。

以英语分析器举例：
```
    @Test
    public void analyzeByAnalyzer() throws Exception {
        String englishAnalyzer = "english";
        String value = "Set the shape to semi-transparent by calling set_trans(5)";
        AnalyzeAPI.analyzeByAnalyzer(client, englishAnalyzer, value);
    }
```    
产生结果为：
```
set, shape, semi, transpar, call, set_tran, 5
```
注意"transparent"、"calling"和"set_trans"是如何转为词干的（stemmed to their root form）。


### 4. 当分析器被使用


当我们索引(index)一个文档，全文字段会被分析为单独的词来创建倒排索引。不过，当我们在全文字段搜索(search)时，我们要让查询字符串经过同样的分析流程处理，以确保这些词在索引中存在。理解每个字段是如何定义的，这样才可以让它们做正确的事：

- 当你查询全文(full text)字段，查询将使用相同的分析器来分析查询字符串，以产生正确的词列表。

- 当你查询一个确切值(exact value)字段，查询将不分析查询字符串，但是你可以自己指定。


### 5. 测试分析器

尤其当你是Elasticsearch新手时，对于如何分词以及存储到索引中理解起来比较困难。为了更好的理解如何进行，你可以使用analyze API来查看文本是如何被分析的。在查询中指定要使用的分析器，以及被分析的文本。

    /**
     * 使用分词器进行词条分析
     * @param client
     * @param analyzer
     * @param value
     */
    public static void analyzeByAnalyzer(Client client, String analyzer, String value){
        IndicesAdminClient indicesAdminClient = client.admin().indices();
        AnalyzeRequestBuilder analyzeRequestBuilder = indicesAdminClient.prepareAnalyze(value);
        analyzeRequestBuilder.setAnalyzer(analyzer);
        AnalyzeResponse response = analyzeRequestBuilder.get();
        // 打印响应信息
        print(response);
    }
打印信息：

    /**
     * 打印响应信息
     * @param response
     */
    private static void print(AnalyzeResponse response){
        List<AnalyzeResponse.AnalyzeToken> tokenList = response.getTokens();
        for(AnalyzeResponse.AnalyzeToken token : tokenList){
            logger.info("-------- analyzeIndex type {}", token.getType());
            logger.info("-------- analyzeIndex term {}", token.getTerm());
            logger.info("-------- analyzeIndex position {}", token.getPosition());
            logger.info("-------- analyzeIndex startOffSet {}", token.getStartOffset());
            logger.info("-------- analyzeIndex endOffSet {}", token.getEndOffset());
            logger.info("----------------------------------");
        }
    }


### 6. 指定分析器

当Elasticsearch在你的文档中探测到一个新的字符串字段，它将自动设置它为全文string字段并用standard分析器分析。

你不可能总是想要这样做。也许你想使用一个更适合这个数据的语言分析器。或者，你只想把字符串字段当作一个普通的字段——不做任何分析，只存储确切值，就像字符串类型的用户ID或者内部状态字段或者标签。为了达到这种效果，我们必须通过映射(mapping)人工设置这些字段。

```
XContentBuilder mappingBuilder;
        try {
            mappingBuilder = XContentFactory.jsonBuilder()
                    .startObject()
                    .startObject(type)
                    .startObject("properties")
                    .startObject("club").field("type", "string").field("index", "analyzed").field("analyzer", "english").endObject()
                    .endObject()
                    .endObject()
                    .endObject();
        } catch (Exception e) {
            logger.error("--------- putIndexMapping 创建 mapping 失败：", e);
            return false;
        }

```


参考：https://www.elastic.co/guide/en/elasticsearch/guide/current/analysis-intro.html#analysis-intro







