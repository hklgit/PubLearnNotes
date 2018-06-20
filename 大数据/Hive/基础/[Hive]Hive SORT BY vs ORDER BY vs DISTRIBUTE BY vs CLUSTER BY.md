
在这篇文章中，我们主要来了解一下 SORT BY，ORDER BY，DISTRIBUTE BY 和 CLUSTER BY 在 Hive 中的表现。

### 1. ORDER BY

在Hive中，ORDER BY保证数据的全部排序，但为此它必须传递给单个reducer，这通常是性能密集型的，因此在严格模式下，配置单元使用ORDER BY使用LIMIT是必须的，这样reducer不会 不会让你负担过重。



































https://www.iteblog.com/archives/1534.html


原文：https://saurzcode.in/2015/01/hive-sort-order-distribute-cluster/
