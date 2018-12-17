

### 1. create

说明:
```
创建表。需要传递表名，列族，以及可选的表配置。
```
语法:
```
create 't1', 'f1'
create 't1', {NAME => 'f1', VERSIONS => 5}
create 't1', {NAME => 'f1'}, {NAME => 'f2'}, {NAME => 'f3'}
create 't1', {NAME => 'f1', VERSIONS => 1, TTL => 2592000, BLOCKCACHE => true}
create 't1', {NAME => 'f1', CONFIGURATION => {'hbase.hstore.blockingStoreFiles' => '10'}}
```
>

Example:
```
hbase(main):003:0> create 'test_counter', 'cf'
0 row(s) in 1.3550 seconds

=> Hbase::Table - test_counter
```


### 5. incr

说明:
```
在指定的表/行/列坐标处递增单元格的值。
```
语法:
```
incr ‘t1’, ‘r1’, ‘c1’
incr ‘t1’, ‘r1’, ‘c1’, 1
incr ‘t1’, ‘r1’, ‘c1’, 10
```

Example:
```
hbase(main):010:0> incr 'test_counter', 'row-key-one', 'cf:count', 1
COUNTER VALUE = 1
0 row(s) in 0.0140 seconds

hbase(main):011:0> incr 'test_counter', 'row-key-one', 'cf:count', 2
COUNTER VALUE = 3
0 row(s) in 0.0100 seconds
```










...
