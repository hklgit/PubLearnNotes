---
layout: post
author: sjf0115
title: Hadoop 二次排序
date: 2017-12-04 14:15:17
tags:
  - Hadoop

categories: Hadoop
---

二次排序的含义为先按某列对数据进行排序，在该次排序的基础之上再按照另一列的值进行排序，例如：
```
a       2
a       5
a       11
b       3
b       8
b       10
b       15
c       1
c       4
c       6
```
这是原始数据集，经过二次排序后，输出的数据为：
```
```
由于 Hadoop 框架默认会进行排序，所以完成二次排序的关键在于控制 Hadoop 的排序操作。

### 2. 第一种方式

将每个 Key 对应的 Value 全部存储到内存（这个只会存储到单台机器），然后对这些 Value 进行相应的排序。但是如果 Value 的数据量非常大，导致单台内存无法存储这些数据，这将会导致程序出现 `OOM`，所以这个方法不是很通用。

```java
public static class SecondarySortMapper extends Mapper<Object, Text, Text, Text> {

    @Override
    protected void map(Object key, Text value, Context context) throws IOException, InterruptedException {

        String line = value.toString();
        String[] params = line.split("\t");
        if(params.length < 2){
            return;
        }
        String k = params[0];
        String v = params[1];
        context.write(new Text(k), new Text(v));

    }

}

public static class SecondarySortReducer extends Reducer<Text, Text, Text, Text> {

    @Override
    protected void reduce(Text key, Iterable<Text> values, Context context)
            throws IOException, InterruptedException {

        List<Integer> list = Lists.newArrayList();
        for(Text text : values){
            list.add(Integer.parseInt(text.toString()));
        }
        // 排序
        Collections.sort(list);
        // 输出
        for(Integer v : list){
            context.write(key, new Text(v.toString()));
        }

    }
}
```

### 3. 第二种方式

这种方法将 Value 中的值和原先的 Key 组成一个新的复合 Key，这样我们就可以利用 Reduce 来根据这个 Key 进行排序。过程如下：

(1) 原始的键值对是(k,v)。这里的k就是原始数据中的 key：
```
(a, 11)
(b, 10)
(c, 4)
(a, 2)
(a, 5)
(c, 6)
(b, 15)
(b, 3)
(b, 8)
(c, 1)
```

(2) 将 k 和 v 组合成新的复合key，这里就是复合之后的 key。复合之后的键值对形式为： ((k,v), v)：
```
((a, 11), 11)
((b, 10), 10)
((c, 4), 4)
((a, 2), 2)
((a, 5), 5)
((c, 6), 6)
((b, 15), 15)
((b, 3), 3)
((b, 8), 8)
((c, 1), 1)
```

- 自定义分区器，将 key (复合之后的 key)相同的键值对发送到同一个 Reduce 中：
```
((a, 11), 11)  ((a, 2), 2)  ((a, 5), 5)  
((b, 10), 10)  ((b, 15), 15)  ((b, 3), 3)  ((b, 8), 8)
((c, 4), 4)  ((c, 6), 6)  ((c, 1), 1)
```
- 自定义分组函数，将k相同的键值对当作一个分组。























原文:
