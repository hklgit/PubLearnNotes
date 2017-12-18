结合Maneesh Varshney的漫画改编，为大家分析HDFS存储机制与运行原理。

### 1. 角色出演

如下图所示，`HDFS`存储相关角色与功能如下：
- `Client`：客户端，系统使用者，调用`HDFS API`操作文件；与`NameNode`交互获取文件元数据；与`DataNode`交互进行数据读写。
- `Namenode`：元数据节点，是系统唯一的管理者。负责元数据的管理；与`client`交互进行提供元数据查询；分配数据存储节点等。
- `Datanode`：数据存储节点，负责数据块的存储与冗余备份；执行数据块的读写操作等。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Hadoop/Hadoop%20%E5%9B%BE%E8%A7%A3HDFS%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86-1.png?raw=true)

### 2. 写数据

#### 2.1 发送写数据请求

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Hadoop/Hadoop%20%E5%9B%BE%E8%A7%A3HDFS%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86-2.png?raw=true)

`HDFS`中的存储单元是`block`。文件通常被分成64或128M一块的数据块进行存储。与普通文件系统不同的是，在`HDFS`中，如果一个文件大小小于一个数据块的大小，它是不需要占用整个数据块的存储空间的。

#### 2.2 文件切分

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Hadoop/Hadoop%20%E5%9B%BE%E8%A7%A3HDFS%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86-3.png?raw=true)


#### 2.3 DataNode分配

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Hadoop/Hadoop%20%E5%9B%BE%E8%A7%A3HDFS%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86-4.png?raw=true)

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Hadoop/Hadoop%20%E5%9B%BE%E8%A7%A3HDFS%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86-5.png?raw=true)

#### 2.4 数据写入

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Hadoop/Hadoop%20%E5%9B%BE%E8%A7%A3HDFS%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86-48?raw=true)

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Hadoop/Hadoop%20%E5%9B%BE%E8%A7%A3HDFS%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86-6.png?raw=true)



















来源于: 京东大数据专家公众号
