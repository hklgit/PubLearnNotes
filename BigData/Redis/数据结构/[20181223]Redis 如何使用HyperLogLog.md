### 1. 概述

Redis 在 2.8.9 版本添加了 HyperLogLog 数据结构。HyperLogLog 是用来做基数统计的算法，其优点是在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定的、并且是很小。

在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存就可以计算接近 2^64 个不同元素的基数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。

### 2. 什么是基数?

比如数据集 {1, 3, 5, 7, 5, 7, 8}， 那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数(不重复元素)为5。基数估计就是在误差可接受的范围内，快速计算基数。

### 3. 命令

HyperLogLog 目前只支持3个命令，`PFADD`、`PFCOUNT`、`PFMERGE`。我们先来逐一介绍一下。

#### 3.1 PFADD

> 最早可用版本：2.8.9。时间复杂度：O(1)。

`PFADD` 命令将所有元素添加到存储在第一个参数指定的 HyperLogLog 数据结构中。如果命令执行之后，基数估计发生变化就返回1，否则返回0。如果指定的key不存在，那么就创建一个空的 HyperLogLog 数据结构。也可以调用不指定元素而只指定键的命令。如果键存在，不执行任何操作，返回0。如果键不存在，则会创建一个新的 HyperLogLog 数据结构，并且返回1。

(1) 语法格式:
```
PFADD key element [element ...]
```
(2) 返回值:
```
整型，如果至少有个元素被添加返回 1， 否则返回 0。
```
(3) Example:
```
127.0.0.1:6379> PFADD hll a b c d e f g
(integer) 1
127.0.0.1:6379> pfcount hll
(integer) 7
```

#### 3.2 PFCOUNT

> 最早可用版本：2.8.9。时间复杂度：O(1)，对于多个比较大的key的时间复杂度是O(N)。

`PFCOUNT` 命令返回指定 HyperLogLog 的基数估算值。对于单个键，该命令返回的是指定键的基数估算值，如果变量不存在，则返回0。对于多个键，返回的是多个 HyperLogLog 并集的基数估算值，它是通过将多个 HyperLogLog 合并为一个临时的 HyperLogLog 计算出来的。HyperLogLog 可以用很少的内存来存储集合的唯一元素。每个 HyperLogLog 只用 12K 加上键本身的几个字节。

(1) 语法格式:
```
PFCOUNT key [key ...]
```
(2) 返回值:
```
整数，返回指定 HyperLogLog 的基数估算值，如果多个 HyperLogLog 则返回基数估算值之和。
```
(3) Example:
```
127.0.0.1:6379> PFADD hll foo bar zap
(integer) 1
127.0.0.1:6379> PFADD hll zap zap zap
(integer) 0
127.0.0.1:6379> PFADD hll foo bar
(integer) 0
127.0.0.1:6379> PFCOUNT hll
(integer) 3
127.0.0.1:6379> PFADD some-other-hll 1 2 3
(integer) 1
127.0.0.1:6379> PFCOUNT some-other-hll
(integer) 3
127.0.0.1:6379> PFCOUNT hll some-other-hll
(integer) 6
```
(4) 限制:

HyperLogLog 的结果并不精确，错误率大概在 0.81% 左右。

> 该命令会修改 HyperLogLog，因此使用8个字节来存储上一次计算的基数。所以，从技术角度来讲，PFCOUNT 是一个写命令。

(5) 性能问题

即使理论上处理一个密集型 HyperLogLog 需要花费较长时间，但是当只指定一个键时，`PFCOUNT` 命令仍然具有很高的性能。这是因为 `PFCOUNT` 会缓存上一次计算的基数，而这个基数很少会变动，因为 `PFADD` 命令大多数情况下不会更新寄存器。所以才可以达到每秒上百次请求的效果。

当使用 `PFCOUNT` 命令处理多个键时，会对 HyperLogLog 进行合并操作，这一步非常耗时，更重要的是通过计算出来的并集的基数是不能缓存的。因此当使用多个键时，`PFCOUNT` 可能需要花费一些时间，毫秒的数量级，不应过过多使用。

> 我们应该记住，该命令的单键和多键执行语义上是不同的并且具有不同的性能。

#### 3.3 PFMERGE

> 最早可用版本：2.8.9。时间复杂度：O(N)，N是要合并的HyperLogLog的数量。

`PFMERGE` 命令将多个 HyperLogLog 合并为一个 HyperLogLog。合并后的 HyperLogLog 的基数估算值是通过对所有 给定 HyperLogLog 进行并集计算得出的。计算完的结果保存到指定的键中。

语法格式:
```
PFMERGE destkey sourcekey [sourcekey ...]
```
返回值:
```
返回 OK。
```
Example:
```
127.0.0.1:6379> PFADD hll1 foo bar zap a
(integer) 1
127.0.0.1:6379> PFADD hll2 a b c foo
(integer) 1
127.0.0.1:6379> PFMERGE hll3 hll1 hll2
OK
127.0.0.1:6379> PFCOUNT hll3
(integer) 6
```

```

redis-cli -h localhost -p 6379 keys "*"
```


...
