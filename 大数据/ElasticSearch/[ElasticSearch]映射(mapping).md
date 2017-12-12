### 1. 概述
为了能够把日期字段处理成日期，把数字字段处理成数字，把字符串字段处理成全文本（Full-text）或精确（Exact-value）的字符串值，Elasticsearch需要知道每个字段里面都包含什么数据类型。这些类型和字段的信息存储在映射（mapping）中。

索引中的每个文档都有一个类型（type）。 每个类型拥有自己的映射（mapping）或者模式定义（schema definition）。映射定义了类型（type）中的字段，每个字段的数据类型，以及字段被Elasticsearch处理的方式（A mapping defines the fields within a type, the datatype for each field, and how the field should be handled by Elasticsearch）。映射还用于设置关联到类型上的元数据。

例如下面的mapping：
```
"mppings":{
    "football-type": {
        "properties": {
            "country": {
                "index": "not_analyzed",
                "store": true,
                "type": "string"
            },
            "club": {
                "store": true,
                "type": "string"
            },
            "name": {
                "store": true,
                "type": "string"
            }
        }
    }
}
```
### 2. 核心字段类型

Elasticsearch支持以下简单字段类型：
类型|数据类型
---|---
String|string
Whole number| byte short integer long
Floating point|float double
Boolean|boolean
Date|date


当你索引一个包含新字段的文档，一个之前没有的字段，Elasticsearch将基于JSON的基本数据类型使用动态映射（https://www.elastic.co/guide/en/elasticsearch/guide/current/dynamic-mapping.html ）猜测字段类型，使用以下规则：

JsonType|FieldType
---|---
Boolean: true 或者 false | "boolean"
Whole number: 123 | "long"
Floating point:123.45 | "double"
String, valid date: "2017-05-04" | "date"
String: "foo bar" | "string"

备注：

这意味着，如果你索引一个带引号的数字，例如，"123"，它将被映射为"string"类型，而不是"long"类型。然而，如果字段已经被映射为"long"类型，Elasticsearch将尝试转换字符串为long，如果强制转换失败则会抛出异常。

举例，索引下面信息（之前没有mapping）：
```
Map<String, Object> map = Maps.newHashMap();
map.put("name", "C罗");
map.put("sex", true);
map.put("age", 31);
map.put("birthday", "1985-02-05");
map.put("club", "皇家马德里俱乐部");
```
产生的mapping结果：
```
{
    "properties": {
        "birthday": {
            "format": "strict_date_optional_time||epoch_millis",
            "type": "date"
        },
        "sex": {
            "type": "boolean"
        },
        "club": {
            "type": "string"
        },
        "name": {
            "type": "string"
        },
        "age": {
            "type": "long"
        }
    }
}
```
### 3. 查看映射

