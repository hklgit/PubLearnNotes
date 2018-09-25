
http://l-qesaasdatanodessd40.ops.cn2.qunar.com:15102/

### 1. 搜索

#### 1.1 空搜索
```
curl -XGET 'localhost:9200/_search?pretty'
```




### 索引数据
```
curl -XPUT  'localhost:9200/my_store/products/1' -d '{
"price" : 10, 
"productID" : "XHDK-A-1293-#fJ3"
}';
```

### 查看数据
```
curl -XGET 'localhost:9200/my_store/products/1?pretty'
```

### 搜索数据

```
curl -XPOST 'localhost:9200/my_store/products/_search?pretty' -d'
{
  "query": {
    "term" : { "price" : 20 } 
  }
}
';
```

### 非评分查询
```
curl -XGET 'localhost:9200/my_store/products/_search?pretty' -d'
{
    "query" : {
        "constant_score" : { 
            "filter" : {
                "term" : { 
                    "price" : 20
                }
            }
        }
    }
}
'
```

### 删除索引

```
curl -XDELETE 'localhost:9200/my_store?pretty';
```

### 分析
```
curl -XGET 'localhost:9200/my_store/_analyze?pretty' -d'
{
  "field": "productID",
  "text": "XHDK-A-1293-#fJ3"
}
';
```

### 清空缓存
```
curl -XPOST 'localhost:9200/qunar-index/_cache/clear'
```

### 集群统计信息
```
curl -XGET 'localhost:9200/_cluster/stats?pretty' 
```

### 查看分段

```
curl -XGET 'localhost:9200/xxx/_segments';
```

### 执行计划explain

```
curl -XGET 'localhost:9200/xxx/_search?pretty=1&explain=true'  -d '
{
    "query" : {
  	"constant_score" : {
          "filter":{
             "term":{
                "entrance":"301"
              }
           }
        }
     }
}
'
```

### 查看配置
```
curl -XGET 'localhost:9200/xxx/_settings/index*';
```

### 更改配置
```
curl -XPUT 'localhost:9200/xxx/_settings' -d'
{ "index.queries.cache.enabled": true }
';
```

###　创建索引时设置配置
```
curl -XPUT 'localhost:9200/xxx' -d'
{
  "settings": {
    "index.cache.query.enable": true
  }
}
';
```