
### 1. Node Query Cache
The query cache is responsible for caching the results of queries. There is one queries cache per node that is shared by all shards. The cache implements an LRU eviction policy: when a cache becomes full, the least recently used data is evicted to make way for new data.

The query cache only caches queries which are being used in a filter context.

The following setting is static and must be configured on every data node in the cluster:

- indices.queries.cache.size
Controls the memory size for the filter cache , defaults to 10%. Accepts either a percentage value, like 5%, or an exact value, like 512mb.

The following setting is an index setting that can be configured on a per-index basis:

- index.queries.cache.enabled
Controls whether to enable query caching. Accepts true (default) or false.




curl XGET http://l-qesaasdatanodessd40.ops.cn2.qunar.com:15102/_stats/query_cache?pretty&human  