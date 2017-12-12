相信很多用户都用过关系型数据库，我们可以在关系型数据库里面创建表（create table），这里要讨论的表和关系型数据库中的表在概念上很类似。

### 1. 内部表

#### 1.1 创建表

我们可以用下面的语句在Hive里面创建一个表：
```
CREATE  TABLE IF NOT EXISTS tmp_order_uid_total(
  uid string,
  bucket_type string,
  file_name string
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
STORED AS TEXTFILE;
```
#### 1.2 导入数据
```
hive> load data local inpath '/home/xiaosi/adv/order_uid_total.txt' overwrite into table tmp_order_uid_total;
Copying data from file:/home/xiaosi/adv/order_uid_total.txt
Copying file: file:/home/xiaosi/adv/order_uid_total.txt
Loading data to table test.tmp_order_uid_total
Table test.tmp_order_uid_total stats: [num_partitions: 0, num_files: 1, num_rows: 0, total_size: 2172505813, raw_data_size: 0]
OK
Time taken: 27.359 seconds
```
**说明**

/home/xiaosi/adv/路径是Linux本地文件系统路径；同时/home/xiaosi/adv/在HDFS文件系统也有这样的一个路径。
从上面的输出我们可以看到数据是先从本地的/home/xiaosi/adv/文件夹下复制到HDFS上的/home/xiaosi/adv/(这个是Hive中的配置导致的)文件中。
最后Hive将从HDFS上把数据移动到test数据库下的tmp_order_uid_total表中。移到表中的数据到底存放在HDFS的什么地方？这个在Hive的${HIVE_HOME}/conf/hive-site.xml配置文件中指定，`hive.metastore.warehouse.dir`属性指向的就是Hive表数据存放的路径（在这配置的是/user/hive/xiaosi/）。Hive每创建一个表都会在`hive.metastore.warehouse.dir`指向的目录下以表名创建一个文件夹，所有属于这个表的数据都存放在这个文件夹里面（/user/hive/xiaosi/test.db/tmp_order_uid_total）。

查看数据：
```
sudo -uxiaosi hadoop fs -ls /user/hive/xiaosi/test.db/tmp_order_uid_total
Found 1 items
-rw-r--r--   3 xiaosi supergroup 2172505813 2016-10-27 12:05 /user/hive/xiaosi/test.db/tmp_order_uid_total/order_uid_total.txt
```
#### 1.3 删除表

如果需要删除tmp_order_uid_total表，可以用下面的命令:
```
hive> drop table tmp_order_uid_total;
Moved: 'hdfs://cluster/user/hive/xiaosi/test.db/tmp_order_uid_total' to trash at: hdfs://cluster/user/xiaosi/.Trash/Current
OK
Time taken: 0.321 seconds
```
从上面的输出我们可以得知，原来属于tmp_order_uid_total表的数据被移到hdfs://cluster/user/xiaosi/.Trash/Current文件夹中（如果你的Hadoop没有取用垃圾箱机制，那么drop table tmp_order_uid_total命令将会把属于该表的所有数据全部删除）。其实就是删掉了属于tmp_order_uid_total表的数据。记住这些，因为这些和外部表有很大的不同。同时，属于tmp_order_uid_total表的元数据也全部删除了。



### 2. 外部表

#### 2.1 外部普通表

```
CREATE EXTERNAL TABLE IF NOT EXISTS tmp_client_behavior (
  dt string,
  time string,
  uid string,
  userName string
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
LOCATION '/user/xiaosi/tmp/data_group/test/client_behavior';
```

#### 2.2 外部分区表
```
CREATE EXTERNAL TABLE IF NOT EXISTS tmp_client_behavior (
  time string,
  uid string,
  userName string
)
PARTITIONED BY(
  dt String
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
LOCATION '/user/xiaosi/tmp/data_group/test/client_behavior';
```
修改表分区
```
alter table tmp_client_behavior add if not exists partition (dt='20170806') location 'day=20170806';
alter table tmp_client_behavior add if not exists partition (dt='20170805') location 'day=20170805';
```

查看分区
```
show partitions tmp_client_behavior;
```
删除分区
```
ALTER TABLE tmp_client_behavior DROP IF EXISTS PARTITION(dt=20170806);
```

### 3. Example
#### 3.1 复合数据类型array创建表
原数据
```
front_page 1,2,3
contact_page 3,4,5
```
建表
```
create external table tmp_laterview(pageid String, adid_list Array<int>) ROW FORMAT DELIMITED
FIELDS TERMINATED BY ' '
COLLECTION ITEMS TERMINATED BY ','
LINES TERMINATED BY '\n'
location '/user/xiaosi/test/laterview';
```
