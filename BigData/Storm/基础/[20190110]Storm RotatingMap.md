RotatingMap 主要用于storm内部消息超时的处理，在Storm被多处使用，包括：Ack的超时处理以及Spout的pending消息的处理，可以理解为对HashMap的一种重构。

RotatingMap的前身是TimeCacheMap，删除了其中的清理线程以及锁机制。
