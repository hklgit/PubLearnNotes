关于Redis与Memcached优劣的讨论一直是一个热门的话题．在性能上Redis是单线程模式，而Memcached支持多线程，所以在多核服务器上后者的性能更高一些．然而，Redis的性能已经足够优异，在绝大部分场合下其性能都不会成为瓶颈，所以在使用时更多应该关心的是二者功能上的区别，如果需要用到更高级的数据类型或者是持久化等功能，Redis将会是Memcached很好的替代品．

### 1. 什么是Redis

`Redis`是一个速度非常快的非关系数据库，它可以存储键(`key`)与5种不同类型的值(`value`)之间的映射，可以将存储在内存的键值对数据持久化到硬盘，可以使用复制特性来扩展读性能，还可以使用客户端分片来扩展写性能等等。

### 2. 对比

高性能键值缓存服务器`memcached`也经常被拿来与`Redis`进行比较：这两者都可用于存储键值映射，彼此的性能也相差无几，但是`Redis`能够自动以两种不同的方式将数据写入硬盘，并且`Redis`除了能存储普通的字符串键之外，还可以存储其他4种数据结构，而`memcached`只能存储普通的字符串键。这些不同之处使得`Redis`可以用于解决更为广泛的问题，并且既可以用作主数据库使用，又可以作为其他存储系统的辅助数据库使用。
