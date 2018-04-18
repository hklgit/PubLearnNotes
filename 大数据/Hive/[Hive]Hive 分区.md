





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
