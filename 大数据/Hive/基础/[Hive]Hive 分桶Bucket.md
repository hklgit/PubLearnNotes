
### 1. 概述

通常 Hive 中的分区提供了一种将 Hive 表数据分隔成多个文件/目录的方法。当如下情况下，分区会产生有效结果：
- 有限个分区
- 分区大小差不多大

但这并不能在所有情况下都能实现，比如当我们根据地理位置（例如国家）对表进行分区时，一些较大的国家会有较大的分区（例如：4-5个国家就站总数据的70-80％），一些小国家则分区比较小（剩余的所有国家可能只占全部数据的20-30％）。 所以，在这些情况下分区并不理想。

为了克服分区的这个问题，Hive 引入 Bucketing 的概念，将表数据集分解为更易管理的一个个小数据集的一种技术。

特征
- Bucketing 概念基于（bucketed列上的哈希函数）mod（按桶的总数）。 hash_function 取决于 bucketing 列的类型。
- 具有相同带有分栏的记录将始终存储在同一个存储桶中。
- 我们使用CLUSTERED BY子句将表分成桶。
- 从物理上讲，每个桶只是表目录中的一个文件，并且桶编号是基于1的。
- Bucking可以与Hive表上的Partitioning一起完成，甚至不需要分区。
- Bucketed表将创建几乎平均分布的数据文件部分。

### 2. 用法

我们可以在 `CREATE TABLE` 语句中使用 `CLUSTERED BY` 子句和可选的 `SORTED BY` 子句创建 bucketed 表。在下面的 HiveQL 的帮助下，我们可以创建具有以上给定要求的 `tmp_bucketed_user` 表。
```
CREATE TABLE tmp_bucketed_user(
  firstname VARCHAR(64),
  lastname  VARCHAR(64),
  address STRING,
  city  VARCHAR(64),
  state VARCHAR(64),
  post  STRING,
  phone1  VARCHAR(64),
  phone2  STRING,
  email STRING,
  web STRING
)
COMMENT 'A bucketed sorted user table'
PARTITIONED BY (country VARCHAR(64))
CLUSTERED BY (state) SORTED BY (city) INTO 32 BUCKETS
STORED AS SEQUENCEFILE;
```










http://hadooptutorial.info/bucketing-in-hive/
https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL+BucketedTables
