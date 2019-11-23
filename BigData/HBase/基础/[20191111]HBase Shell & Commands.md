
### 2. HBase Shell Usage

- 确保用 HBase Shell 对所有名称使用双引号，例如表名和列名。
- 逗号分隔命令参数。
- 在输入要运行的命令之后，键入<RETURN>。
- 在表的创建和更改中，我们使用配置字典，它们是Ruby哈希。看起来像：`{‘key1’ => ‘value1’, ‘key2’ => ‘value2’, …}`。

### 3. 连接HBase Shell

通过使用以下命令，我们可以通过 Shell 连接到正在运行的HBase：
```
./bin/hbase shell
```
键入 help 然后回车可以查看 Shell 命令以及选项的列表：
```
hbase(main):001:0> help
HBase Shell, version 2.1.6, rba26a3e1fd5bda8a84f99111d9471f62bb29ed1d, Mon Aug 26 20:40:38 CST 2019
Type 'help "COMMAND"', (e.g. 'help "get"' -- the quotes are necessary) for help on a specific command.
Commands are grouped. Type 'help "COMMAND_GROUP"', (e.g. 'help "general"') for help on a command group.

COMMAND GROUPS:
  Group name: general
  Commands: processlist, status, table_help, version, whoami
...
```
如果想退出，可以通过键入 exit 退出 Shell：
```
hbase(main):002:0> exit
```
### 4. HBase Shell 命令

#### 4.1 General命令
![](1)

##### 4.1.1 Status

使用该命令展示 HBase 集群的状态，例如服务器数量：
```
hbase(main):001:0> status
1 active master, 0 backup masters, 1 servers, 0 dead, 3.0000 average load
Took 0.3686 seconds
```
后面可以添加 `summary`，`simple`，`detailed` 或 `replication` 参数。默认不填为`summary`：
```
hbase> status
hbase> status 'simple'
hbase> status 'summary'
hbase> status 'detailed'
hbase> status 'replication'
hbase> status 'replication', 'source'
hbase> status 'replication', 'sink'
```

##### 4.1.2 version

使用该命令展示 HBase 版本：
```
hbase(main):005:0> version
2.1.6, rba26a3e1fd5bda8a84f99111d9471f62bb29ed1d, Mon Aug 26 20:40:38 CST 2019
```

##### 4.1.3 table_help

##### 4.1.4 whoami

使用该命令展示当前 HBase 用户：
```
hbase(main):010:0> whoami
smartsi (auth:SIMPLE)
    groups: staff, everyone, localaccounts, _appserverusr, admin, _appserveradm, _lpadmin, com.apple.sharepoint.group.1, _appstore, _lpoperator, _developer, _analyticsusers, com.apple.access_ftp, com.apple.access_screensharing, com.apple.access_ssh
```

#### 4.2 DDL命令
![](2)

##### 4.2.1 create

使用该命令创建表，在这里必须指定表名和列族名，以及可选的命名空间。列族名可以是一个简单的字符串，也可以是包含 NAME 属性的字典：
```
hbase(main):002:0> create 't1', 'f1', 'f2', 'f3'
Created table t1
Took 2.7450 seconds
=> Hbase::Table - t1
```
上述可以使用字典方式创建：
```
create 't1', {NAME => 'f1'}, {NAME => 'f2'}, {NAME => 'f3'}
```
上面命令均没有指定命名空间，默认为`default`，使用如下命令在 `ns1` 命名空间下创建 `t1` 表：
```
hbase(main):026:0> create 'ns1:t1', 'f1', 'f2', 'f3'
Created table ns1:t1
Took 2.4030 seconds
=> Hbase::Table - ns1:t1
```

