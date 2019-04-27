### 1. 概述

Redis 在 2.8.9 版本添加了 HyperLogLog 结构。HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。

在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。

### 2. 什么是基数?

比如数据集 {1, 3, 5, 7, 5, 7, 8}， 那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数(不重复元素)为5。 基数估计就是在误差可接受的范围内，快速计算基数。

### 3. 命令

HyperLogLog目前只支持3个命令，PFADD、PFCOUNT、PFMERGE。我们先来逐一介绍一下。

#### 3.1 PFADD

> 最早可用版本：2.8.9。时间复杂度：O(1)。

将参数中的元素都加入指定的 HyperLogLog 数据结构中，这个命令会影响基数的计算。如果执行命令之后，基数估计改变了，就返回1；否则返回0。如果指定的key不存在，那么就创建一个空的 HyperLogLog 数据结构。该命令也支持不指定元素而只指定键值，如果不存在，则会创建一个新的 HyperLogLog 数据结构，并且返回1；否则返回0。

#### 3.2 PFCOUNT

> 最早可用版本：2.8.9。时间复杂度：O(1)，对于多个比较大的key的时间复杂度是O(N)。

对于单个key，该命令返回的是指定key的近似基数，如果变量不存在，则返回0。对于多个key，返回的是多个 HyperLogLog 并集的近似基数，它是通过将多个 HyperLogLog 合并为一个临时的 HyperLogLog，然后计算出来的。HyperLogLog 可以用很少的内存来存储集合的唯一元素。（每个HyperLogLog只有12K加上key本身的几个字节）

HyperLogLog 的结果并不精准，错误率大概在0.81%。

需要注意的是：该命令会改变 HyperLogLog，因此使用8个字节来存储上一次计算的基数。所以，从技术角度来讲，PFCOUNT是一个写命令。

性能问题

即使理论上处理一个存储密度大的HyperLogLog需要花费较长时间，但是当指定一个key时，PFCOUNT命令仍然具有很高的性能。这是因为PFCOUNT会缓存上一次结算的基数，而多数PFADD命令不会更新寄存器。所以才可以达到每秒上百次请求的效果。

当处理多个key时，最耗时的一步是合并操作。而通过计算出来的并集的基数是不能缓存的。所以多个key的处理速度一般在毫秒级。

#### 3.3 PFMERGE

> 最早可用版本：2.8.9。时间复杂度：O(N)，N是要合并的HyperLogLog的数量。

用法：PFMERGE destkey sourcekey [sourcekey …]

合并多个HyperLogLog，合并后的基数近似于合并前的基数的并集（observed Sets）。计算完之后，将结果保存到指定的key。

除了这三个命令，我们还可以像操作String类型的数据那样，对HyperLogLog数据使用SET和GET命令。关于HyperLogLog的原理以及其他细节，我将在明天的文章中进行介绍，敬请期待。



参考:[Redis命令详解：HyperLogLog](https://mp.weixin.qq.com/s/vZ2c0lKXJNR1x8Is10iWkA)

[走近源码：神奇的HyperLogLog](https://mp.weixin.qq.com/s/4uyaGeqz9nfSnZGih4JBBw)

[神奇的HyperLogLog算法](http://www.rainybowe.com/blog/2017/07/13/%E7%A5%9E%E5%A5%87%E7%9A%84HyperLogLog%E7%AE%97%E6%B3%95/index.html?utm_source=tuicool&utm_medium=referral)


...
