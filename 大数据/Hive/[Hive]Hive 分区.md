
```

set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode=nonstrict;
set hive.exec.max.dynamic.partitions.pernode=1000;
set hive.enforce.bucketing = true;
```




### 添加分区
```sql
alter table tmp_toutiao_ads_show add partition(dt='20180417') location 'hdfs://qunarcluster/user/wirelessdev/log/ads/show/20180417';
```
### 查看分区
```sql
show partitions tmp_toutiao_ads_show;
```
### 删除分区
```sql
ALTER TABLE tmp_toutiao_ads_show DROP PARTITION (dt='20180417');
```
### 添加多个分区
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS ods_deeklink_click_order (
  line string
)
PARTITIONED BY(
  dt string,
  type string
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
LOCATION '/user/wirelessdev/data_group/adv/ods_deeklink_click_order';
```
```sql
ALTER TABLE ods_deeklink_click_order add partition(dt='20180521',type='all') location 'day=20180521/type=all';
ALTER TABLE ods_deeklink_click_order add partition(dt='20180521',type='new') location 'day=20180521/type=new';
```