我们可以使用mapping API来查看Elasticsearch中的映射。
```
IndicesAdminClient indicesAdminClient = client.admin().indices();
GetMappingsRequestBuilder getMappingsRequestBuilder = indicesAdminClient.prepareGetMappings(index);
GetMappingsResponse response = getMappingsRequestBuilder.get();
// 结果
for(ObjectCursor<String> key : response.getMappings().keys()){
    ImmutableOpenMap<String, MappingMetaData> mapping = response.getMappings().get(key.value);
    for(ObjectCursor<String> key2 : mapping.keys()){
        try {
            logger.info("------------- {}", mapping.get(key2.value).sourceAsMap().toString());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
输出结果：
```
14:45:26.453 [main] INFO  com.sjf.open.api.indexAPI.IndexAPI - ------------- {properties={age={type=long}, birthday={type=date, format=strict_date_optional_time||epoch_millis}, club={type=string}, name={type=string}, sex={type=boolean}}}
```
### 4. 自定义字段映射

虽然大多数情况下基本数据类型已经能够满足，但你也会经常自定义单个字段的映射，特别是字符串类型字段。 自定义类型可以使你完成一下几点：

- 区分全文（full text）字符串字段和准确字符串字段。比如字符串"北京"，全文字符串字段默认情况下会分成"北"和“京”两个词，但大多数情况下我们需要的是一个城市名称，不需要分词，所以它应该是一个准确的字符串字段，应该设置index属性为"not_analyzed"）。
- 使用特定语言的分析器（不同国家语言断词方式不一样，不同国家的人可能会使用不同的设置）
- 优化部分匹配字段
- 指定自定义日期格式

#### 4.1 type

映射中最重要的字段参数是type。除了string类型的字段，你可能很少需要映射除type之外的其他映射（For fields other than string fields, you will seldom need to map anything other than type）：
```
{
    "number_of_clicks": {
        "type": "integer"
    }
}
```
string类型的字段，默认的，考虑到包含全文本，它们的值在索引前要经过分析器分析，并且在此字段上进行全文搜索前要把查询字符串经过分析器的处理。

#### 4.2 index

对于string字段，其中最重要的一个映射参数是index，另一个是analyzer下面讲解。

index参数控制字符串以何种方式被索引。它包含三种方式：
值|说明
---|---
analyzed|首先分析这个字符串，然后索引．换言之，以全文形式索引此字段
not_analyzed|索引这个字段，使之可以被索引，但是索引内容和指定值一样．不分析此字段
no|不索引这个字段．这个字段不能被搜索到


对于字符串类型字段，index默认值为not_analyzed。如果我们想映射字段为确切值，我们需要设置它为not_analyzed：
```
{
    "tag": {
        "type":     "string",
        "index":    "not_analyzed"
    }
}
```
备注：

其他简单类型（long、double、date等等）也接受index参数，但相应的值只能是no和not_analyzed，它们的值不能是analyzed。



#### 4.3 analyzer

对于index为analyzed的字符串字段，使用analyzer参数来指定哪一种分析器将在搜索和索引的时候使用。默认的，Elasticsearch使用standard分析器，但是你可以通过指定一个内建的分析器来更改它，例如可以指定whitespace、simple或english等分析器。
```
{
    "tweet": {
        "type":     "string",
        "analyzer": "english"
    }
}
```
### 5. 更新映射

你可以在第一次创建索引的时候为type指定映射（You can specify the mapping for a type when you first create an index）。此外，之后你可以为一个新类型添加映射（或者为已有的类型更新映射）（Alternatively, you can add the mapping for a new type (or update the mapping for an existing type) later）。

备注：

你可以向已有映射中增加字段，但是你不能修改已经存在的字段映射（Although you can add to an existing mapping, you can’t change existing field mappings）。如果一个字段的映射已经存在，这可能意味着那个字段的数据已经被索引。如果你改变了字段映射，那已经被索引的数据可能会出现错误，不能被正确的搜索到。

我们可以更新一个映射来增加一个新字段，但是不能把已有字段的index属性从analyzed改到not_analyzed。



在使用下面代码设置映射时，首先创建一个空的索引：
```
// mapping
XContentBuilder mappingBuilder;
try {
    mappingBuilder = XContentFactory.jsonBuilder()
        .startObject()
        .startObject(type)
        .startObject("properties")
            .startObject("name").field("type", "string").field("store", "yes").endObject()
            .startObject("sex").field("type", "string").field("store", "yes").endObject()
            .startObject("college").field("type", "string").field("store", "yes").endObject()
            .startObject("age").field("type", "long").field("store", "yes").endObject()
            .startObject("school").field("type", "string").field("store", "yes").field("index", "not_analyzed").endObject()
        .endObject()
        .endObject()
        .endObject();
} catch (Exception e) {
    logger.error("--------- putIndexMapping 创建 mapping 失败：", e);
    return false;
}
IndicesAdminClient indicesAdminClient = client.admin().indices();
PutMappingRequestBuilder putMappingRequestBuilder = indicesAdminClient.preparePutMapping(index);
putMappingRequestBuilder.setType(type);
putMappingRequestBuilder.setSource(mappingBuilder);
// 结果
PutMappingResponse response = putMappingRequestBuilder.get();
```
产生结果：
```
"mappings":{
    "test-type": {
        "properties": {
            "college": {
                "store": true,
                "type": "string"
            },
            "school": {
                "index": "not_analyzed",
                "store": true,
                "type": "string"
            },
            "sex": {
                "store": true,
                "type": "string"
            },
            "name": {
                "store": true,
                "type": "string"
            },
            "age": {
                "store": true,
                "type": "long"
            }
        }
    }
}
```
下面我们再添加一个分析器为english的字段到上面映射中，我们新的字段会合并到上面已经存在的映射中。
```
        // mapping
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
产生结果：
```
"mappings":{
    "test-type": {
        "properties": {
            "college": {
                "store": true,
                "type": "string"
            },
            "school": {
                "index": "not_analyzed",
                "store": true,
                "type": "string"
            },
            "sex": {
                "store": true,
                "type": "string"
            },
            "club": {
                "analyzer": "english",
                "type": "string"
            },
            "name": {
                "store": true,
                "type": "string"
            },
            "age": {
                "store": true,
                "type": "long"
            }
        }
    }
}
```

参考：https://www.elastic.co/guide/en/elasticsearch/guide/current/mapping-intro.html#core-fields











