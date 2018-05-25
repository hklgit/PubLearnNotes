
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
```sql
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

CREATE EXTERNAL TABLE IF NOT EXISTS tmp_bucket_user_record (
  first_name string,
  last_name string,
  address string,
  city string,
  state string,
  post string,
  phone1 string,
  phone2 string,
  email string,
  web string
)
COMMENT 'bucket table test'
PARTITIONED BY (country string)
CLUSTERED BY (state) SORTED BY (city) INTO 16 BUCKETS
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
STORED AS TEXTFILE;

CREATE EXTERNAL TABLE IF NOT EXISTS tmp_user_record (
  first_name string,
  last_name string,
  address string,
  country string,
  city string,
  state string,
  post string,
  phone1 string,
  phone2 string,
  email string,
  web string
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
LOCATION '/user/xiaosi/tmp/data_group/example/user_record';


SET hive.enforce.bucketing = true;
INSERT OVERWRITE TABLE tmp_bucket_user_record PARTITION (country)
SELECT first_name, last_name, address, city, state, post, phone1, phone2, email, web, country
FROM tmp_user_record;
```


```
hive> INSERT OVERWRITE TABLE tmp_bucket_user_record PARTITION (country)
    > SELECT first_name, last_name, address, city, state, post, phone1, phone2, email, web, country
    > FROM tmp_user_record;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
Query ID = xiaosi_20180525133229_918211d6-20ee-46f6-a66f-f3a0242dc441
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks determined at compile time: 16
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
Starting Job = job_1504162679223_25177338, Tracking URL = http://xxx:9981/proxy/application_1504162679223_25177338/
Kill Command = /home/q/hadoop/hadoop-2.2.0/bin/hadoop job  -kill job_1504162679223_25177338
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 16
2018-05-25 13:32:40,048 Stage-1 map = 0%,  reduce = 0%
2018-05-25 13:32:47,286 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 2.61 sec
2018-05-25 13:32:55,435 Stage-1 map = 100%,  reduce = 6%, Cumulative CPU 4.9 sec
2018-05-25 13:32:56,453 Stage-1 map = 100%,  reduce = 44%, Cumulative CPU 18.94 sec
2018-05-25 13:32:57,471 Stage-1 map = 100%,  reduce = 81%, Cumulative CPU 33.43 sec
2018-05-25 13:32:58,489 Stage-1 map = 100%,  reduce = 88%, Cumulative CPU 36.17 sec
2018-05-25 13:32:59,507 Stage-1 map = 100%,  reduce = 94%, Cumulative CPU 38.97 sec
2018-05-25 13:33:03,578 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 41.57 sec
MapReduce Total cumulative CPU time: 41 seconds 570 msec
Ended Job = job_1504162679223_25177338
Loading data to table hivedata.tmp_bucket_user_record partition (country=null)
         Time taken to load dynamic partitions: 0.337 seconds
         Time taken for adding to write entity : 0.0 seconds
MapReduce Jobs Launched:
Stage-Stage-1: Map: 1  Reduce: 16   Cumulative CPU: 41.57 sec   HDFS Read: 391225 HDFS Write: 280710 SUCCESS
Total MapReduce CPU Time Spent: 41 seconds 570 msec
OK
Time taken: 35.7 seconds
```

```
hive> show partitions tmp_bucket_user_record;
OK
country=AU
country=CA
country=country
country=UK
country=US
```

```
[jifeng.si@l-livedata2.wap.cn1 ~]$  sudo -uxiaosi hadoop fs -ls /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/
Found 16 items
-rwxr-xr-x   3 xiaosi supergroup          0 2018-05-25 13:33 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000000_0
-rwxr-xr-x   3 xiaosi supergroup          0 2018-05-25 13:33 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000001_0
-rwxr-xr-x   3 xiaosi supergroup        806 2018-05-25 13:32 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000002_0
-rwxr-xr-x   3 xiaosi supergroup      12405 2018-05-25 13:33 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000003_0
-rwxr-xr-x   3 xiaosi supergroup          0 2018-05-25 13:33 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000004_0
-rwxr-xr-x   3 xiaosi supergroup      17140 2018-05-25 13:32 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000005_0
-rwxr-xr-x   3 xiaosi supergroup        950 2018-05-25 13:32 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000006_0
-rwxr-xr-x   3 xiaosi supergroup          0 2018-05-25 13:33 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000007_0
-rwxr-xr-x   3 xiaosi supergroup          0 2018-05-25 13:33 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000008_0
-rwxr-xr-x   3 xiaosi supergroup          0 2018-05-25 13:33 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000009_0
-rwxr-xr-x   3 xiaosi supergroup      11314 2018-05-25 13:32 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000010_0
-rwxr-xr-x   3 xiaosi supergroup      15262 2018-05-25 13:32 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000011_0
-rwxr-xr-x   3 xiaosi supergroup          0 2018-05-25 13:33 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000012_0
-rwxr-xr-x   3 xiaosi supergroup       4427 2018-05-25 13:32 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000013_0
-rwxr-xr-x   3 xiaosi supergroup       6132 2018-05-25 13:32 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000014_0
-rwxr-xr-x   3 xiaosi supergroup          0 2018-05-25 13:33 /user/hive/warehouse/hivedata.db/tmp_bucket_user_record/country=AU/000015_0
```



http://hadooptutorial.info/bucketing-in-hive/
https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL+BucketedTables