> 命名空间是表的逻辑分组，类似于关系数据库系统中的数据库。这种抽象为多租户相关功能奠定了基础。具体使用可以查阅[HBase 命名空间 Namespace](http://smartsi.club/hbase-namespace-commands-examples)

##### 4.2.2 exists

可以使用 `exists` 命令判断表是否存在：
```
hbase> exists 't1'
hbase> exists 'ns1:t1'
```
使用如下命令查看表 `t1` 是否已经存储：
```
hbase(main):003:0> exists 't1'
Table t1 does exist
Took 0.0914 seconds
=> true
hbase(main):029:0> exists 'ns1:t1'
Table ns1:t1 does exist
Took 0.1863 seconds
=> true
```

##### 4.2.3 describe

可以使用 `describe` 命名查看表信息：
```
hbase> describe 't1'
hbase> describe 'ns1:t1'
```
或者简写为：
```
hbase> desc 't1'
hbase> desc 'ns1:t1'
```
例如，使用如下命令查看表 `t1` 具体信息：
```
hbase(main):043:0> describe 't1'
Table t1 is ENABLED
t1
COLUMN FAMILIES DESCRIPTION
{NAME => 'f1', VERSIONS => '1', EVICT_BLOCKS_ON_CLOSE => 'false', NEW_VERSION_BEHAVIOR => 'false', KEEP_DELETED_CELLS => 'FALSE', CACHE_DATA_ON_WRITE => 'false', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER', MIN_VERS
IONS => '0', REPLICATION_SCOPE => '0', BLOOMFILTER => 'ROW', CACHE_INDEX_ON_WRITE => 'false', IN_MEMORY => 'false', CACHE_BLOOMS_ON_WRITE => 'false', PREFETCH_BLOCKS_ON_OPEN => 'false', COMPRESSION => 'NONE', BLOCKCACHE
 => 'true', BLOCKSIZE => '65536'}
{NAME => 'f2', VERSIONS => '1', EVICT_BLOCKS_ON_CLOSE => 'false', NEW_VERSION_BEHAVIOR => 'false', KEEP_DELETED_CELLS => 'FALSE', CACHE_DATA_ON_WRITE => 'false', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER', MIN_VERS
IONS => '0', REPLICATION_SCOPE => '0', BLOOMFILTER => 'ROW', CACHE_INDEX_ON_WRITE => 'false', IN_MEMORY => 'false', CACHE_BLOOMS_ON_WRITE => 'false', PREFETCH_BLOCKS_ON_OPEN => 'false', COMPRESSION => 'NONE', BLOCKCACHE
 => 'true', BLOCKSIZE => '65536'}
{NAME => 'f3', VERSIONS => '1', EVICT_BLOCKS_ON_CLOSE => 'false', NEW_VERSION_BEHAVIOR => 'false', KEEP_DELETED_CELLS => 'FALSE', CACHE_DATA_ON_WRITE => 'false', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER', MIN_VERS
IONS => '0', REPLICATION_SCOPE => '0', BLOOMFILTER => 'ROW', CACHE_INDEX_ON_WRITE => 'false', IN_MEMORY => 'false', CACHE_BLOOMS_ON_WRITE => 'false', PREFETCH_BLOCKS_ON_OPEN => 'false', COMPRESSION => 'NONE', BLOCKCACHE
 => 'true', BLOCKSIZE => '65536'}
3 row(s)
Took 0.0319 seconds
```

##### 4.2.5 list

可以使用 `list` 命令查看 HBase 中自定义表，可以使用正则表达式查询：
```
hbase> list
hbase> list 'abc.*'
hbase> list 'ns:abc.*'
hbase> list 'ns:.*'
```
例如，使用如下命令查看 HBase 中所有用户自定义的表：
```
hbase(main):045:0> list
TABLE
ns1:t1
ns1:test
t1
test
4 row(s)
Took 0.0075 seconds
=> ["ns1:t1", "ns1:test", "t1", "test"]
```




```
hbase(main):009:0> drop 't1'
Took 0.4618 seconds
```


```
hbase(main):050:0> disable 'test'
Took 0.7409 seconds
hbase(main):051:0> disable 'ns1:test'
Took 0.4331 seconds
```


```
hbase(main):052:0> drop 'test'
Took 0.2300 seconds
hbase(main):053:0> drop 'ns1:test'
Took 0.2357 seconds
```


This command creates a table.
ii. List
It lists all the tables in HBase.
iii. Disable
This command disables a table.
iv. Is_disabled
Whereas, it verifies whether a table is disabled.
v. enable
This command enables a table.
vi. Is_enabled
However, it verifies whether a table is enabled or not.
Let’s discuss Books for HBase
vii. Describe
It shows the description of a table.
viii. Alter
This command alters a table.
ix. Exists
This one verifies whether a table exists or not.
x. Drop
This command drops a table from HBase.
xi. Drop_all
Whereas,  this command drops the tables matching the ‘regex’ given in the command.
xii. Java Admin API
Previously, to achieve DDL functionalities through programming, when the above commands were not there, Java provides an Admin API. Basically, HBaseAdmin and HTableDescriptor are the two important classes in this package which offers DDL functionalities, under org.apache.hadoop.hbase.client package.


























原文:[HBase Shell & Commands – Usage & Starting HBase Shell](https://data-flair.training/blogs/hbase-shell-commands/)
